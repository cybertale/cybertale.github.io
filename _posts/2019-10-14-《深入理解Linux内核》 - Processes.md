---
layout: post
title: 《深入理解Linux内核》 - Processes
author: 宋强
tags: linux
date: 2019-10-14 11:22 +0800
---

# Processes, Lightweight Processes, and Threads

对于一个Process的最常见定义：一个程序执行的实例。

本质上一个Process是一个当前程序运行状态和数据的集合，因为程序运行状态实际上也是由数据来表示的（PC和CPSR等）。

从内核的角度来看，Process是一个使用资源的实体，包括CPU资源，内存资源等。

Linux系统中的多线程都是由POSIX Thread来在用户空间实现的，旧版的Linux系统没有对线程单独的优化，多线程程序从内核的角度来说没有差别。

### 仅使用process实现多线程的缺点

问题主要出在阻塞上，如果仅使用process，想要实现线程的阻塞比较复杂。

### Lightweight Process

Linux为了解决这个问题，引入了lightweight process，这种process可以指定几个之间共享资源，包括地址空间和打开的文件。

### Lightweight Process和Thread之间的关系

Lightweight Process是现代系统实现thread的方式，区别于原来的仅使用Process来实现thread，所以thread包含使用了LWP技术的thread和没使用LWP技术的thread，但是现在系统中thread和lightweight process就是一个东西。

两种thread最根本的区别是实现thread的过程中是否有kernel介入。

Linux中使用了LWP的pthread库有LinuxThreads，Native POSIX Thread Library(NPTL)还有Next Generation Posix Threading Package(NGPT)。

# Process Descriptor

存储着进程当前状态和其他所有相关信息的数据集，类型为`task_struct`。

这一章仅介绍其中的两个和process最相关的属性。

## Process State

表示process当前的状态，是由一个flag数组表示的。当前的Linux系统中所有的这些flag是互斥的，所以一个时间只会有一个被置位。

process的可能状态：

| 状态                 	|                                                                                                                            	|
|----------------------	|----------------------------------------------------------------------------------------------------------------------------	|
| TASK_RUNNING         	| 处于就绪态或者运行态                                                                                                       	|
| TASK_INTERRUPTABLE   	| 由于等待事件被挂起，可被使用信号唤醒                                                                                       	|
| TASK_UNINTERRUPTABLE 	| 由于等待事件被挂起，不可被其他信号唤醒                                                                                     	|
| TASK_STOPPED         	| 被发送了SIGSTOP, SIGTSTP, SIGTTIN, SIGTTOU信号导致进入停止态                                                               	|
| TASK_TRACED          	| 线程被调试器挂起。当被使用例如ptrace的指令进行调试的时候每种信号都会使其进入到这个状态。                                   	|
| EXIT_ZOMBIE          	| 进程已经执行结束，但是parent进程还未执行wait()系列操作，由于数据可能parent进程还需要，所以操作系统不会删除这个进程的数据。 	|
| EXIT_DEAD            	| 如果parent进程执行了wait()系列操作就会使进程进入这个状态，也是最终状态。                                                   	|

### 设置进程状态的几种方法

可以直接通过赋值方式进行：

`p->state = TASK_RUNNING`

也可以通过使用`set_task_state()`和`set_current_task_state()`两个宏操作，这两个宏操作同时预防了由于代码优化造成的指令执行位置的改变。（可能是添加了volatile）

## Identifying a Process

### 通过Process Desciptor地址

在Linux系统中，所有可以被调度的单元都需要有自己的Process Descriptor，包括LWP，根据这个条件，我们就可以通过task_struct的地址来区分各个进程。

### 通过PID

每个Process Descriptor都具有一个PID，新分配的PID通常是上一个分配的+1,直到系统开始使用到了一个PID分配的最大值时，系统开始从小到大回收已经不被使用的PID。这个最大值的默认大小为32767(PID_MAX_DEFAULT - 1)。我们可以通过设置`/proc/sys/kernel/pid_max`的值来设置这个最大值，要注意实际pid_max的值是我们设置的值-1.

32位系统中最大的`pid_max`数值为32767,而64位系统中最大的pid_max数值为4194303.

系统内通过维护了一个`pidmap_array`的一个数组来通过位图的方式显示当前pid的使用状况，是否正在使用中。由于32位系统4KB的位数刚好是32768,所以32位系统可以仅使用一个页来存储整个`pidmap_array`，64位系统的话需要额外的页来存储这个位图。

存储了`pidmap_array`的页永远不会被释放，也就是属于静态的。

#### 线程组

根据POSIX 1003.1c标准，所有线程需要具有同样的PID，方便使用进程操作命令时对同属一个进程的所有线程进行操作，但是在Linux中根据刚才说的LWP都具有自己的PID，和这个矛盾。Linux为了实现这个要求，引入了线程组的概念，线程组的组长LWP中的tgid存储着线程组的pid，使用getpid()等进程指令时，同一个线程组的所有线程都会返回这个线程组组长的tpid。

## Process Descriptor Handling

对于每一个Linux创建的进程，都会被分配一块小的进程独立内存，里面包含两个数据结构：

[图]

esp为80x86中栈顶指针。

- `thread_info`结构体（从0地址开始存储）
- 内核模式下的堆栈（从最大地址向下增长）

### thread_info结构体

这个结构体存储一些进程信息，也具有一个task属性，存储着指向process descriptor的指针，process descriptor中也有一个`thread_info`指针指向这个结构体。

通常情况下这个内存空间的大小是8KB，也就是两个Page Frame，通常存储在地址连续的两个页中。并且第一个页的地址要和$2^{13}$次方对齐。但是这个小内存在整个系统可用动态内存很小的情况下很可能会造成内存碎片化，所以80x86系统中可以设置其大小为4KB，一个页。

内核模式下的程序不怎么使用堆栈，所以8KB的内存空间在一般情况下都是足够的，但是如果这个小内存被分配在一个页里，大小是4KB的话就很可能造成栈增长覆盖掉`thread_info`，所以内核会在他们之间插入额外的栈来检测这种内存溢出。

内核进程不会被用户进程打断，所以从用户态转向内核态的时候内核态的栈永远是空的（先前的程序都执行完了）。

用来表示这段小内存的C语言结构为：

```c
union thread_union {
        struct thread_info thread_info;
        unsigned long stack[1024];
};
```

内核使用`alloc_thread_info`和`free_thread_info`宏来申请和释放这段小内存。

### Identifying the current process

主要是指内核如何识别当前进程的。内模式下栈和thread_info结构体处于同一个小内存中，并且第一页的内存有对齐，所以根据esp的值就可以知道第一页的基地址。如果是8KB大小的话，那就是esp去掉LSB开始13位，4KB的话，就是esp去掉LSB开始12位。

这个过程在内核中是通过`current_thread_info()`这个函数来实现的。

`current()`宏用于获得当前进程的process descriptor，由于task属性在`thread_info`中是第一个属性，所以`thread_info`的指针其实就是process descriptor的指针。

多CPU操作系统中，将process descriptor和内核栈绑定非常方便，每个CPU只需要通过访问本地的数据结构就能得到process descriptor。旧版系统中由于current是一个全局静态变量，多CPU下就需要使用current数组来表示每个CPU的process descriptor指针。（这种优化是一种面向对象的类的概念上的优化）

## Doubly linked lists 内核使用的双向链表

核心的一个结构体是一个具有前后指针的结构体，使用了面向对象继承的思想来实现对这个结构体的扩展，

[图]
