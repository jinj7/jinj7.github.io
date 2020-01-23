---
layout: post
title: ovs fastpath匹配流程
categories: [ovs]
keywords: ovs, fastpath
---


1. 接收报文
2. 从报文中提取struct sw_flow_key，包含后续匹配需要的所有信息
3. 使用报文四元组计算skb_hash值（32bits）
4. 使用skb_hash值（32bits）的每个字节的数值（8bits），作为索引值，依次获取mask cache中的对应entry。将报文计算出的skb_hash值，与mask cache entry中的对应数值进行比较，如果匹配执行步骤5，否则作为备选
5. 根据mask cache entry中记录的mask_index，获取mask array中所对应的struct sw_flow_mask
6. 使用步骤2获取的struct sw_flow_key，与步骤5获取的struct sw_flow_mask，进行掩码运算后，计算出新的hash值，进而获取table instance中的hash bucket
7. 逐个遍历步骤6获取的hash冲突链上的所有struct sw_flow
8. 使用步骤2获取的struct sw_flow_key，与步骤7获取的struct sw_flow，根据步骤5获取的truct sw_flow_mask，进行带掩码比较。完全相同，表示查找报文对应的流表项已成功，否则，继续查找
9. 如果未查到报文匹配的流表项，fastpath查找结束，进入slow path

![ovs-fastpath](/images/blog/ovs-fastpath.png)

> 步骤4中，会找到最优的（skb_hash完全相同，如果有）和备选的mask cache entry，来获取mask array中所对应的struct sw_flow_mask；如果使用上述的struct sw_flow_mask查找流表项都失败了，那么只能依次遍历mask array中所有的struct sw_flow_mask，来查找流表项

