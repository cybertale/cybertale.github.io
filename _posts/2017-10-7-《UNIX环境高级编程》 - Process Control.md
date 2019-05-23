---
layout: post
title: 《UNIX环境高级编程》 - Process Control
author: 宋强
tags: UNIX
date: 2017-10-07 20:06 +0800
---

# 进程标识符

可以用一下函数获得进程有关的一些识别信息：

```c++
#include <unistd.h>
pid_t getpid(void);       //进程号
pid_t getppid(void);      //父进程号
uid_t getuid(void);       //实际用户号
uid_t geteuid(void);      //有效用户号
gid_t getgid(void);       //实际组号
gid_t getegid(void);      //有效组号
```

# fork

```c++
#include <unistd.h>
pid_t fork(void);
```

fork创建一个子进程，两个进程共享代码段，但是数据段，包括堆和栈等，子进程都有一个副本。

fork“调用一次，返回两次”，一次父进程返回子进程ID号，一次子进程返回0。

通常很多应用里会在子进程后使用exec，所以并不在意父进程的代码段怎么样。

# 继承的属性

子进程会继承很多父进程的属性，包括：

* 打开的文件描述符
* 实际用户ID、实际组ID、有效用户ID、有效组ID
* 附加组ID
* 进程组ID
* 会话ID
* 控制终端
* 设置用户ID标志和设置组ID标志
* 当前工作目录
* 根目录
* 文件模式创建屏蔽字
* 信号屏蔽和安排
* 针对任意打开文件描述符的在执行时关闭（close-on-exit）标志
* 环境
* 连接的共享存储段
* 存储映射
* 资源映射

父子之间的区别是：

* fork返回值不同
* 进程ID不同
* 父进程ID不同
* 子进程的tms_utime、tms_stime、tms_cutime和tms_ustime均被设置为0
* 父进程设置的文件锁不会被子进程继承
* 子进程的未处理的闹钟会被清空
* 子进程的未被处理的信号将被设置为空集

# vfork

由于很多子进程会立即调用exec，vfork就是为了这种情形而制作的fork的变体，他不会复制父进程的代码段，并且保证子进程优先执行，在子进程调用exit或者exec之后，父进程才会得以执行。

# exit

子进程结束之后必须由父进程进行善后处理，如果父进程在子进程结束之前结束了，那么子进程会被init进程领养，init进程一定会进行处理，而如果父进程在子进程之后结束但是并没有进行善后处理，就会使子进程成为一个僵死进程（Zombie）。

# wait和waitpid

```c++
#include <sys/wait.h>
pid_t wait(int *statloc);
pid_t waitpid(pid_t pid, int *statloc, int options);
```

子进程结束时，会异步向父进程附送一个SIGCHLD信号，这两个函数用于阻塞等待这个信号，但是waitpit的options也提供了一个选项可以非阻塞查询。

statloc指向存放进程终止状态的单元，如果不在意可以给其空指针。

检查终止状态的宏：

|          宏          |                            说明                            |
|:--------------------:|:----------------------------------------------------------:|
|   WIFEXITED(status)  |                          正常返回                          |
|  WIFSIGNALED(status) | 这种情况可以使用WTERMSIG(status)取得使子进程终止的信号编号 |
|  WIFSTOPPED(status)  |   暂停状态，可以使用WSTOPSIG来取得使子进程暂停的信号编号   |
| WIFCONTINUED(status) |                     暂停后又继续的进程                     |

waitpid中的pid值含义如下：

* pid == -1：等待任一子进程。
* pid  > 0：等待其进程ID与pid相等的子进程。
* pid == 0：等待其组ID等于调用进程组ID的任意子进程。
* pid < -1：等待其组ID等于pid绝对值的进程

options有三种情况：

* WCONTINUED：若实现支持作业控制，那么pid指定的任一子进程暂停后继续，则返回其状态。
* WNOHANG：子进程并不是立即可用的，那么不阻塞，返回0。
* WUNTRACED：若实现支持作业控制，并且pid指定的进程已经暂停但是还没有报告过，那么会返回其状态。

# waitid

```c++
#include <sys/wait.h>
int waitid(idtype_t idtype, id_t id, siginfo_t *infop, int options);
```

类似于waitpid，但是比那个更灵活，它允许一个进程指定要等待的子进程，使用一个单独的参数指定要等待的进程的类型，idtype：

|  常量  |                      说明                      |
|:------:|:----------------------------------------------:|
|  P_PID |   等待一个特定进程，id包含要等待的子进程的ID   |
| P_PGID | 等待一个特定进程组的任一子进程，id包含进程组ID |
|  P_ALL |                 等待任一子进程                 |

# wait3和wait4

这两个函数提供的功能要比waitid还要多一个，是利用一个附加参数来实现的：

```c++
#include <sys/types.h>
#include <sys/wait.h>
#include <sys/time.h>
#include <sys/resource.h>
 
pid_t wait3(int *statloc, int options, struct rusage *rusage);
pid_t wait4(pid_t pid, int *statloc, int optionjs, struct rusage *rusage);
```

具体应用书上没写，待补充。

# 竞争条件

如果一个进程要等待其父进程终止，可以采用如下的方法：

```c++
while(getppid() != 1)
        sleep(1);
```

# exec

```c++
#include <unistd.h>
int execl(const char *pathname, const char *arg0, ... /* (char *)0 */ );
int execv(const char * pathname, char *const argv[]);
int execle(const char *pathname, const char *arg0, .../* (char* 0, char *const envp[]) */);
int execve(const char *pathname, char *const argv[], char *const envp[]);
int execlp(const char *filename, const char *arg0, .../* (char *)0 */);
int execvp(const char *filename, char *const argv[]);
```

前四个取路径名作为参数，后两个取文件名作为参数，使用文件名的时候如果包含/符号，那么视为路径名，如果没有就在PATH中寻找。

l代表参数使用列表，v代表参数使用数组，e代表传输环境表，p代表使用filename，用这四个字母代表的含义来区分这四个函数。

# 更改用户ID和组ID

这一个通常用于程序中希望能够动态改变权限的时候，使用两个函数来更改：

```c++
#include <unistd.h>
int setuid(uid_t uid);
int setgid(gid_t gid);
```

更改的基本规则：

* 若进程拥有超级用户特权，setuid会将实际用户ID、有效用户ID以及保存的设置用户ID更改成期望的ID。
* 若进程没有超级用户特权，但是uid等于实际用户ID或保存的设置用户ID，则setuid可以将有效用户ID设置为uid，而不改变另两个。
* 如果两个条件都不满足，出错返回。

还有两个函数只更改有效用户ID和有效组ID：

```c++
#include <unistd.h>
int seteuid(uid_t uid);
int setegid(gid_t gid);
```

# 解释器文件

所有的UNIX系统都支持解释器文件，解释器文件的开始形式是：

```bash
#!pathname [optional-argument
```

也就是这个文件可以直接运行，虽然他是个文本文件，但是他会用第一行的pathname指定的程序来执行后面的文本程序。

# system

```c++
#include <stdlib.h>
int system(const char *cmdstring);
```

用于执行命令字符串，但是对操作系统依赖性强。

# 进程会计

```c++
typedef u_int16_t comp_t
 
struct acct
{
        char ac_flag;         /* Flags.  */
        u_int16_t ac_uid;     /* Real user ID.  */
        u_int16_t ac_gid;     /* Real group ID.  */
        u_int16_t ac_tty;     /* Controlling terminal.  */
        u_int32_t ac_btime;       /* Beginning time.  */
        comp_t ac_utime;      /* User time.  */
        comp_t ac_stime;      /* System time.  */
        comp_t ac_etime;      /* Elapsed time.  */
        comp_t ac_mem;        /* Average memory usage.  */
        comp_t ac_io;         /* Chars transferred.  */
        comp_t ac_rw;         /* Blocks read or written.  */
        comp_t ac_minflt;     /* Minor pagefaults.  */
        comp_t ac_majflt;     /* Major pagefaults.  */
        comp_t ac_swaps;      /* Number of swaps.  */
        u_int32_t ac_exitcode;    /* Process exitcode.  */
        char ac_comm[ACCT_COMM+1];    /* Command name.  */
        char ac_pad[10];      /* Padding bytes.  */
};
```

# 用户标识

getuid获取的是用户ID，获取登录名可以使用下面的函数：

```c++
#include <unistd.h>
char *getlogin(void);
```

# 进程时间

任一进程都可以调用times函数来获取墙上时钟时间、用户CPU时间和系统CPU时间：

```c++
#include <sys/time.h>
clock_t times(struct tms *buf);
```

tms的结构为：

```c++
struct tms
{
        clock_t tms_utime;      /* User CPU time.  */
        clock_t tms_stime;      /* System CPU time.  */

        clock_t tms_cutime;     /* User CPU time of dead children.  */
        clock_t tms_cstime;     /* System CPU time of dead children.  */
};

```