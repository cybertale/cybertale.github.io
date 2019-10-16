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
| TASK_INTERRUPTIBLE   	| 由于等待事件被挂起，可被使用信号唤醒                                                                                       	|
| TASK_UNINTERRUPTIBLE 	| 由于等待事件被挂起，不可被其他信号唤醒                                                                                     	|
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

### Doubly linked lists 内核使用的双向链表

核心的一个结构体是一个具有前后指针的结构体，使用了面向对象继承的思想来实现对这个结构体的扩展，

[图]

Linux中使用的双向链表的头指针是一个指向了只具有前后指针的dummy头，类型是`list_head`，其他的被包含的节点的类型是`list_node`。

| list操作                     	|                                                                                                                       	|
|------------------------------	|-----------------------------------------------------------------------------------------------------------------------	|
| list_add(n, p)               	| 把n指向的新节点插在p指向的节点的后面。                                                                                	|
| list_add_tail(n, p)          	| 把n指向的新节点插在p指向的节点的前面。                                                                                	|
| list_del(p)                  	| 删除p指向的节点。                                                                                                     	|
| list_empty(p)                	| 查看链表是否为空。                                                                                                    	|
| list_entry(p, t, m)          	| 我们的外部结构体是t，并且里面list_head的名字是m，返回p指向的节点的子类对象t类型的结构体的指针。                       	|
| list_for_each(p, h)          	| h是链表的头指针，每个循环体内，p会包含遍历的节点的指针。                                                              	|
| list_for_each_entry(p, h, m) 	| h是链表的头指针，每个循环体内，返回包含了list_head的子类对象的地址。（为什么不需要t？to_container_of应该是需要t才对） 	|

内核也会使用单向链表，双向链表的优势是找到最后一个元素的时间是O(1)的，但是有的时候并不需要这个优势，而且单项链表节省非常多的内存空间。内核中的Hash结构就经常使用单向链表。

单向链表对应的头是`hlist_head`，节点是`hlist_node`，操作也是前面加h，包括`hlist_add`, `hlist_del`, `hlist_empty`, `hlist_entry`, `hlist_for_each_entry`等。

### The process list

这个链表组织的对象是process descriptor，里面没有dummy头，第一个插入的节点是`init_task`对象，也就是0号进程（swap进程，交换进程）。

使用`SET_LINKS`和`REMOVE_LINKS`宏来在process list中插入和删除对象。

使用`for_each_process(p)`宏来遍历整个链表，会遍历`init_task`到最后一个节点。

### The list of TASK_RUNNING processes

在每次调度的时候内核需要找到优先级最高的处于就绪态的进程，在2.6之前的内核中，内核实现了一个链表存储着所有处于就绪态的进程的descriptor指针，但是并不是按优先级排布的（耗时间），同时也决定了这样的话链表的元素越多，找到一个优先级最高的元素所需要花费的时间越长，理论上是O(n)。

2.6之后引入了哈希表来解决这个问题，内核准备了140个优先级，同时有140个descriptor链表，按照优先级向哈希表中的链表插入元素，这样找到就绪态中的元素的时间就变成了O(1)。

其中存储着这个哈希表的结构是

```c
typedef struct prio_array_t {
        int nr_active;          //有多少process
        unsigned long bitmap[5];        //优先级位图，当相应优先级链表内有元素时被置位，否则被复位。
        struct list_head queue[140];    //就绪态进程descriptor的哈希表。
};
```

使用`enqueue_task(p, array)`和`dequeue_task(p, array)`来插入和删除元素。array是指`prio_array_t`的实例指针，p是相应descriptor的指针。

# Relationships Among Processes

process之间的关系包括亲子关系，sibling关系，为了描述process之间的这些关系，process descriptor中引入了下面几个属性：

| 属性名      	|                                                                                                                   	|
|-------------	|-------------------------------------------------------------------------------------------------------------------	|
| real_parent 	| 指向创建他的进程的descriptor。如果创建他的process已经退出，那么会指向init进程的descriptor。                       	|
| parent      	| 指向他的当前parent，大多数情况下这个值都和real_parent相同，一般trace的时候会用到，让trace进程变成他的parent进程。 	|
| children    	| 指向一个包含了所有他的child进程的链表。                                                                           	|
| sibling     	| 包含一个前指针和一个后指针指向sibling进程。                                                                       	|

注意到Linux系统中0号进程和1号进程都是内核创建的，其他所有进程都是1号进程开始创建的。

进程之间具有一些其他的非亲属之间的关系，包括process group leader, thread group leader等。

| 属性名          	|                                                                                                 	|
|-----------------	|-------------------------------------------------------------------------------------------------	|
| group_leader    	| group_leader的descriptor的指针                                                                  	|
| signal->pgrp    	| 进程的group leader的PID                                                                         	|
| tgid            	| 进程的thread leader的PID                                                                        	|
| signal->session 	| 进程的login session leader的PID                                                                 	|
| ptrace_children 	| 进程的所有正在被ptrace监控的子进程的链表的头的指针                                              	|
| ptrace_list     	| 当当前进程正在被trace的时候，返回被trace的sibling进程的链表中，在他前面或者在他后面的元素的指针 	|

### The pidhash table and chained list

使用PID来获取process descriptor不是很容易，Linux中同一个进程有多个PID，主要是以下四种：

| 属性名       	|         	|                     	|
|--------------	|---------	|---------------------	|
| PIDTYPE_PID  	| pid     	| 最基础的PID         	|
| PIDTYPE_TGID 	| tgid    	| thread组leader PID  	|
| PIDTYPE_PGID 	| pgid    	| process组leader PID 	|
| PIDTYPE_SID  	| session 	| session组leader PID 	|

Linux使用哈希表来存储这些descriptor，所以总共有四个哈希表，在内核初始化的时候动态申请，存储在`pid_hash`这个数组中。每个哈希表的大小依赖于总的可用RAM大小。

使用`pid_hashfn(x)`这个宏来实现将pid转换成一个哈希表index。

这些哈希表中，节点的数据结构如下：

```c
struct {
        int nr;                         //pid
        struct hlist_node pid_chain;    //chain中前后对象的指针
        struct list_head pid_list;      //同一个pid下前后对象的指针
};
```

***这里有些疑问需要加强理解***

per-pid链表是指哈希链表中所有由同样pid值的节点构成的链表，也是这个哈希表中我们要查询的东西。

推测pid_chain代表的是内核中包括所有进程的pid链表。

下面介绍一些对pid哈希表进行操作的宏：

- `do_each_task_pid(nr, type, task)`：
- `while_each_task_pid(nr, type, task)`：对指定type下，pid都为nr的process descriptor进行遍历。
- `find_task_by_pid_type(type, nr)`：返回指定的process descriptor，如果不匹配就返回NULL。
- `find_task_by_pid(nr)`：相当于`find_task_by_pid_type(PIDTYPE_PID, nr)`。
- `attach_pid(task, type, nr)`：把task指向的process descriptor插入到类型type， pid为nr的队列里。如果pid已经存在，就直接插入到per-pid队列中。
- `detach_pid(task, type)`:从per-pid队列中删除。这里会查询如果这个pid在被删除时不再任何其他哈希表中被引用，代表这个pid已经没有对应的进程，将会从PID位图中被释放。
- `next_thread(task)`：返回同样tgid的per-pid链表中的下一个LWP的process descriptor。

# How Processes Are Organized

前面讲过，就绪态进程有一个专门的链表进行维护，那么其他状态的进程呢？Linux分两种情况来处理：

- 处于TASK_STOPPED, EXIT_ZOMBIE和EXIT_DEAD状态下的进程：没有专门的队列维护，这些进程只会通过pid或者他们的parent进程访问。
- 处于TASK_INTERRUPTIBLE和TASK_UNINTERRUPTIBLE的进程：先根据各种等待的事件进行分类，然后加入到每种事件的等待队列（wait queue）中。

## Wait queues

等待队列在Linux中的主要应用在中断事件处理、进程同步与定时器。

进程等待某种事件的时候会将自己加入到相应的等待队列中并放弃控制权，等待被内核唤醒。

等待队列的队列头的数据结构：

```c
struct __wait_queue_head {
        spinlock_t lock;
        struct list_head task_list;
};
typedef struct __wait_queue_head wait_queue_head_t;
```

队列中元素的数据结构：

```c
struct __wait_queue {
        unsigned int flags;
        struct task_struct *task;       //目标进程descriptor
        wait_queue_func_t func;         //如何唤醒进程的方法，第7章详细讨论，默认情况下使用的是default_wake_function()
        struct list_head task_list;     //前后指针，指向等待同一事件的节点
};
typedef struct __wait_queue wait_queue_t;
```

flag中包含一些标志信息，包括是否对等待的资源的获取是互斥的。

## Handling wait queues

使用`DECLARE_WAIT_QUEUE_HEAD(name)`来静态声明和初始化等待队列头，使用`init_waitqueue_head(q, task)`来动态初始化等待队列头。

`DEFINE_WAIT`宏会声明一个等待队列头并且将当前正在运行的进程插入到等待队列中，然后使用`autoremove_wake_function()`作为要使用的唤醒函数。

`autoremove_wake_function()`会调用`default_wake_fcuntion()`唤醒进程，之后将其移除等待队列。

使用`init_waitqueue_func_entry()`控制初始化等待队列对象的唤醒函数。

对等待队列的元素进行操作：

- `add_wait_queue()`：插入一个非互斥进程到队列头。
- `add_wait_queue_exclusive()`：插入一个互斥进程到队列尾。
- `remove_wait_queue()`：从等待队列中移除一个进程。
- `waitqueue_active()`：查询当前等待队列是否为空。

## 使进程进入睡眠的等待队列的操作

### sleep_on()

将当前进程加入到某个等待队列中。

```c
void sleep_on(wait_queue_head_t *wq)
{
        wait_queue_t wait;
        init_waitqueue_entry(&wait, current);
        current->state = TASK_UNINTERRUPTIBLE;
        add_wait_queue(wq, &wait);
        schedule();
        remove_wait_queue(wq, &wait);
}
```

这个函数也是一个典型的如何将一个进程加入到等待队列的操作流程。

### interruptible_sleep_on()

和`sleep_on()`的唯一差别就是任务进入的是TASK_INTERRUPTIBLE状态。

### sleep_on_timeout(), interruptible_sleep_on_timeout()

相比前两个增加了timeout参数。

核心差别是他们的`sleep_on()`中使用的是`schedule_timeout()`而不是`schedule()`。

### prepare_to_wait(), prepare_to_wait_exclusive(), finish_wait()

2.6版本中刚刚加入的另一种使当前进程进入睡眠的操作，典型操作流程如下：

```c
DEFINE_WAIT(wait);
prepare_to_wait_exclusive(&wq, &wait, TASK_INTERRUPTIBLE);
...             //这段可以干嘛现在还不确定
if (!condition)
        schedule()
finish_wait(&wq, &wait);
```

### wait_event(), wait_event_interruptible()

等待某个事件，实现方式为：

```c
DEFINE_WAIT(__wait);
for (;;) {
        prepare_to_wait(&wq, &__wait, TASK_UNINTERRUPTIBLE);
        if (condition)
                break;
        schedule();
}
finish_wait(&wq, &__wait);
```

`sleep_on`系列函数无法根据自定义的表达式进行唤醒，而且经常引起竞争，所以不是很鼓励使用。

如果想要互斥插入等待事件队列，必须调用`prepare_to_wait_exclusive()`或者`add_wait_queue_exclusive()`，其他的都不是互斥的。

除非使用了`DEFINE_WAIT`或者`finish_wait()`, 否则唤醒的时候必须先唤醒进程再移除进程。

内核唤醒睡眠中的方法包括`wake_up`, `wake_up_nr`, `wake_up_all`, `wake_up_interruptible`, `wake_up_interruptible_nr`, `wake_up_interruptible_all`, `wake_up_interruptible_sync`还有`wake_up_locked`。

不带有`sync`的所有方法会在唤醒的时候判断是否有更高优先级的进程被唤醒，是否需要调度，而带有`sync`的方法表示唤醒行为不会进行这个判断。

`wake_up_lock`方法相比一般的`wake_up`方法不同之处在于可以在队列中的锁已经被锁住的时候调用。***不知用处在何***

wake_up的一个典型实现如下：

```c
void wake_up(waie_queue_head_t *q)
{
        struct list_head *tmp;
        wait_queue_t *curr;

        list_for_each(tmp, &q->task_list) {
                curr = list_entry(tmp, wait_queue_t, task_list);
                if (curr->func(curr, TASK_INTERRUPTIBLE | TASK_UNINTERRUPTIBLE,,
                                0, NULL) && curr->flags))
                        break;
        }
}
```

这个函数遍历等待队列，或缺wait_queue_t类型对象然后尝试使用他们的唤醒函数唤醒，当成功唤醒了任何一个互斥等待的进程的时候退出。因为在一个队列中非互斥的都被在队列头插入，互斥的都被插到尾部，所以唤醒一个互斥的时候非互斥的一定都已经被尝试唤醒过了。不过通常一个等待队列不会出现互斥和非互斥同时存在的情况。

# Process Resource Limits

每个进程有一系列的资源闲置，包括如下资源：

- RLIMIT_AS：地址空间最大字节数，每次进程申请空间的时候内核都会检查这个值。
- RLIMIT_CORE：core dump文件大小限制，如果值为0内核将不会创建core dump文件。
- RLIMIT_CPU：进程可以使用CPU的秒数，如果超时进程将收到一个SIGXCPU信号，之后如果进程没有终止的话系统会向其发送SIGKILL信号。
- RLIMIT_DATA：heap大小限制。
- RLIMIT_FSIZE：可允许增大的最大文件大小限制，如果进程想要将一个文件的大小扩展到比这个值大，将会收到一个SIGXFSZ信号。
- RLIMIT_LOCKS：最大锁限制。
- RLIMIT_MEMLOCK：最大不可换出内存限制。内核在每次进程申请`mlock()`或者`mlockall()`申请锁定内存中的page frame的时候检查。
- RLIMIT_MSGQUEUE：POSIX消息队列中允许的最大字节数。
- RLIMIT_NOFILE：最大可打开的文件数。
- RLIMIT_NPROC：用户可拥有的最大进程数（***子进程？***）。
- RLIMIT_RSS：进程可拥有的最大page frame数。
- RLIMIT_SIGPENDING：最大可pending的信号数目。
- RLIMIT_STACK：最大栈大小限制。

所有这些限制都存储在process descriptor中的signal的rlim域中，rlim是一个包含上面所有限制的数组，类型为
```c
struct rlimit {
        unsigned long rlim_cur;
        unsigned long rlim_max;
};
```
普通用户可以设置rlim_cur，rlim_max是rlim_cur可以设置的最大值，只有有超级权限的用户才可以通过`getrlimit()`和`setrlimit()`设置rlim_max。

大多数资源的大小闲置都是RLIM_INFINITY，也就是不限制。

# Process Switch

## Hardware Context

硬件上下文就是当前的CPU状态，主要的硬件上下文信息存储在进程descriptor中，剩下的部分存储在kernel stack。

旧版的Linux系统使用80x86架构带的far jump功能进行硬件上下文切换，但是2.6以后Linux改为了使用软件上下文切换，使用软件上下文切换的主要原因为：

- 硬件上下文切换的话无法很好的对切换参数进行校验，有可能出现切换为病毒软件执行的情况。
- 软硬件上下文切换的耗时差距不大，而且软件的话将来还有优化空间。

上下文切换只会在内核空间执行，用户空间使用的寄存器信息将会被保存在内核模式的栈中。

## Task State Segment

TSS本来用于硬件上下文切换，但是虽然Linux不使用硬件上下文切换，仍然需要使用TSS，这是因为80x86在有些情况下还是会使用TSS中的数据：

- 从用户空间切换到内核空间的时候，内核会从TSS中取得内核模式栈的地址。
- 用户空间使用了in或者out命令的时候，系统需要检查TSS中的IO权限位图。

在每次硬件上下文切换的时候，内核会将TSS更改成要执行的进程需要的，只有正在CPU上运行的进程有TSS值，没有在运行的进程没有维护TSS。

每个TSS具有一个8字节的TSS descriptor，包括32位的TSS开始地址和20位的Limit域。

在Intel的原设计中，每个进程要指向他自己的TSS，因为Linux中同一个CPU上的所有进程使用的都是同一个TSS，所以TSS的busy位永远是1.

TSSD被存储在GDT中，tr寄存器存储TSSD的基址和大小限制。

### The thread field

从上文可知当前进程的上下文没办法被储存在TSS中，Linux在进程descriptor中有一个`thread_struct`类型的field属性，用于存储大部分硬件上下文，但不包括通用寄存器，通用寄存器的值被存储在内核模式栈中。

## Performing the process switch

内核只会在schedule中进行上下文切换，切换上下文主要是两步：

1. 切换Page Global Directory的地址，相当于切换了进程的虚拟内存空间。
2. 切换内核模式栈和硬件上下文。

### The switch_to macro

上面得第二步是由`switch_to(prev, next, last)`这个宏完成的，这个宏里的前两个参数，一个是置换之前的进程描述符指针，一个是要置换的进程描述符的指针，last是最特别的，这里要讲到整个切换的过程。

假设我们要将A进程切换到B进程，那么执行切换的时候，prev = A, next = B，之后我们进入B的执行流程。 在之后的某一刻，我们要切换回A进程，大概率现在运行的不是B进程，假设是C进程，那么执行切换的时候，prev = C，next = A，但是一旦进入到A的执行流中，现场会被还原成A中的现场，也就是prev = A, next = B，这个时候我们就丢失了对C的索引，所以last主要是用来保存C的。所以`switch_to`的调用形式一般为`switch_to(A, B, A)`, `switch_to(C, A, C)`。

假设schedule的代码段如下，注意prev与next的值的变化：

```c
void schedule()
{
        ...
        switch_to(A, B, A);     //prev = A, next = B    从A切换到B
        ...
}       //prev = C, next = B. 此时是从C返回的，如果不特殊保存C的索引C的索引就丢失了
```

### switch_to的编程实现

switch_to是用内联汇编编写的，
```asm
"asm"{
        ;把prev和next保存起来
        movl prev, %eax         prev -> eax
        movl next, %edx
        ;保存eflags和ebp寄存器
        pushfl
        pushl %ebp
        movl %esp, 484(%eax)    ;484偏移是prev->thread.esp，就是把esp保存起来
        movl 484(%edx), %esp    ;读取next的esp
        movl $1f, 480(%eax)     ;将label 1的地址存储在prev->thread.eip中，当prev被唤醒之后会执行label 1的程序
        pushl 480(%edx)         ;保存next的eip
        jmp __switch_to         ;跳转到c语言部分
1:
        popl %ebp               ;当A重新获取CPU的时候一开始就开始恢复ebp和eflags
        popfl
        movl %eax, last         ;从eax恢复正在转换的prev(C)
}
```

***硬件上下文切换的时候eax不变？？***

### __switch_to函数

函数原型为
```c
__switch_to(struct task_struct *prev_p, struct task_struct *next_p);
```

执行`__switch_to`的时候，他的两个参数来自于%eax和%edx，而不是栈或者通用寄存器，所以需要特殊的编译器指令来实现。使用了`__attribute__`和`regparm`两个关键字，并且这两个关键字都是非标准c关键字，是gcc独有的。这里并未将他们怎么工作的，需要单独学习。

这个函数的主要工作内容：

```c
___switch_to(struct task_struct *prev_p, struct task_struct *next_p);
{
        __unlaze_fpu();         //判断是否需要保存prev的FPU，MMX和XMM寄存器的内容，需要的话就会保存，下一小节会讲。
        cpu = smp_processor_id();       //获取当前CPU ID，这个宏从current->thread_info.cpu中获取这个ID，并存储到本地的cpu变量中。init_tss[cpu].esp0 = next_p->thread.esp;       //把next_p->thread.esp0恢复到当前CPU的TSS的esp0中。
        cpu_gdt_table[cpu][6] = next_p->thread.tls_array[0];  //恢复三个TLS段。
        cpu_gdt_table[cpu][7] = next_p->thread.tls_array[1];
        cpu_gdt_table[cpu][8] = next_p->thread.tls_array[2];

        "asm" { //保存fs和gs
                movl %fs, 40(%esi)      //fs -> prev_p->thread.fs
                movl %gs, 44(%esi)
                
                movl 40(%ebx), %fs
                movl 44(%ebx), %gs      //如果之前的fs和gs不为零（代表有有用值），那么久要恢复他们
        }

        //debug寄存器只有debugger需要，并且有人使用过了（7号值不为0）才需要恢复。
        if (next_p->thread.debugreg[7]) {
                loaddebug(&next_p->thread, 0);
                loaddebug(&next_p->thread, 1);
                loaddebug(&next_p->thread, 2);
                loaddebug(&next_p->thread, 3);
                /* 没有4，5 */
                loaddebug(&next_p->thread, 6);
                loaddebug(&next_p->thread, 7);
        }

        //更新当前CPU TSS中的io权限位图。只有当两个进程之中至少有一个有自定义位图的时候才需要。 
        //handle_io_bitmap中，如果next_p没有自己的位图，io_bitmap的值将被设置为0x8000，如果有的话将被设置为0x9000，由于这两个值都超过了TSS的索引上嫌，所以会出发一个"General Protection"异常，异常中会判断io_bitmap的值，如果是0x8000的话就向用户空间报错，如果是0x9000的话就将io_bitmap的值设置为真正的位图偏移(104),并且强制恢复执行。
        if (prev_p->thread.io_map_ptr || next_p->thread.io_map_ptr)
                handle_io_bitmap(&next_p->thread, &init_tss[cpu]);

        return prev_p;  //这个才是保证在整个汇编执行过程中eax一直存储着要被替换进程的描述符地址的地方，由于返回值默认存储在eax，所以prev_p就又被存储在了eax里。
}
```
