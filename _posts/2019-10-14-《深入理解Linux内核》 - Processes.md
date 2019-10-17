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

通常通过`alloc_thread_info()`来申请这段内存。

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

## Saving and Loding the FPU, MMX, and XMM Registers

80x86架构相关，暂略。

# Creating Processes

传统的UNIX系统在创建子进程的时候会拷贝整个地址空间，这样的话很慢，现代系统通过几个机制来避免这些问题：

- Copy On Write：子进程和parent进程都使用同一段内存，其中一个需要写这段共享的数据的时候就把这段数据拷贝一段新的然后写。
- Lightweight Processes：一组的LWP之间数据共享，所以不需要拷贝变量。
- vfork()：船舰一个和parent进程共享地址空间的子进程，子进程执行间parent进程会阻塞直到子进程退出

## The clone(), fork(), and vfork() System Calls

LWP时使用clone()函数拷贝的，clone()具有下面这些参数：

- fn：指定要执行的函数，当函数返回的时候子进程结束，并且返回的整形数作为返回值。
- arg：传递给fn的参数指针。
- flags：下面列表。
- child_stack：指定用户模式下子进程的栈指针，每次创建都应该申请一块新的栈空间给子进程。
- tls：指定子进程的TLS段的数据结构，只有在CLONE_SETTLS被设置了才有意义。
- ptid：指定一个parent进程用户空间的变量地址，将存储子进程的PID，只有CLONE_PARENT_SETTID被设置才有意义。
- ctid：相比ptid，指定的是紫禁城的用户空间的变量地址，只有CLONE_CHILD_SETTID被设置才有意义。

clone()可选的flags：

- CLONE_VM：共享内存描述符和所有的页。
- CLONE_FS：共享根目录，当前工作目录和新文件默认权限。
- CLONE_FILES：共享打开的文件表。
- CLONE_SIGHAND：共享信号处理函数表和正在阻塞和pending的信号表，如果要设置这个位那么CLONE_VM需也要被设置。
- CLONE_PTRACE：如果parent进程正在被trace，那么子进程也会被trace，通常debugger会强制这个位为1。
- CLONE_VFORK：如果正在执行vfork()就会被置位。
- CLONE_PARENT：将子进程的parent设置为调用的进程。
- CLONE_THREAD：将子进程插入到parent进程所在的tgid哈希队列中，设置相应的tgid和group_leader，如果要设置这个位，必须要先设置CLONE_SIGHAND。
- CLONE_NEWNS：子进程具有自己的目录配置，和CLONE_FS是互斥的。
- CLONE_SYSVSEM：共享System V undoable semaphore operations（第19章详细讲）。
- CLONE_SETTLS：申请一个新的TLS段给子进程。
- CLONE_PARENT_SETTID：将子进程的PID信息写到parent进程的指定变量中。
- CLONE_CHILD_CLEARTID：置位的时候内核会检测当子进程退出或者开始执行新程序的时候，清楚ctid指向的变量并且唤醒任何等待这个事件的进程。
- CLONE_DETACHED：deprecated。
- CLONE_UNTRACED：用于清除CLONE_PTRACE。
- CLONE_CHILD_SETTID：将子进程的PID写到指定的子进程的一个变量中。
- CLONE_STOPPED：强制子进程一创建就是TASK_STOPPED状态。

clone是c库自带的一个函数，调用这个函数会调用到系统调用clone，之后会将任务转向系统服务`sys_clone()`执行。但是`sys_clone()`不具有fn和arg两个参数，而是`clone()`自己将fn和arg两个参数放到自己的返回地址上，这样`clone()`执行完之后不是退回原来的地方，而是“返回”到fn函数进行执行，参数为arg。

fork()系统调用相当于设置SIGCHLD信号（***对应哪个flag***）并且其他的flags都为空，并且`child_stack`就是当前parent的stack，所以parent进程和子进程共享用户模式栈，但他们其实是基于Copy On Write，有人要对栈进行修改的时候他们两个的栈就不是一个了。

vfork()系统调用相当于设置SIGCHLD还有CLONE_VM和CLONE_VFORK，并且子进程栈指针和母进程栈指针相同。

### The do_fork() function

clone(), fork()和vfork()实际上都是由do_fork()来处理的，do_fork()有下面这些参数：

- clone_flags：clone里的flags
- stack_start：clone里的child_stack
- regs：指向当从用户模式切换到内核模式时，保存到内核模式栈的通用寄存器的值。
- stack_size：不使用，给0
- parent_tidptr：clone里的ptid
- child_tidptr：clone里的pcid

do_fork()使用一个辅助函数copy_process()来配置好进程描述符和子进程需要的其他的内核数据结构，这里先讲do_fork()都做了些什么：

1. 查询`pidmap_array`申请一个新的未使用的PID。
2. 检查母进程的ptrace属性(current->ptrace)，看母进程是否正在被debugger追踪并且希望追踪子进程(***CLONE_PTRACE??***)，如果子进程需要被追踪就设置好CLONE_PTRACE。
3. 调用`copy_process()`对进程数据结构进行拷贝，如果成功的话会返回新的进程描述符指针。
4. 如果CLONE_STOPPED被置位或者子进程需要被追踪(p->ptrace中PT_PTRACED位被置位)，那么就将子进程设置为TASK_STOPPED状态并给子进程发送一个SIGSTOP信号。
5. 如果CLONE_STOPPED没有被设置，那么将调用`wake_up_new_task()`，这个函数进行如下操作：
    1. 调整母进程和子进程的调度参数（详见第七章）。
    2. 如果子进程会和母进程运行在同一个CPU上，并且他们的CLONE_VM位没有设置，也就是不共享地址空间，那么内核会将子进程插入到就绪队列中母进程的前面，让子进程先于母进程执行。如果不这么做而导致母进程先执行的话，由于Copy On Write，母进程可能会创建很多不必要的页（***这里不是很明白***）。
    3. 与2相反，如果子进程不和母进程运行在同一个CPU上或者他们的CLONE_VM被置位，那么子进程将被插入到就绪队列里母进程后面。
6. 如果CLONE_STOPPED被设置，设置子进程为TASK_STOPPED状态。
7. 如果母进程正在被追踪，他会将子进程的PID存储在current->ptrace_message中并调用ptrace_notuify()来通知母进程的母进程，向其发送一个SIGHLD信号，也就是正在追踪他的进程他创建了个子进程。
8. 如果CLONE_VFORK被设置，母进程会被插入到一个等待队列中，等待子进程释放自己的地址空间。
9. 返回子进程的PID，函数终止。

### The copy_process() function

copy_process()的参数是do_fork()基础上再加上一个子进程的PID，他的主要步骤如下：

1. 检查clone_flags是否兼容，下面几种情况会导致出错：
    1. CLONE_NEWS和CLONE_FS同时被设置。
    2. CLONE_THREAD被设置，但是CLONE_SIGHAND没有被置位。
    3. CLONG_SIGHAND被置位，但是CLONE_VM没有被置位。
2. 调用security_task_create()和security_task_alloc()。Linux提供钩子函数允许这些函数被扩展。
3. 调用`dup_task_struct()`获取子进程的进程描述符，他的步骤主要包括：
    1. 调用__unlazy_fpu()保存当前进程的FPU，MMX，SSE/SSE2寄存器内容到current->thread_info中，之后子进程会将这个thread_info的值拷贝到子进程的进程描述符中。
    2. 调用`alloc_task_struct()`来申请一个新的进程描述符，并将其指针保存到tsk这个局部变量中。
    3. 调用`alloc_thread_info()`来申请一个新的thread_info，包括内核模式栈，并将其地址存储到ti这个局部变量中。
    4. 将当前进程的进程描述符的内容拷贝带子进程的进程描述符中，并将子进程的thread_info设置为ti。
    5. 拷贝当前进程thread_info的内容到子进程的thread_info中，并将thread_info->task的值设置为tsk。
    6. 将进程描述符中的使用计数`tsk->usage`设置为2，表示当前进程描述符正在被使用并且进程正在工作。（***这个计数器是干嘛的***）
    7. 返回新的进程描述符指针tsk。
4. 检查current->signal->rlim[RLIMIT_NPROC].rlim_cur是否小于等于当前用户已经拥有的进程数，如果是的话将会返回一个错误，除非用户有超级用户权限。进程描述符中有一个user属性，指向类型为`user_struct`的指针，包含每个用户的信息，其中有进程数（tsk->user->processes）。
5. 将4中讲的进程数加一并且增加一个user的使用计数（tsk->user->__count）。
6. 检查当前系统中的进程数没有大于系统允许的进程数（max_threads）。
7. 如果内核函数实现了某个内核模块新进程的执行域与可执行格式，要增加使用计数。（二十章会讲）
8. 设置几个和进程状态有关的关键变量：
    1. 初始化内核锁计数。`tsk->lock_depth = -1;`（第五章）
    2. 初始化执行计数。`tsk->did_exec = 0`这个东西反应这个进程执行的所有`execve()`的数目。
    3. 更新一些进程标志，包括清除PF_SUPERPRIV，表示进程还没用到任何超级用户权限，置位PF_FORKNOEXEC，表示进程还没有执行过。
9. 保存新进程的PID到`tsk->pid`。
10. 如果CLONE_PARENT_SETTID被置位，拷贝子进程PID到`parent_tidptr`变量中。
11. 初始化子进程中所有的`list_head`类型和`spin_lock`类型的结构体，并设置有关未处理信号，定时器和定时器统计信息的一些属性。
12. 调用`copy_semundo()`，`copy_files()`，`copy_fs()`，`copy_sighand()`，`copy_signal()`，`copy_mm()`，和`copy_namespace()`，根据clone指定的flag讲母进程中的一些信息拷贝到子进程中。
13. 调用`copy_thread()`来初始化子进程内核模式栈，它们的值为母进程调用到`clone()`系统调用时候的值，先前保存过。同时将eax的值设置为0（返回值）。thread.esp被初始化为子进程的内核模式栈的基地址，并且一个汇编函数`ret_from_fork()`的地址被存储在thread.eip中。如果母进程使用IO访问位图，那就也给子进程复制一份。最后如果CLONE_SETTLS位被置位，子进程的TLS段就被设置为tls指定的数据结构的值。
14. 如果CLONE_CHILD_SETTID或者CLONE_CHILD_CLEARTID被置位，内核将会拷贝tsk->set_chid_tid和tsk->clear_child_tid的child_tidptr的值到子进程对应属性中。这些标志表示用户模式下child_tidptr指向的变量需要被改变，通常是在后面改变的。
15. 关闭子进程中thread_info中的TIF_SYSCALL_TRACE标志，这样的话ret_from_fork()将不会通知调试进程有关系统调用终止的信息。但并不代表对系统调用的追踪终止了，对系统调用的追踪还是有tsk->ptrace中的PTRACE_SYSCALL控制的。
16. 将tsk->exit_signal属性设置为clone_flags中表示信号的低几位，除非CLONE_THREAD被置位，这种情况下直接被设置为-1.应为只有一个线程组的最后一个线程退出的时候才会通知线程组长的母进程。
17. 调用sched_fork()来完成调度器中新进程数据结构的初始化，同时设置新进程的状态到TASK_RUNNING，preempt_count到1，同时关闭内核的抢占。而且，为了让调度器更公平，这个函数将母进程剩下的时间片时间在子进程和母进程中共享。
18. 将thread_info->cpu设置为smp_processor_id()。
19. 初始化指定母子关系的属性，要注意的是当CLONE_PARENT和CLONE_THREAD被设置的时候，他会初始化tsk->real_parent和tsk->parent为current->real_parent，也就是将子进程的母进程设置为当前进程的母进程，否则就会将子进程的母进程设置为当前进程。
20. 如果子进程不需要被追踪，会设置tsk->ptrace为0。
21. 执行SET_LINKS宏将新进程的描述符插入到进程链表中。
22. 如果子进程设置为需要被追踪（tsk->ptrace中的PT_PTRACED被置位），那么会将tsk->parent设置为current->parent并将进程加入到调试队列中。
23. 调用attach_pid()将新进程的PID插入到PID哈希表中。
24. 如果子进程是线程组组长（CLONE_THREAD复位），那么：
    1. 将tsk->tgid初始化为tsk->pid。
    2. 将tsk->group_leader初始化为tsk。
    3. 调用三次attach_pid()将子进程添加到PIDTYPE_TGID，PIDTYPE_PGID和PIDTYPE_SID的哈希表中。
25. 否则，如果不是线程组组长，那么：
    1. 将tsk->tgid初始化为current->tgid。
    2. 将tsk->group_leader初始化为current->group_leader。
    3. 调用attach_pid()将子进程添加到PIDTYPE_TGID哈希表中。
26. nr_threads增加1。
27. total_forks增加1.
28. 返回tsk，函数结束。

所以可以通过fork()的返回值来判断是子进程还是母进程主要是对eax寄存器修改的结果。

## Kernel Threads

传统的UNIX系统使用间歇运行的守护进程执行一些重要任务，包括写缓存，换出不用的页等，这样响应很不好。由于他们基本都运行于内核空间，Linux将这些任务交付给内核线程进行执行，内核线程的特点包括：

- 内核线程只运行于内核模式，但是常规线程可能运行于内核模式或者用户模式。
- 由于内核线程只运行于内核模式，所以他们只使用虚拟地址空间的大于PAGE_OFFSET的部分，普通线程会用到整个4GB的地址空间。

### 创建一个内核线程

使用kernel_thread()来创建一个内核线程，包括三个参数，要执行的函数指针，参数指针还有clone标志位集合。

kernel_thread()内部会调用到do_fork()，使用下面的参数：

```c
do_fork(flags | CLONE_VM | CLONE_UNTRACED, 0, pregs, 0, NULL, NULL);
```

如果没有CLONE_VM，复制的用户空间地址空间根本不会被使用，会浪费资源。CLONE_UNTRACED表明了内核线程不可被TRACE。

copy_thread()会找到CPU初始化CPU寄存器的初始化值，然后把这些值所在数据结构的地址给pregs。（***这段有问题***）

主要目的是：

- ebx和edx会被设置为fn和arg。
- eip寄存器会被设置为下面这个代码段的地址：

```asm
movl %edx, %eax
pushl %edx
call *%ebx
pushl %eax
call do_exit
```

这样内核线程就开始执行fn(arg)了函数了，如果这个函数终止，内核线程会调用_exit()系统调用返回fn()的返回值。

### Process 0

0号进程是Linux系统中所有进程的祖先，叫做空闲（idle）进程，由于历史原因，也叫做交换（swapper）进程，是在Linux初始化阶段创建的进程。这个进程所有的数据结构都是静态声明的，有下面这些：

- 一个进程描述符。
- 一个thread_info和内核模式栈的小内存段。
- 下面的表：
  - init_mm
  - init_fs
  - init_files
  - init_signals
  - init_sighand
- 存储在swapper_pg_dir中的Page Global Directory地址。

start_kernel()函数初始化所有内核需要的数据结构，使能中断，创建另一个内核线程，1号进程，也叫做init进程：

```c
kernel_thread(init, NULL, CLONE_FS | CLONE_SIGHAND);
```

新创建的1号进程和0号进程共享所有的进程独立数据结构，当1号进程被调度器选中的时候，开始执行init()函数。

当1号进程开始执行init之后，0号进程开始执行cpu_idle()函数，这个函数一直循环执行一些指令，最主要是hlt这个汇编指令，并且中断都使能，这个进程只有在所有其他进程都没有处于就绪态的时候才会被选中运行。

在多CPU系统中每个CPU都有一个0号进程。系统启动时，BIOS只启动一个CPU，运行在这个CPU上的0号进程初始化需要的内核数据结构，然后启动其他CPU并使用copy_process()为他们创建0号进程，并且传递进参数0，指定他们的PID都为0.并且内核还会设置thread_info->cpu为适当的CPU ID。

### Process 1

这个进程执行的init()会调用execve()来执行init，这样init就作为一个常规进程并拥有自己的进程数据结构。init进程起到监控所有其他进程的作用，直到系统关闭才会被销毁。

### Other kernel threads

一些内核线程是初始化的时候创建的，另一些是需要的时候才创建的，下面是一些内核函数的介绍：

- keventd(events)：执行在keventd_wq队列中的函数。
- kapmd：处理有关高级电源管理的事件。
- kswapd：回收内存用。
- pdflush：将已更改的页写回磁盘来回收内存。
- kblockd：执行kblockd_workqueue队列中的函数。这个函数会间歇性的激活块设备驱动。
- ksoftirqd：运行tasklet，每个CPU都有一个这个服务。

# Destroying Processes

一般进程在执行函数退出后退出，并且需要通知内核，让内核释放资源，包括内存，打开的文件等。

通常这个过程是通过调用exit()库函数完成的，会释放所有c库申请的内存，和每个注册的退出函数，最后会调用一个系统调用将进程清除出系统。exit()函数可以手动加在进程里，c编译器总是在main函数结尾加上一个exit()。

有时候内核可能想要终止一整个线程组，通常在一个进程接收到无法处理的信号或者是CPU发生异常的时候发生。

## Process Termination

Linux 2.6中有两个系统调用来终止一个用户空间进程：

- exit_group()：终止整个线程组，主要的实现函数为do_group_exit()。这个函数通常被exit()这个c库函数调用。
- _exit()：终止单个进程，处于同一个线程组的其他进程不受影响，实现的主要函数是do_exit()。这个通常是pthread_exit()这个库函数调用的。

### The do_group_exit() function

终止所有当前线程组的进程，这个函数会受到一个退出码作为参数，正常退出时是由exit_group()这个函数传递的，非正常退出时是内核提供的错误码。

这个函数的主要流程包括：

1. 检查SIGNAL_GROUP_EXIT是否为0,如果不为0代表内核已经开始销毁这个进程组，如果这样的话，内核会认为current->signal->group_exit_code作为返回码并跳到第4步。
2. 否则将SIGNAL_GROUP_EXIT设置为1。，并将退出码设置到current->signal->group_exit_code。
3. 调用zap_other_threads()来销毁当前线程组中的其他所有进程。这个函数会在pid哈希表中找到所有相同tgid的线程并向他们发送SIGKILL信号，他们最终都会调用do_exit()退出。
4. 调用do_exit()并传递终止码。

### The do_exit() function

所有进程的结束都是这个函数处理的，他会移除大部分其他数据结构中对当前进程的引用。他的执行步骤为：

1. 置位进程描述符中的PF_EXITING来表示这个进程将被清除。
2. 如果需要的话，使用del_timer_sync()将进程描述符从动态定时器队列中移除。
3. 将进程描述符相关的数据结构，包括分页，互斥量，文件系统，打开文件描述符，命名空间和IO权限位图都移除，使用的方法为exit_mm()，exit_sem()，__exit_files()，exit_fs()，exit_namespace()还有exit_thread()，这些函数同时会释放没有进程再引用的数据结构。
4. 如果当前进程中实现执行域或者可执行格式的函数包括在内核模块中，要减计数。
5. 将进程描述符的exit_code设置为传递进来的终止码。
6. 调用exit_notify()函数，他包括下面这些步骤：
   1. 更新所有进程和他们parent进程的母子关系，所有结束进程的子进程都变为当前线程组中其他进程的子进程，如果没有其他进程了，就会变为init进程的子进程。
   2. 检查进程描述符的exit_signal是否为-1,并且他是否是线程组中最后一个进程，这个情况下会发送一个SIGHLD信号给母进程通知子进程的终止。
   3. 如果exit_signal为-1,并且他不是线程组中最后一个进程的话，只有进程在被trace的情况下才会向parent进程发送SIGHLD信号。
   4. 如果exit_signal为-1,并且没有被追踪，那么设置进程描述符的exit_state为EXIT_DEAD并且调用release_task()来回收内存，减少进程描述符计数。由于我们是最后一个使用这个进程的，他的usage_count是1,描述符不会被立刻释放。
   5. 否则，如果exit_signal不为-1或者没有被追踪，设置exit_state为EXIT_ZOMBIE。
   6. 在进程描述符中设置PF_DEAD。
7. 调用schedule()来让新进程进行运行。schedule()会检查PF_DEAD位并减少描述符的使用计数。

## Process Removal

UNIX系统中允许进程监控子进程的状态，所以一个子进程结束后他的描述符不可以立刻释放，因为母进程可能还需要查询他的执行结果，这就是为什么需要EXIT_ZOMBIE状态存在。母进程在调用wait系列函数查询子进程状态之后子进程的描述符才被正式销毁，状态也会从EXIT_ZOMBIE变为EXIT_DEAD。

母进程如果在子进程之前结束，子进程变为孤儿进程，init进程的子进程，init进程会为每一个孤儿进程在结束后调用wait系列的函数来销毁他们。

release_task()函数用于将最后一格描述符中的数据结构释放，两种情况下会被用在僵尸进程上：

- 如果母进程不关心子进程的信号，子进程的do_exit()会执行。
- 如果母进程需要子进程的信号，则需要wait4()或者waitpid()来执行。

第一种情况下描述符被调度器释放，第二种情况下描述符被wait函数释放。

release_task()函数的主要步骤：

1. 减少用户拥有的已结束进程的计数。
2. 如果进程正在被追踪，这个函数会将自己移除调试进程的ptrace_children列表并将进程设置回原来的母进程的子进程。
3. 调用__exit_signal()来取消任何还未处理的信号并释放进程的信号描述符，还会调用exit_itimers()来释放POSIX定时器。如果描述符不再被任何其他LWP所使用（***一个描述符可以多个用？***），那么这里就可以释放这个描述符了。
4. 调用__exit_sighand()来清除所有的信号处理函数。
5. 调用__unhash_process()，包括下面步骤：
   1. nr_threads减一。
   2. 调用两次detach_pid()将进程移除PIDTYPE_TGID和PIDTYPE_PID的哈希表。
   3. 如果进程是线程组组长，再调用两次将其从PIDTYPE_PGID和PIDTYPE_SID中移除。
   4. 使用REMOVE_LINKS宏将进程描述符从总进程描述符链表中去除。
6. 如果进程不是线程组组长，并且组长已经是僵尸进程而且是线程组最后一个进程，会给母进程发送信号通知进程的终止。
7. 调用sched_exit()调整母进程的时间片。（***需要补充信息，主要是为什么？时间片数目和子进程数有关？***）
8. 调用put_task_struct()来减少进程描述符的使用计数，如果计数变为0,会将所有对进程的索引断开，包括：
   1. 减少用户信息中的使用计数（__count），并且在计数变为0的时候释放（***释放的用户数据结构？？***）。
   2. 释放进程描述符的内存空间还有thread_info和内核模式栈使用的小内存段。
