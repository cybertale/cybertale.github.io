---
layout: post
title: 《UNIX环境高级编程》 - Signal
author: 宋强
tags: UNIX
date: 2017-10-08 19:28 +0800
---

# 产生信号的情况

* 在终端按下按键。
* 硬件异常，比如说被零除。
* 进程调用kill函数将信号发送给进程或者进程组。
* 用户使用kill命令，不过此命令只是在其内部调用kill函数。
* 检测到某些软件条件，例如管道终止后还有进程想要写管道。

# 信号处理的方式

* 忽略此信号。
* 捕捉信号，使用自定义函数进行处理。
* 执行系统默认操作。

# 一些信号的介绍

* SIGABRT：调用abort函数时产生此信号，进程异常终止。
* SIGALRM：调用alarm函数设置的计时器超时时，会产生此信号。
* SIGBUS：指示一个实现定义的硬件故障。
* SIGCHLD：一个进程终止时，这个信号将被发送给父进程。
* SIGCONT：此作业控制信号将被发送给需要继续执行，但是却处于停滞状态的进程。
* SIGEMT：指示一个实现定义的硬件故障。
* SIGFPE：表示算术运算异常。
* SIGHUP：如果终端接口检测到一个连接断开，将把这个信号发送给相关的控制进程。如果会话首进程终止，这个信号也将被发送给前台进程组中的每一个进程。
* SIGILL：执行了一条非法硬件指令。
* SIGINT：用户按下中断键（一般是CTRL+C还有DELETE）时，这个信号将被发送给前台进程组中的没一个进程。
* SIGIO：指示一个异步IO事件。
* SIGIOT：指示一个实现定义的硬件故障。
* SIGKILL：两个不能被捕捉或者忽略的信号之一，提供给系统管理员终止进程用。
* SIGPIPE：如果在写管道时读进程终止，则会产生一个此信号。
* SIGPOLL：当在一个可轮询设备上发生特定事件时将会产生此信号。
* SIGPROF：setitimer函数设置的梗概统计间隔计数器已到期时产生此信号。
* SIGPWR：依赖于系统的信号，主要用于具有UPS的系统，如果电源失效，UPS开始工作，软件会接收到这个信号，存储信息并终止。
* SIGQUIT：当用户在终端上按退出键时，会产生此信号，发送到前台进程组中的所有进程，但并不会终止前台进程。
* SIGSEGV：指示进行了一次无效的内存引用。
* SIGSTKFLT：用于指示数学协处理器的栈故障。
* SIGSTOP：作业控制信号，用于停止一个进程，不可以被捕捉或者忽略。
* SIGSYS：指示一个无效的系统调用。
* SIGTERM：由kill发送的系统默认终止信号。
* SIGTRAP：指示一个实现定义的硬件故障。
* SIGTSTP：交互式停止信号，用户在终端按下挂起键（CTRL+Z）时产生此信号，送至前台进程组的所有进程。
* SIGTTIN：当有一个后台进程组中的进程试图读取控制终端时，将会产生此信号。但是当读进程忽略或者阻塞此信号或者读进程所属的进程组时孤儿进程组时将不会产生此信号。
* SIGTTOU：当一个后台进程组中的进程试图写到控制终端时将会产生此信号。
* SIGURG：此信号通知进程发生了一个紧急情况。
* SIGUSR1：用户自定义信号。
* SIGUSR2：用户自定义信号。
* SIGVTALRM：由setitimer函数设置的虚拟间隔时间到期时产生此信号。
* SIGWINCH：内核维持与每个终端或伪终端相关联的窗口大小。
* SIGXCPU：如果进程超过了其软CPU时间时限，产生此信号。
* SIGXFSZ：如果进程超过了其软文件长度限制，那么将产生此信号。

# signal

```c++
#include <signal.h>
void (*signal (int signo, void (*func)(int)))(int);
```

利用这个函数注册一个信号捕捉函数。

进程调用exec之后会将所有信号响应方式变为默认状态。

# kill和raise

```c++
#include <signal.h>
int kill(pid_t pid, int signo);
int raise(int signo);
```

raise(signo)等价于kill(getpid(), signo)。

kill的pid分为四种情况：

* pid>0：将该信号发送给进程号为pid的进程。
* pid=0：将该进程号发送给属于同一进程组的所有进程。
* pid<0：将该信号发送给进程组ID为pid绝对值的所有进程，需要具有权限。
* pid=-1：发送给所有具有权限发送的进程。

当signo为0时，kill将不会发送信号，kill将会利用这个来检查相应进程是否仍然存在，如果不存在返回-1。

# alarm和pause

```c++
#include <unistd.h>
unsigned int alarm(unsigned int seconds);
int pause(void);
```

alarm将会设置一个计数器，到时后会产生SIGALRM信号。

pause函数会将进程挂起，直到进程捕捉到一个信号。

sleep通常是由这两个函数的组合实现的。

还可以用于设置阻塞程序阻塞时间上限。

# 信号集

POSIX.1定义了数据类型sigset_t用于表示信号集，还有五个信号集操作函数：

```c++
#include <signal.h>
int sigemptyset(sigset_t *set);
int sigfillset(sigset_t *set);
int sigaddset(sigset_t *set, int signo);
int sigdelset(sigset_t *set, int signo);
int sigismember(const sigset_t *set, int signo);
```

强两个信号都是初始化一个信号集，一个是不包含任何信号，一个是包含所有信号，然后两个是增删信号，最后一个是判断信号是否在信号集内，在内返回1，否则返回0。

# sigprocmask

```c++
#include <signal.h>
int sigprocmask(int how, const sigset_t *restrict set, sigset_t *restrict oset);
```

用于设置进程信号的屏蔽位，如果一个信号被屏蔽，那么将不能传送给这个进程。

若oset为非空指针，进程的信号屏蔽字将通过oset返回。

若set是一个非空指针，那么how会指示如何修改信号屏蔽字：

|     how     |                  说明                 |
|:-----------:|:-------------------------------------:|
|  SIG_BLOCK  |              屏蔽某个信号             |
| SIG_UNBLOCK |            解除屏蔽某个信号           |
| SIG_SETMASK | 信号屏蔽字将直接使用set指向的信号集的 |

# sigpending

```c++
#include <signal.h>
int sigpending(sigset_t *set);
```

返回当前被屏蔽的信号集。

# sigaction

```c++
#include <signal.h>
int sigaction(int signo, const struct sigaction *restrict actm struct sigaction *restrict oact);
```

用于代替UNIX早期版本中的signal函数，signo是信号编号，若act非空，那么修改为act动作，如果oact非空，那么通过oact返回原来的动作。

sigaction的结构如下：

```c++
struct sigaction{
        void (*sa_handler)(int);
        sigset_t sa_mask;
        int sa_flags;
        void (*sa_sigaction)(int, siginfo_t *, void *);
};
```

sa_handler也可以为SIG_IGN或者SIG_DFL。

sa_mask是一个信号集，在调用捕捉函数之前这个信号集会被加入到进程的信号屏蔽字中。

sa_flags指定对信号处理的各个选项：

|     选项     |                                                           说明                                                           |
|:------------:|:------------------------------------------------------------------------------------------------------------------------:|
| SA_INTERRUPT |                                            此信号中断的系统调用不会重新启动。                                            |
| SA_NOCLDSTOP | 若signo时SIGCHLD，当子进程停止时，不产生此信号，当进程终止时，仍然产生信号，若已设置此标志，当继续运行时，会产生此信号。 |
| SA_NOCLDWAIT |                                       若是SIGCHLD，则子进程终止时，不创建僵死进程。                                      |
|  SA_NODEFER  |                                        捕捉到此信号在执行函数过程中，不自动阻塞。                                        |
|  SA_ONSTACK  |                               若用sigaltstack生命了一替换栈，将此信号发送给替换栈上的进程。                              |
| SA_RESETHAND |                      此信号捕捉函数的入口处，将此信号的处理方式复位为SIG_DFL，并清除SA_SIGINFO标志。                     |
|  SA_RESTART  |                                           此信号终端的系统调用会自动重新启动。                                           |
|  SA_SIGINFO  |                                               为信号处理选项提供附加信息。                                               |

siginfo_t包含了信号产生原因的有关信息，在新的linux中这个结构体较为复杂，但都至少要包含si_signo和si_code，这两个字段可以确定错误原因：

| si_signo |    si_code    |              原因             |
|:--------:|:-------------:|:-----------------------------:|
|  SIGILL  |   ILL_ILLOPC  |           非法操作码          |
|          |   ILL_ILLOPN  |           非法操作数          |
|          |   ILL_ILLADR  |          非法地址模式         |
|          |   ILL_ILLTRP  |            非法陷入           |
|          |   ILL_PRVOPC  |           特权操作码          |
|          |   ILL_PRVREG  |           特权寄存器          |
|          |   ILL_COPROC  |          协处理器出错         |
|          |   ILL_BADSTK  |           内部栈出错          |
|  SIGFPE  |   FPE_INTDIV  |           整数除以0           |
|          |   FPE_INTOVF  |            整数溢出           |
|          |   FPE_FLTDIV  |           浮点除以0           |
|          |   FPE_FLTOVF  |            浮点上溢           |
|          |   FPE_FLTUND  |            浮点下溢           |
|          |   FPE_FLTRES  |         浮点结果不精确        |
|          |   FPE_FLTINV  |         无效的浮点运算        |
|          |   FPE_FLTSUB  |            下标越界           |
|  SIGSEGV |  SEGV_MAPERR  |        地址未映射到对象       |
|          |  SEGV_ACCERR  |       对于映射对象无权限      |
|  SIGBUS  |   BUS_ADRALN  |         无效的地址对齐        |
|          |   BUS_ADRERR  |        不存在的物理地址       |
|          |   BUS_OBJERR  |       对象特有的硬件出错      |
|  SIGTRAP |   TRAP_BRKPT  |          进程断点陷入         |
|          |   TRAP_TRACE  |          进程跟踪陷入         |
|  SIGCHLD |   CLD_EXITED  |          子进程已终止         |
|          |   CLD_KILLED  |   子进程已异常终止（无core）  |
|          |   CLD_DUMPED  |   子进程已异常终止（有core）  |
|          |  CLD_TRAPPED  |      被跟踪的子进程已陷入     |
|          |  CLD_STOPPED  |          子进程已停止         |
|          | CLD_CONTINUED |       停止的子进程已继续      |
|  SIGPOLL |    POLL_IN    |            数据可读           |
|          |    POLL_OUT   |            数据可写           |
|          |    POLL_MSG   |          输入消息可用         |
|          |    POLL_ERR   |             IO出错            |
|          |    POLL_PRI   |        高优先级消息可用       |
|          |    POLL_HUP   |          设备断开连接         |
|    Any   |    SI_USER    |         kill发送的信号        |
|          |    SI_QUEUE   |       sigqueue发送的信号      |
|          |    SI_TIMER   | timer_settime设置的计时器超时 |
|          |   SI_ASYNCIO  |         异步IO请求完成        |
|          |    SI_MESGO   |      一条消息到达消息队列     |

# sigsetjmp和siglongjmp

```c++
#include <setjmp.h>
int sigsetjmp(sigjmp_buf env, int savemask);
void siglongjmp(sig_jmp_buf, int val);
```

和原来的两个函数的区别就是sigsetjmp增加了一个参数savemask，如果savemask非零，那么在调用的时候就会保存进程的当前信号屏蔽字，调用siglongjmp的时候如果保存了屏蔽字，就会从中恢复屏蔽字。

# sigsuspend

```c++
#include <signal.h>
int sigsuspend(const sigset_t *sigmask);
```

用于提供原子操作使解除阻塞和pause成为一体，让信号在pause之后被接收到。

执行之后会将信号屏蔽字设置为sigmask指向的值。

# abort

```c++
#include <stdlib.h>
void abort(void);
```

此函数将给调用进程发送一个SIGABRT信号，规定调用方式为raise(SIGABRT)。