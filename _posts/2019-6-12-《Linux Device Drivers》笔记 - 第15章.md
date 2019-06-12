---
layout: post
title: 《Linux Device Drivers》笔记 - 第15章 Memory Mapping and DMA
author: 宋强
tags: linux
date: 2017-10-30 21:29 +0800
---

# Linux的内存管理

## 地址类型

Linux内部有多种地址存在，内核代码并没有说明什么情况下在什么地址下执行，我们的程序需要很注意才行，下面是Linux中存在的地址类型：

* 用户虚拟地址：用户空间所能看见的常规地址，32或者64位取决于机器架构。
* 物理地址：这个地址在处理器和系统内存之间使用，32位或者64位长，有些情况下32位机也能使用64位地址。
* 总线地址：在外围总线和内存之间使用，通常和物理地址相同。有一些体系结构提供了IOMMU，能够实现外围总线地址到内存地址的重新映射，从而让CPU访问外围设备更容易。
* 内核逻辑地址：内核逻辑地址是内核状态下常规的地址空间，经常被视为物理地址，主要是很多情况下内核逻辑地址和物理地址仅仅是存在一个偏移，逻辑地址通常保存在unsigned long或者void *中，kmalloc返回的就是这种地址。
* 内核虚拟地址：内核逻辑地址和虚拟地址的相同之处在于他们都是将内核空间的地址映射到物理地址，不同之处在于逻辑地址必须是线性连续的，而虚拟地址不一定，所以所有的逻辑地址都是虚拟地址，vmalloc返回的就是虚拟地址。

下面的两个方法可以实现逻辑地址和物理地址的相互转换，但是仅仅对低端内存有效：

```c++
#include <asm/page.h>
__pa() //逻辑到物理
__va() //物理到逻辑
```

不同的内核函数需要不同的地址，但是内核并没有类型上的区分，所以编程的时候需要注意。

## 物理地址和页

物理地址被分割成叫做页的单元，每个页的大小随着体系结构不同而不同，但是大多数使用4096个字节，定义在<asm/page.h>中的PAGE_SIZE宏给出了这个体系下的页的大小。

所有的内存地址，无论是虚拟的还是物理的，都被分为页号和页内偏移量两部分，通常页内偏移量满足页大小，比如4096下偏移量为12位。

通常将内存地址向右移动去除页内偏移量之后的剩余的页号也叫做页帧，应改转移多少位由PAGE_SHIFT决定。

## 高端与低端内存

x86体系结构中，32位架构，内核必须将4GB的虚拟地址空间分割为用户空间和内核空间，典型的分配是将3GB分配给用户空间，1GB分配给内核空间，这样在上下文过程中可以使用同样的映射。内核空间中占用空间最多的是实际物理内存的虚拟映射，内核无法操作没有被映射到内核空间的物理内存，但是内核空间部分还需要存储自己的代码，所以内核能够操作的物理内存通常是1GB减去自己的代码空间。

低端内存和高端内存给出了定义：

* 低端内存：存在于内核空间上的逻辑地址内存。
* 高端内存：不存在逻辑内存得物理内存，处在内核内存得虚拟内存上。（应该就是在实际内存条上需要swap出来得部分吧）

## 内存映射和页结构

由于内核使用逻辑地址访问内存，剩下得物理内存没有办法访问，但是需要存储他们得相关信息，这个使用得是定义在<linux/mm.h>中的page结构，这个结构的主要成员如下：

* atomic_t count：页的访问计数。
* void *virtual：页面被映射得时候存储被映射的虚拟地址，不映射的时候为空。
* unsigned long flags：描述页状态得标志，其中PG_locked表示被锁定，PG_reserved表示禁止内存管理系统访问。

驱动程序通常只需要关注直接操纵page就可以了，至于page是如何在系统中存储的则不需要关心，下面是一些在page和虚拟地址之间的操作：

| 头文件                                    | 函数（宏）                                               | 说明                                                                                                                                     |
|-------------------------------------------|----------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------|
| \<asm/page.h\>                            | struct page *virtual_to_page(void *kaddr);               | 将逻辑地址转换为page指针                                                                                                                 |
|                                           | struct page *pfn_to_page(int pfn);                       | 使用给定的页帧号返回page指针。                                                                                                           |
| \<linux/mm.h\>                            | void *page_address(struct page *page);                   | 如果页被映射的话，返回内核虚拟地址。                                                                                                     |
| \<linux/highmem.h\>                       | void *kmap(struct page *page);                           | 返回低端内存的逻辑地址，为高端内存分配映射，但是映射总数有限，同时多个任务对同一个页进行map也是可以的。                                  |
|                                           | void kunmap(struct page *page);                          | 解除映射。                                                                                                                               |
|  \<linux/highmem.h\> \<asm/kmap_types.h\> | void *kmap_atomic(struct page *page, enum km_type type); | 是kmap函数的高性能版本，需要给出type，通常为KM_USR\[0-1\]、KM_IRQ\[0-1\]，分别代表用户空间使用的程序和中断程序，拥有这个的代码必须是原子的。 |
|                                           | void kunmap_atomic(void *addr, enum km_type, type);      | 解除映射。                                                                                                                               |

## 页表

页表就是一个存储着虚拟页到物理页之间的映射信息的表。

# 虚拟内存区

虚拟内存区是用来管理进程中具有相同的某些属性的内存段的，一个进程的内存映射至少包含三段区域：

* 程序代码（text）
* 多个数据区，包括初始化数据区、未初始化数据取（BSS）还堆栈。
* 活动的内存映射映射的区域。

查看/proc/<进程ID>/maps就可以看到目标进程的内存映射，/proc/self/maps可以查看当前进程的。

每一行的显示标准都是：

```bash
start-end perm offset major:minor inode image
```

* start, end：虚拟内存的起始地址和结束地址。
* perm：对于这段内存的访问权限。
* offset：内存区域在映射文件中的偏移。
* major, minor：拥有映射文件的设备的主次设备号，一般是包含特殊文件的磁盘的。
* inode：文件索引节点号。
* image：被映射的文件名称。

结构体vm_area_struct用于存储这样的段，段中的成员和上面的相对应，仅仅是没有image。

vm_area_structi结构
用户空间调用mmap时，系统实际上就是创建了一个新的VMA，支持mmap的驱动程序需要帮助VMA进行初始化，但是VMA在操作系统中是内核管理的，驱动程序不可以随意增加和删除。

这里对VMA的vm_area_struct 结构的成员做一些说明，这个结构体定义在\<linux/mm.h\>中：

| 成员                                 | 说明                                                                                                                      |
|--------------------------------------|---------------------------------------------------------------------------------------------------------------------------|
| unsigned long vm_start;              | VMA所覆盖的虚拟内存开始地址                                                                                               |
| unsigned long vm_end;                | VMA所覆盖的虚拟内存结束地址                                                                                               |
| struct file *vm_file;                | 指向和这个区域相关联的file结构体的指针，可选                                                                              |
| unsigned long vm_pgoff;              | 以页为单位，文件中该区域的偏移量                                                                                          |
| unsigned long vm_flags;              | 描述这个区域的一些标志，驱动程序感兴趣的是VM_IO和VM_RESERVED，分别代表是IO映射区域还有内存管理系统不要将这个VMA交换出去。 |
| struct vm_operations_struct *vm_ops; | 内核能调用的对这个内存区域的一些操作。                                                                                    |
| void *vm_private_data;               | 驱动程序用于保存自身信息。                                                                                                |

vm_operations_struct

| 操作                                                                                                                                    | 说明                                                                                                                           |
|-----------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------|
| void (*open)(struct vm_aread_struct *vma);                                                                                              | 当VMA产生一次新的引用的时候会调用这个函数完成初始化，其中一种情况就是mmap的情况下内核不会调用这个函数而是使用mmap中的初始化。  |
| void (*close)(struct vm_area_struct *vma);                                                                                              | 销毁这个区域的时候所调用的函数。                                                                                               |
| struct page *(*nopage)(struct vm_area_struct *vma, unsigned long address, int *type);                                                   | 当一个进程要访问合法的VMA页，但是这个页又不在内存的时候将会负责将目标VMA页调度进来，如果这个函数为空的话则将会分配一个新的页。 |
| int (*populate)(struct vm_area_struct *vm, unsigned long address, unsigned long len, pgprot_t prot, unsigned long pgoff, int nonblock); | 用户空间访问页之前将这些页装载进内存，一般情况下驱动程序不必实现populate方法。                                                 |

## 内存映射处理

内存映射结构整合了虚拟内存链表、页表和大量的内存管理信息，每个进程都有一个定义在\<linux/shed.h\>中的struct mm_struct结构体来表示这个结构，tcb中能够找到他的指针，Linux中的线程就是通过多个进程共享这个结构来实现的。

## mmap设备操作

mmap操作的核心就是将设备的IO操作映射为对内存的操作，但是有一些限制：

* 不是所有设备都可以进行映射，比如串口这样面向流的设备就不可以进行映射。
* 映射区域的起始地址和大小必须为PAGE_SIZE的整数倍。

X Window系统利用内存映射来读写显存数据，大多数PCI设备通过内存映射来操作内部寄存器。

有关于mmap系统调用可以见UNIX -- Advanced IO一节中最后有关于内存映射的描述。

mmap系统调用在执行的时候很多工作都由内核完成了，驱动程序要实现的只是一小部分，为了执行mmap，驱动程序只需要给相应的地址建立合适的页表，并拥有效的vm_operation_struct替换内部的就好。

建立页表的方式有两种：

* 使用remap_pfn_range函数一次全部建立。
* 使用nopage的方法每次建立一页。

先介绍了简单的第一种：

### remap_pfn_range

有两个函数负责为物理地址建立新的页表：

```c++
#include <linux/>
int remap_pfn_range(struct vm_area_struct *vma, unsigned long virt_addr, unsigned long pfn, unsigend long size, pgprot_t prot);
int io_remap_pfn_range(struct vm_area_struct *vma, unsigned long virt_addr, unsigned long phys_addr, unsigned long size, pgprot_t prot);
```

* vma：这个范围内的页映射到这个区域中。
* virt_addr：重映射的时候的用户虚拟地址，为virt_addr和virt_addr+size之间的内存建立页表。
* pfn：要映射的物理内存的页帧号，通常从vma->vm_pgoff中取。
* size：映射区域大小，字节单位。
* prot：新版VMA的保护属性，vma->vm_page_prot相同。

第一个函数映射的是系统RAM内存使用，第二个是映射IO内存使用， 除了SPARC外其他系统中这两个函数等价。（x86中也是等价的吗）

## 为VMA添加操作

对VMA的open和close的调用操作都是由内核处理的，VMA这里的两个接口类似于hook，只是让驱动程序做自己想做的事，很多情况下只是打印日志。

当我们将open替换的时候还没有执行我们想要执行的open，所以在更换完VMA中的操作之后手动调用一下open。p421.

## 使用NOPAGE映射内存

使用mremap系统调用会对内存进行重新映射，内存变小的话内核自己就能处理，变多的时候就会调用驱动程序的nopage，nopage负责映射新的内存进去。

nopage被调用的时候，传给他的地址已经是错误内存所在的页的开始位置，nopage需要定位并返回指向用户所需要的页的page指针，这个函数还要使用get_page宏来增加引用计数。

nopage方法还可以在type中保存错误的类型，但其实就是有错变成VM_FAULT_MINOR。

注意，这种方法不能用于PCI内存，主要是由于PCI内存都在系统高端，用户无法获取页帧导致的，这种情况下必须使用第一种方法。

## 重映射特定的IO区域

映射内存的时候也需要判断是否超出了实际能访问的IO内存。

如果不希望系统进行自己扩展的话可以自己实现nopage之后内部直接返回-SIGBUS。

## 重新映射RAM

remap_pfn_range还有另一个限制，他只能访问保留页和超出物理内存的物理地址，也意味着他能映射高端PCI和ISA内存。

## 执行直接IO访问

有些情况下重新映射IO代表着内存上和性能上的浪费，而不入直接使用IO进行访问，这样的设备例如SCSI磁带机。

实现直接IO的关键是一个函数：

```c++
#include <linux/mm.h>
int get_user_pages(struct task_struct *tsk,
struct mm_struct *mm,
unsigned long start,
int len,
int write,
int force,
struct page **pages,
struct vm_area_struct **vmas);
```

* tsk：指向执行IO的任务的指针。
* mm：描述被映射地址空间的内存管理结构的指针。
* start：用户空间缓冲区地址开始。
* len：用户空间缓冲区长度。
* write：如果write非零，对映射的页有写权限。
* force：这个标志告诉这个函数不要考虑对制定内存页的保护，驱动程序总是设置他为0.
* pages：输出参数，如果成功的话会返回一个描述用户空间缓冲区的page的链表的指针。
* vmas：输出参数，成功的话返回vma的指针。

如果页需要回存，那么一定要设置页为dirty的，否则内核将认为页没有被改变过而直接释放：

```c++
#include <linux/page-flags.h>
void SetPageDirty(struct page *page);
```

最后释放页缓存：

```c++
#include <>
void page_cache_release(struct page *page);
```

这一节的实际操作应该是映射出来操作然后映射回去，类似读-复制-写。

# 异歩IO

一般都不需要使用异步IO，只有少量的字符驱动需要使用异步IO来优化性能。

有的时候内核使用同步IOCB，也就是实际上是同步的异步操作，驱动程序使用下面的函数来检查是不是必须使用同步IOCB：

```c++
#include <linux/aio.h>
int is_sync_kiocb(struct kiocb *iocb);
```

将来某一个时刻异步IO完成的时候，驱动程序需要通知内核操作已经完成，这个是通过下面的函数实现的：

```c++
#include <linux/aio.h>
int aio_complete(struct kiocb *iocb, long res, long res2);
```

# 直接内存访问

## DMA传输数据概览

在Linux系统中有两种方式可以引发DMA传输数据，一种是软件上的例如read，另一种则是传统硬件上的触发信号，他们需要的步骤如下：

软件上的：

1. 进程调用read，驱动程序分配一个DMA缓冲区，并将数据传送到这个缓冲区，进程开始休眠。
2. 硬件将数据写到DMA缓冲区中，写完之后产生一个中断。
3. 中断处理程序获得输入的数据，应答中断，唤醒进程，进程唤醒之后就已经有想要的数据了。

硬件上的（和传统的DMA有一半的相似，相比传统的这个相当于每次都声明一下DMA传输目的缓冲区）：

1. 硬件产生一个中断，表示数据到来。
2. 中断处理程序分配一个缓冲区，告诉硬件向哪里传输数据。
3. 设备通过DMA将数据写入到缓冲区，完成后再产生一个中断。
4. 中断处理程序响应中断，之后唤醒需要数据的进程，之后执行清理工作。

## 分配DMA缓冲区

使用DMA的时候需要注意的是，他需要的缓冲区在大于一页的时候页必须是连续的，PCI或者ISA总线在使用DMA读写数据的时候使用物理地址，必须连续。

对于驱动程序模块，要在运行时动态的分配DMA缓冲区。

### DIY分配

如果内存中充满了碎片而导致无法从内核申请到足够的连续空间，那么推荐在系统启动的时候设置启动参数使得高端的一部分内存得以保留，内核无法分配，这个时候DMA再将这部分内存作为缓冲区进行使用。

另一种更好的替代方法就是尽量使用分散/聚集IO，这样的话可以缓解对大块内存的需求。

## 总线地址

Linux中的DMA提供的是用户空间的内存和设备内存之间的数据交换，用户空间使用逻辑地址，而设备通常使用总线地址，Linux内核通过两个简单的函数提供这两种地址之间的转换，但是并不推荐使用，因为他们只适用于没有IOMMU的情况的机器：

```c++
#include <asm/io.h>
unsigned long virt_to_bus(volatile void *address);
void *bus_to_virt(unsigned long address);
```

## 通用DMA层

替代上一节讲述的两个函数的最好的方法是使用通用DMA层，各种架构对于缓存的处理都是不一样的，Linux提供了一个架构无关的DMA层来避免这种问题。

### 处理复杂的硬件

默认情况下大多数硬件都支持32位的DMA，但是有些东西不支持，要在当前体系架构下设置DMA的宽度，使用这个函数：

```c++
#include <linux/dma-mapping.h>
int dma_set_mask(struct device *device, u64 mask);  //返回0代表失败，非零成功
```

mask这个掩码表示的是寻址能力，24位寻址能力使用0x0FFFFFF，通过u64也可以看出来Linux希望支持最高64位DMA寻址能力。

这个函数也用来检查可不可以，因为有些情况下这样的DMA寻址宽度是不被支持的。

### DMA映射

DMA映射的本质是要分配的DMA缓冲区和设备要访问DMA缓冲区的地址的 集合（这个地址通常是总线地址）。

DMA新建一个dma_addr_t类型的结构来代表总线地址。

根据DMA缓冲区的长短，PCI区分两种DMA映射：

* 一致性DMA映射：就是预留一部分空间CPU和外设都能够访问的用来存储数据，和传统DMA类似。
* 流式DMA映射：流失映射可以优化性能，并且节省内存空间的使用，使用DMA的时候优先考虑这种映射方式。

### 建立一致性DMA映射

建立：

```c++
#include <>
void *dma_alloc_coherent(struct device *dev, size_t size, dma_addr_t *dma_handle, int flag);
```

size用来表示要申请分配的缓冲区的大小，这个函数总共返回两处地址，第一个是返回值，返回的是驱动程序使用的内核虚拟地址，第二个是dma_addr_t返回的设备使用的总线地址，flag还是kmalloc使用的flag。

释放缓冲区：

```c++
#include <>
void dma_free_coherent(struct device *dev, size_t size, void *vaddr, dma_addr_t dma_handle);
```

### DMA池

使用上一节的方法建立的一致性DMA的最小分配单位是页，但是如果我们需要更小单位的DMA分配的话就需要使用DMA池，池的内存空间需要显式分配。

创建：

```c++
#include <llinux/dmapoll.h>
struct dma_poll *dma_poll_create(const char *name, struct device *dev, size_t size, size_t align, size_t allocation);
```

align是硬件字节对其原则，allocation代表内存边界的最大大小（和size的效果不是冲突嘛？？）

释放DMA池：

```c++
#include <linux/dmapoll.h>
void dma_pool_destroy(struct dma_pool *pool);
```

不过销毁前，需要返回所有使用的DMA池缓存：

```c++
#include <linux/dmapool.h>
void *dma_pool_alloc(struct dma_pool *pool, int mem_flags, dma_addr_t handle);
```

返回不需要的缓冲区的话可以使用：

```c++
#include <linux/dmapool.h>
void dma_pool_free(struct dma_pool *pool, void *vaddr, dma_addr_t addr);
```

### 建立流式DMA映射

建立流式DMA映射的时候，需要告诉内核数据流动的方向：

* DMA_TO_DEVICE：内存到设备。
* DMA_FROM_DEVICE：设备到内存。
* DMA_BIDIRECTIONAL：双向数据。
* DMA_NONE：调试用，平常不能用。

创建和删除映射缓冲区：

```c++
#include <linux/>
dma_addr_t dma_map_single(struct device *dev, void *buffer, size_t size, enum dma_data_direction direction);
void dma_unmap_single(struct device *dev, void *buffer, dma_addr_t dma_addr, enum dma_data_direction direction);
```

有几点需要注意的：

* 缓冲区被映射之后属于设备而不属于CPU，在缓冲区解除映射之前CPU都不能访问缓冲区的数据（这个主要是外设需要将缓冲区的数据写入到内存，而不等待的话有可能内存没有得到更新）。
* DMA的活动期间不能解除映射。

也可以不撤销映射就访问缓冲区，这个函数的作用是将DMA缓冲区交给CPU，这样应用程序就可以直接访问这段缓冲区了：

```c++
#include <>
void dma_sync_single_for_cpu(struct device *dev, dma_handle_t bus_addr, size_t enum dma_data_direction);
```

在DMA活动之前需要将缓冲区交还：

```c++
#include <>
void dma_sync_single_for_device(struct device *dev, dma_handle_t bus_addr, size_t enum dma_data_direction);
```

### 单页流式映射

建立和释放页内存：

```c++
#include <>
dma_addr_t dma_map_page(struct device *dev, struct page *page, unsigned long offset, size_t size, enum dma_data_direction direction);
void dma_unmap_page(struct device *dev, dma_addr_t dma_address, size_t size, enum dma_data_direction direction);
```

### 分散/聚集映射

类似于reaadv和writev触发的一次多个缓冲区的读写操作，内部使用scatterlist表示每一块缓冲区，这个结构包含下面几个属性：

* struct page *page：page结构指针。
* unsigned int length：长度。
* unsigned int offset：页内偏移量。

要进行映射，就需要对其中的每一段内存进行映射，不过是使用一个函数来完成的：

```c++
#include <linux/scatterlist.h>
int dma_map_sg(struct device *dev, struct scatterlist *sg, int nents, enum dma_data_direction direction);
```

nents代表数量。

这个里面sg是返回值，驱动程序传输sg里面的每一段缓冲区，从sg中提取需要的信息linux提供了宏：

```c++
#include <linux/scatterlist.h>
dma_addr_t sg_dma_address(struct scatterlist *sg);          //返回总线地址
unsigned int sg_dma_len(struct scatterlist *sg);            //返回长度
```

传输结束后要解除映射：

```c++
#include <linux/scatterlist.h>
void unmap_dma_sg(struct device *dev, struct scatterlist *list, int nents, enum dma_data_direction);
```

流式DMA也有两个用于在不解除映射的时候访问缓冲区的方法：

```c++
#include <linux/scatterlist.h>
void dma_sync_sg_for_cpu(struct device *dev, struct scatterlist *sg, int nents, enum dma_data_direction);
void dma_sync_sg_for_device(struct device *dev, struct scatterlist *sg, int nents, enum dma_data_direction);
```

### PCI双重地址周期映射（DAC）

Linux通用DMA层并不支持DAC，如果某些驱动程序像鸦片使用DAC的话只能自己使用特定的函数，掩码的话需要使用：

```c++
#include <linux/pci.h>
int pci_dac_set_dma_mask(struct pci_dev *pdev, u64 mask);
```

DAC映射：

```c++
#include <linux/pci.h>
dma64_addr_t pci_dac_page_to_dma(struct pci_Dev *dev, struct page *page, unsigned long offset, int direction);
```

每次创建一个页的映射，direction和dma_data_direction等价。

不解除映射访问的函数：

```c++
#include <linux/pci.h>
void pci_dac_dma_sync_single_for_cpu(struct pci_dev *pdev, dma64_addr_t dma_addr, size_t len, int direction);
void pci_dac_dma_sync_single_for_device(struct pci_dev *pdev, dma64_addr_t dma_addr, size_t len, int direction);
```

# ISA设备的DMA

ISA设备的DMA分为两种，一种是native DMA，另一种则是bus-masterDMA，前一种是主机控制，第二种是传统设备控制的DMA。

有三处涉及这里的DMA传输数据：

* DMA控制器：DMA控制器存储了DMA传输的信息，包括源地址、目的地址和数据位宽度等，还有跟踪状态使用的计数器。
* 外围设备：硬件设备。
* 设备驱动程序：设备驱动程序的工作不多，只是负责将一些设置信息传送给DMA控制器。

## 注册DMA

获得和释放对DMA通道的所有权：

```c++
#include <asm/dma.h>
int request_dma(unsigned int channel, const char *name);
void free_dma(unsigned int channel);
```

申请之后name是一个显示设备名字的字符串，将出现在/proc/dma中。

这个函数返回-EINVAL表示所请求的通道数超出了范围，返回-EBUSY表示通道正在被另一个设备所占用。

建议的方式是请求DMA通道之后请求一个中断线，然后在中断之前释放通道，这样的顺序可以有效的防止死锁。

## 和DMA控制器通信

驱动程序的主要任务就是在DMA传输开始的时候配置好DMA控制器的信息。

DMA通道是一个共享资源，可能会出现多个处理器同时操作的情况，Linux规定了一个自旋锁，但是驱动程序不能够直接操作，需要通过下面两个函数：

```c++
#include <linux/dma.h>
unsigned long claim_dma_lock()；              //获取DMA自旋锁并返回当前的中断状态。
void release_dma_lock(unsigend long flags);   //释放自旋锁并且返回之前的中断状态。
```

使用下面的函数的时候应该拥有自旋锁，但是在DMA IO期间不应该拥有自旋锁：

```c++
#include <asm/dma.h>
void set_dma_mode(unsigned int channel, char mode);     //总共有三种情况，DMA_MODE_READ、DMA_MODE_WRITE还有DMA_MODE_CASCADE，级联模式用于将第一控制器连接到下一个控制器的顶端，这里没有讨论。
void set_dma_addr(unsigned int channel , unsigned int addr);  //为DMA缓冲区分配地址，存储的是addr的低24位，总线地址。
void set_dma_count(unsigned int channel, unsigned int count);   //设置传输字节数。
void disable_dma(unsigned int channel);    //在配置控制器前通常要禁用要配置的通道。
void enable_dma(unsigned int channel);    //配置完之后使能。
int get_dma_residue(unsigned int channel);   //这个函数返回DMA传输剩余的字节数，使用这个函数的时候也需要获取自旋锁。
void clear_dma_ff(unsigned int channel);    //ff代表触发器，这里DMA用于控制对16位寄存器的访问，两次8位操作之间要进行ff操作，配置DMA开始之前需要使用这个函数清除触发器。
```

为DMA传输做准备的示例：

```c++
int dad_dma_prepare(int channel, int mode, unsigned int buf, unsigned int count)
{
        unsigend long flags;
        flags = claim_dma_lock();
        disable_dma(channel);
        clear_dma_ff(channel);
        set_dma_mode(channel, mode);
        set_dma_addr(channel, virt_to_bus(buf));
        set_dma_count(channel, count);
        enable_dma(channel);
        release_dma_lock(flags);
 
        return 0;
}
```