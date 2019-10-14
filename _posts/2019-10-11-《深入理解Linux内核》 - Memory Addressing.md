---
layout: post
title: 《深入理解Linux内核》 - Memory Addressing
author: 宋强
tags: linux
date: 2019-10-11 9:08 +0800
---

# The Linux GDT

Linux系统中每一个CPU对应一个GDT，全部存储在`cpu_gdt_table`这个数组中，所有GDT的地址和大小存储在`cpu_gdt_descr`这个数组中。

## Linux中的GDT(Global Descriptor Table)

| Linux's GDT         	| Segment Selectors  	|
|---------------------	|--------------------	|
| null                	|                    	|
| reserved            	|                    	|
| reserved            	|                    	|
| reserved            	|                    	|
| not used            	|                    	|
| not used            	|                    	|
| TLS #1              	| 0x33               	|
| TLS #2              	| 0x3b               	|
| TLS #3              	| 0x43               	|
| reserved            	|                    	|
| reserved            	|                    	|
| reserved            	|                    	|
| kernel code         	| 0x60 (__KERNEL_CS) 	|
| kernel data         	| 0x68 (__KERNEL_DS) 	|
| user code           	| 0x73 (__USER_CS)   	|
| user data           	| 0x7b (__USER_DS)   	|
| TSS                 	| 0x80               	|
| LDT                 	| 0x88               	|
| PNPBIOS 32-bit code 	| 0x90               	|
| PNPBIOS 16-bit code 	| 0x98               	|
| PNPBIOS 16-bit data 	| 0xa0               	|
| PNPBIOS 16-bit data 	| 0xa8               	|
| PNPBIOS 16-bit data 	| 0xb0               	|
| APMBIOS 32-bit code 	| 0xb8               	|
| APNBIOS 16-bit code 	| 0xc0               	|
| APMBIOS data        	| 0xc8               	|
| not used            	|                    	|
| not used            	|                    	|
| not used            	|                    	|
| not used            	|                    	|
| not used            	|                    	|
| double fault TSS    	| 0xf8               	|
Linux中GDT内容的分布。

- not used条目是为了和32字节的cache大小一致。

**TSS(Task State Segment)**

每个CPU各有一个，被放在init_tss这个数组中初始化。
第三章会仔细讲。

**TLS(Thread Local Storage)**

三个用于多线程系统为每个线程提供本地段，set_thread_area()和get_thread_area()两个系统调用来创建和释放TLS段。

**APM(Advanced Power Management)**

三个APM段由Linux在使用高级电源管理驱动的时候私有的代码和数据使用。

**PNP(Plug and Play)**

五个PNP段由Linux PNP驱动的似有代码和数据使用。

**Double Fault TSS**

内核使用的特殊TSS段，用于处理Double Fault异常，详见第四章。

虽然每一个内核都具有一个GDT，但是他们中的大部分的值都是一致的，除了几个，例如TSS是任务独立的，还有LDT和TLS，其他的属性理论上也有可能被改变，但是大多数情况下是一致的。

## Linux中的LDT(Local Descriptor Table)

大多数Linux线程不使用LDT，内核创建了一个默认的LDT供大多数线程共享，存储在`defat_ldt`数组中。

LDT中包含五个条目，但是只有两个是常用的，一个是iBCS的执行入口(Call Gate)，还有一个是Solaris/x86的执行入口，这种入口用于执行预定义代码的时候改变CPU权限。

如果需要创建自定义的LDT，需要使用modify_ldt()，Wine就使用了这个来执行Windows代码。

# 硬件分页系统

## Page Frame和Page

通常Page用来指代一个页大小的数据，而Page Frame用于指代存储页数据的空间。例如：RAM被分成大小相同的Page Frame，每一个里面存放了一个Page的数据。

页表存放在主存空间中，这个在CSAPP的虚拟内存一章中也有讲。

## 常规分页(Regular Paging)

[图2-7]

Intel的CPU中的分页系统，将32位地址分为三部分：

- Directory：MSB开始10位。
- Table：中间的10位。
- Offset：剩下的12位。

也就是两级页表。

Page Directory的地址被存放在cr3寄存器中。

### Page Directory和Page Table中条目的结构

这两种表中条目的结构是相同的，他们包含的属性如下：

| Fields                            	|                                                                                                                               	|
|-----------------------------------	|-------------------------------------------------------------------------------------------------------------------------------	|
| Present flag                      	| 表示页是否在主存中。                                                                                                          	|
| page frame物理地址中MSB开始的20位 	| 因为这两个表4KB对齐，所以只需要保存目标地址的高20位。这个条目分别代表着一个目标Page Table的地址或者一个目标Page Frame的地址。 	|
| Accessed flag                     	| 每当对应的Page Frame被访问的时候就会被置位，需要软件复位。通常用于操作系统将不被访问的页置换出去的时候用。                    	|
| Dirty flag                        	| 只有Page Table条目使用，表示指向的Page Frame中的数据被改变，需软件复位，操作系统保持数据一致用。                              	|
| Read/Write flag                   	| 包含对Page Table和Page Frame的读写权限。                                                                                      	|
| User/Supervisor flag              	| 包含访问需要的权限等级。                                                                                                      	|
| PCD和PWT flag                     	| 控制硬件Cache对Page和Page Table行为。                                                                                         	|
| Page size flag                    	| 只有Page Directory有，如果置位，表示指向的Page Frame是一个扩展版Page Frame,大小在2MB-4MB之间。                                	|
| Global flag                       	| 只有Page Table有，用于在奔腾处理器中防止常用页被替换出TLB缓存。                                                               	|

## 扩展分页(Extended Paging)

[图2-8]

奔腾处理器模型开始允许扩展版分页系统，允许4MB大小的页，对应要将Page Directory中的size位置位。主要用于需要大量连续物理地址的时候，这样可以节省内存和TLB条目。

注意扩展分页和常规分页是共存的，主要看每个Directory中Page Directory中size的设置。

需要在CPU中置位PSE使能扩展分页。

## 页的硬件访问权限保护

分段系统中使用四个特权等级中的两个来表示用户空间和内核空间，而分页系统中主要使用Page Directory和Page Table中的User/Supervisor位表示权限。当这个flag复位的时候，只有内核模式才能访问，当被置位时，用户模式也可以访问。

## 示例：虚拟地址0x20021406的翻译过程

首先高Page Directory的地址已知，虚拟地址中的高10位表示条目偏置，也就是0x080，128,Page Directory中的第129个条目，之后Page Directory中的第129个条目存储着目标Page Table的地址，找到这个Page Table之后中间10位表示Page Table中的偏置，也就是0x21，33,第34个条目，他的条目里又存储着Page Frame的地址，之后使用后12位作为页内偏移量找到需要的字节数据。

Intel的奔腾处理器为了突破32位机屋里内存8GB的限制引入了带有36个内存访问地址引脚的CPU结构，而且引入了新的分页机制，包括奔腾Pro使用的Physical Address Extension(PAE)和奔腾III使用的Page Size Extension(PSE-36).

这两种分页机制支持将32位的虚拟地址转换成36位的物理地址，但是Linux不使用，这里也不讨论。

## 64位机器的分页架构

64位系统的分页机制相对32位系统最大的不同是分页层级的增加， 这个主要是由于如果分页层级不足的话页会过大，非常理想。

但是提供多层级的分页的方式对于不同的架构来说都是不同的，例如几种64位的架构的分页方式：

| 架构   	| 页大小 	| 使用64位中的 多少位作地址位 	| 分页层级数 	| 层级分配位数 	|
|--------	|--------	|-----------------------------	|------------	|--------------	|
| alpha  	| 8KB    	| 43                          	| 3          	| 10+10+10+13  	|
| ia64   	| 4KB    	| 39                          	| 3          	| 9+9+9+12     	|
| ppc64  	| 4KB    	| 41                          	| 3          	| 10+10+9+12   	|
| sh64   	| 4KB    	| 41                          	| 3          	| 10+10+9+12   	|
| x86_64 	| 4KB    	| 48                          	| 4          	| 9+9+9+9+12   	|

# Hardware Cache

Cache具体包括Cache Controller和一部分真正存储Cache内容的内存单元，通常是SRAM。

Cache line：Cache被分为许多lines，每个lines都包含一小段连续的存储单元，通过burst方式和DRAM交换数据。

Cache entriey：用来描述line状态和储存其信息的条目，Cache Controller维护了一个Cache entries的表，每个entry包含了一个tag和一些表达当前line状态的flag信息

Cache entry tag：包含一部分当前line映射的物理地址信息。

物理地址在被映射的时候被分成三个部分，分页系统对应的三个部分，10+10+12,在分页系统访问Cache的时候，只会比较tag信息，如果tag信息相符就会出发Cache hit，否则就是Cache miss。

### 有Cache情况下写数据的两种方式：write-through和write-back

- write-through：写数据的时候同时写cache和主存。
- write-back：只写cache,在执行cache flush的时候才写主存。Cache flush通常是cache miss触发的。

## Cache Snooping（缓存窥探）

多CPU情况下，每个CPU都具有一个本地Cache，这样的话如果使用的是write-back（大多数情况下都是），在一个CPU修改了Cache中的数据的时候需要通知其他所有Cache看他们有没有同样tag的条目需要更新，这个过程叫做Cache Snooping，这个过程也全部都是硬件完成的，不需要软件参与。

奔腾系统的Cache允许分页系统控制每一个Cache Frame的缓存行为，通过Page Table和Page Directory的PCD(Page Cache Disable)和PWT(Page Write-Through)位控制。Linux将其所有页的这两个位清零使用。

## TLB 可以看CSAPP第九章有关虚拟内存的讲解

由于页表存储在主存中，访问页表时间受限主存访问速度，所以增加的TLB，快速虚拟地址翻译物理地址。

每个CPU具有一个本地TLB，不需要保持一致，主要是由于一个CPU上运行的线程通常访问的地址空间一致，不同CPU运行不同的线程那么他们的地址空间不一致，不需要更新别人的TLB。

# Paging in Linux

为了兼容32位和64位系统的分页机制（主要是分页系统层级的问题），Linux2.6.10之前使用的是三级分页系统，后俩2.6.11之后采用的是四级分页系统，当前的四级包括：

- Page Global Directory
- Page Upper Directory
- Page Middle Directory
- Page Table

但是每一部分有多少位和使用多少层根据架构不同而不同，所以是不确定的。

在应用到32位系统的时候，两层分页就足够了，所以Linux在使用这种分层的时候Page Upper Directory和Page Middle Directory不会使用,但是位了保证同样的32位系统代码可以兼容的在64位系统上运行，Linux系统在32位模式下这两层只包含一个entry并全部映射当前Page Global Directory需要映射的地址，相当于直接跳过它们。

32位PAE支持的系统多使用一层，Global对应Page Directory Table，Middle对应Page Directory，Table对应Page Table。

### Linux利用虚拟内存实现进程的方便之处

- 为不同的进程分配不同的物理内存空间并且增加保护。
- Page和Page Frame之间的解耦。

***此章节后面的部分为Linux对于分页系统的具体实现，后期有兴趣再补充。***
