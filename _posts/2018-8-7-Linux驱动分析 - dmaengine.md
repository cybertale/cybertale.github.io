---
layout: post
title: Linux驱动分析 - dmaengine
author: 宋强
tags: Linux dma
date: 2018-08-07 17:00 +0800
---

# dmaengine注册

dmaengine的底层驱动回调函数在dma_device结构体中注册，实现时先alloc一段内存给这个结构体，并初始化以下几个属性：
* channels：需要使用INIT_LIST_HEAD宏初始化一个链表。
* src_addr_widths：包含支持的源数据宽度的掩码。
* dst_addr_width：包含支持的目的数据宽度的掩码。
* directions：包含支持的传输的掩码，包括slave传输和mem2mem。
* residue_granularity：可以为dma_residue_ganularity的三种枚举类型中的一种，分别代表不支持余量、能够报告哪些块被传送了或者哪些Burst被传送了（Burst和chunk概念差别？？）。
* dev：包含指向device结构体的指针。

# dmaengine支持的传输模式设置

要增加支持的模式需要使用

```c++
void dma_cap_set(enum dma_transcation_type tx_type, dma_cap_mask_t *dstp);
```

第一个参数是要增加的模式，第二个是dma_device的cap_mask属性，模式通常还会影响源数据指针和目的数据指针的增减模式。

支持的模式有：
* DMA_MEMCPY：内存到内存拷贝。
* DMA_XOR：设备支持加速与异或有关的算法，比如RAID5.
* DMA_XOR_VAL：设备支持使用异或算法来进行奇偶校验。
* DMA_PQ：设备支持RAID6的P+Q计算，P是指一个异或计算，Q是指一个Reed-Solomon算法计算。
* DMA_PQ_VAL：设备支持使用RAID6 P+Q算法执行奇偶校验。
* DMA_INTERRUPT：设备支持执行dummy DMA传输操作以周期性的产生中断。
* DMA_SG：设备支持scatter-gather形式的内存到内存传输，这个不包含一块连续内存到一块连续内存的，连续内存的属第一种，比较特别。
* DMA_PRIVATE：设备仅支持从机传输，不能够进行异步传输。（应该是说必须同步指定开始，不过这一条和下一条ELC上rapid说没人能解释到底是做什么的）
* DMA_ASYNC_TX：我们不用关心，FRAMEWORK有时候会设置。
* DMA_SLAVE：设备支持内存到设备的DMA传输，无论是连续内存传输还是scatter-gather形式传输都属于scatter-gather，连续内存在内存到设备传输中采用仅有一个传输块的方式设置。
* DMA_CYCLIC：支持循环缓冲，传输块会形成环形链表进行DMA传输操作。
* DMA_INTERLEAVE：用于将数据从非连续缓冲区传送到非连续缓冲区，通常用于二维数据传输，比如framebuffer显示数据传输。

# 需要实现的底层回调方法

## device_alloc_chan_resources, device_free_chan_resources

会在驱动第一次调用dma_request_channel和最后一次调用dma_release_channel的时候调用，负责申请和释放内存等资源，可以进入睡眠。

## device_prep_dma_*

* 和功能相对应的准备函数，实现哪些取决于我们提供的功能。
* 这些函数需要创建一个或者多个硬件描述符，对应一次或多次DMA传输。
* 可以在中断上下文中调用。
* 申请内存的时候需要使用GFP_NOWAIT，但是驱动应该在probe和准备期间尽量将所有需要申请的内存都申请了，这样就不会给这块分配内存带来太多压力。
* 返回一个dma_async_tx_descriptor结构的描述符，用于标识此次传输，可以使用dma_async_tx_descriptor_init初始化，之后设置两个属性，flags和tx_submit，第一个是用户给的，第二个是一个我们需要实现的回调函数，可以将传输描述符插入到dma传输等待队列，等待issue_pending。这个结构体中还可以给callback_result赋值，传输完成之后将会调用这个函数并且返回传输结果，返回的dmaengine_result结构体有两个属性，result代表结果，residue对于支持余数的dma传输会返回余数。

## device_issue_pending

在传输描述符队列中取出第一个要传输的然后传输之后将指针指向下一个要传输的传输描述符，可以在中断过程中调用。

## device_tx_status

返回指定传输描述符代表的传输剩余传输的字节数，如果是循环传输，那么只返回当前周期传输剩余的字节数。
* tx_state参数可能为NULL。
* 可以在中断过程中调用。

## device_config

使用新的配置配置指定的传输。
* 只能设置没有在传输队列的传输符。
* dma_slave_config的direction属性不再使用，函数名就可以判断传输方向。
* 对于设备到内存传输是必须有的，但是内存到内存传输不可以有。

## device_pause

立刻暂停正在进行的传输。

## device_resume

立刻恢复传输。

## device_terminal_all

放弃所有等待队列和正在进行得DMA传送，回调函数不会执行，可以在原子过程中调用并且不能进入睡眠。

## device_synchronize

终止需要终止的传输过程，并且保证之前使用过的传送符占用的内存已被释放，还有之前的传送的回调函数全部都执行完了。
可能进入睡眠。

# 一些概念

## dma_run_dependencies

在Async TX传送的结尾调用，slave传输不需要，用于在传送前执行必要的前提动作。

## dma_cookie_t

递增的DMA传送ID，用处不明。

## DMA_CTRL_ACK

如果没设置这个值，驱动程序不可以在client响应前复用传输描述符，客户端使用async_tx_ack()响应，不过即使设置了也不一定可以重用。

## DMA_CTRL_REUSE

如果设置了这个值，传输描述符可以被复用，也意味着使用过后不要释放传输符占用的内存。
使用dmaengine_desc_set_reuse()来设置这个值，但只有通道允许重用才会成功，使用dmaengine_desc_clear_reuse()来清除这个值。
方便如果要跳过某个传输后面的可以直接使用描述符。
只有这个值被设置之后才需要手动释放传输符内存，使用dmaengine_desc_free()来释放。

# 参考

1. [ELC 2015 - An Overview of the Kernel DMAEngine Subsystem - Maxime Ripard, Free Electrons](https://www.youtube.com/watch?v=6DCgMYhCulc)
2. [Documentation/dmaengine/provider.txt](https://www.kernel.org/doc/Documentation/dmaengine/provider.txt)