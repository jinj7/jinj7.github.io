---
layout: post
title: dpdk内存管理
categories: [dpdk]
keywords: dpdk, memory
---

### 1、hugepage

1. 从/proc/mounts中读取hugepage的挂载目录，创建hugepage file，执行文件的mmap映射，记录文件的虚拟地址和物理地址；
2. 先按照物理地址递增的顺序排序，尝试匹配出一些尽可能大的、物理地址连续的内存区域，再次进行文件的mmap映射，至此将获取到一些物理地址连续、并且虚拟地址也连续的内存区域；
3. 利用先前对hugepage处理的结果，新建memseg结构。

> 从属同一个memseg的hugepage，从属于同一个numa node，hugepage大小一致，物理地址连续，并且虚拟地址也连续。


### 2、malloc_heap

- 每个numa node都会对应一个malloc_heap结构；
- 每个memseg，根据自己从属的numa node，挂载到对应的malloc_heap结构上，即多个memseg可能会对应同一个malloc_heap结构；


### 3、malloc_elem
- malloc_elem结构内存分配在单个memseg之中，初始状态下的内存分布为：
      
    ![malloc_elem](/images/blog/malloc_elem-Page-1.png)
      
- 初始状态下，所有malloc_elem结构都会挂载到malloc_heap结构的空闲链表中；
    > malloc_heap结构具有多条空闲链表，malloc_elem结构按照所指示的内存大小，挂载到不同的空闲链表上，和kernel中的buddy system原理类似。


### 4、函数malloc_elem_alloc源码实现

函数rte_malloc和rte_zmalloc_socket的底层实现都是调用函数rte_malloc_socket，而函数rte_malloc_socket申请内存的核心就是函数malloc_elem_alloc。它的源码实现如下：

```c
/*
 * reserve a block of data in an existing malloc_elem. If the malloc_elem
 * is much larger than the data block requested, we split the element in two.
 * This function is only called from malloc_heap_alloc so parameter checking
 * is not done here, as it's done there previously.
 */
struct malloc_elem *
malloc_elem_alloc(struct malloc_elem *elem, size_t size, unsigned align,
        size_t bound)
{
    struct malloc_elem *new_elem = elem_start_pt(elem, size, align, bound);
    const size_t old_elem_size = (uintptr_t)new_elem - (uintptr_t)elem;
    const size_t trailer_size = elem->size - old_elem_size - size -
        MALLOC_ELEM_OVERHEAD;

    elem_free_list_remove(elem);

    if (trailer_size > MALLOC_ELEM_OVERHEAD + MIN_DATA_SIZE) {
        /* split it, too much free space after elem */
        struct malloc_elem *new_free_elem =
                RTE_PTR_ADD(new_elem, size + MALLOC_ELEM_OVERHEAD);

        split_elem(elem, new_free_elem);
        malloc_elem_free_list_insert(new_free_elem);
    }

    if (old_elem_size < MALLOC_ELEM_OVERHEAD + MIN_DATA_SIZE) {
        /* don't split it, pad the element instead */
        elem->state = ELEM_BUSY;
        elem->pad = old_elem_size;

        /* put a dummy header in padding, to point to real element header */
        if (elem->pad > 0) { /* pad will be at least 64-bytes, as everything
                             * is cache-line aligned */
            new_elem->pad = elem->pad;
            new_elem->state = ELEM_PAD;
            new_elem->size = elem->size - elem->pad;
            set_header(new_elem);
        }

        return new_elem;
    }

    /* we are going to split the element in two. The original element
     * remains free, and the new element is the one allocated.
     * Re-insert original element, in case its new size makes it
     * belong on a different list.
     */
    split_elem(elem, new_elem);
    new_elem->state = ELEM_BUSY;
    malloc_elem_free_list_insert(elem);

    return new_elem;
}
```

参照malloc_elem结构的初始化状态，结合分析上段代码，可以画出下图所示的几种场景：
  
![malloc_elem](/images/blog/malloc_elem-Page-2.png)
  
- 场景(1)即为初始状态，绿色的free malloc_elem标识整段空闲可用内存，灰色的dummy malloc_elem标识这段内存的末尾边界。
- 场景(2)为一般情况，新申请内存从空闲可用内存的尾部开始，向前预留特定长度的空间，包括会为新建的malloc_elem预留空间，并将其设置为红色的used malloc_elem。
- 场景(3)为一种特殊情况，从尾部向前预留内存，需要考虑对齐cache和bound；在某些情况下，致使预留内存空间的末尾与dummy malloc_elem有一定的距离；如果这个距离足够大，能够继续被用来进行内存分配，这时就需要创建一个free malloc_elem，来代表这段空闲可用内存。
- 场景(4)为另一种特殊情况，从尾部向前预留内存，如果扣除预留内存后，原来的空闲可用内存足够小，不足以继续被用来进行内存分配，这时会产生一个红色的used malloc_elem和一个蓝色的padding malloc_elem。

### 5、free

- 与malloc的相反，要释放的malloc_elem，与前向或者后向可能空闲的malloc_elem合并成一个更大的空闲内存区域。
- 同一memseg内，查找前向malloc_elem结构只需要借助前向成员指针prev，查找后向malloc_elem结构只需要进行地址累加。

