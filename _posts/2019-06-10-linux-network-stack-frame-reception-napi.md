---
layout: post
title: NAPI vs 非NAPI
subtitle: Linux网络协议栈 — Frame Reception
categories: [linux, network]
keywords: linux frame-reception
---
# NAPI模式

以驱动ixgb为例，硬件会使用中断事件通知CPU，有数据帧可用了，驱动中断处理函数ixgb_intr先关闭硬件中断，再调用函数__netif_rx_schedule，将当前接收数据帧的设备加入到以softnet_data{}->poll_list为表头的net_device{}结构链表中，并调度执行软中断NET_RX_SOFTIRQ。

软中断处理函数net_rx_action，依次遍历到接收到数据帧的设备时，执行net_device{}->poll所指的函数，实际会执行驱动程序中设备特有的函数ixgb_clean。

轮询函数ixgb_clean，先会执行函数ixgb_clean_rx_irq，循环处理每个接收到的数据帧，与非NAPI模式不同的是，它不再借助函数netif_rx将skb添加到输入队列softnet_data{}->input_pkt_queue中，而是直接调用函数netif_receive_skb。如果设备中的所有skb都处理完毕了，最后再调用函数netif_rx_complete，将设备从以softnet_data{}->poll_list为表头的net_device{}结构链表中摘除，打开硬件中断，退出轮询模式。


# 非NAPI模式

以驱动ixgb为例，硬件会使用中断事件通知CPU，该数据帧已经可用了，驱动中断处理函数ixgb_intr最后会调用函数ixgb_clean_rx_irq。
函数ixgb_clean_rx_irq，循环处理每个接收到的数据帧，对每个数据帧最终都执行函数netif_rx，将此skb添加到输入队列softnet_data{}->input_pkt_queue中，每当输入队列softnet_data{}->input_pkt_queue变空时，才会调用函数netif_rx_schedule，把输入设备(softnet_data{}->backlog_dev)排入轮询列表(softnet_data{}->poll_list)中，调度软中断NET_RX_SOFTIRQ去执行。

软中断处理函数net_rx_action，依次遍历到接收到数据帧的设备时，执行net_device{}->poll所指的函数，非NAPI模式的设备都会执行函数process_backlog，函数从输入队列softnet_data{}->input_pkt_queue中依次出队skb，最终调用函数netif_receive_skb完成处理。
