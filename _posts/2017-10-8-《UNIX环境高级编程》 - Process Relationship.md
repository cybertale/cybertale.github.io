---
layout: post
title: 《UNIX环境高级编程》 - Process Relationship
author: 宋强
tags: UNIX
date: 2017-10-08 17:02 +0800
---

# 进程组

进程组通常是共同任务的进程的集合，组长进程的进程ID和进程组ID相同。

获取进程组ID使用getpgrp：

```c++
#include <unistd.h>
pid_t getpgrp(void);
```

也可以获取其他进程进程组ID：

```c++
#include <unistd.h>
pid_t getpgid(pid_t pid);
```

进程组内只要有进程存在进程组就不会消亡，可以通过setpgid来创建或者加入一个进程组：

```c++
#include <unistd.h>
int setpgid(pid_t pid, pid_t pgid);
```

将pid进程设置为pgid进程组进程，如果pid为0，将使用调用进程，如果pgid是0，则pid指定的进程id将作为进程组ID。

进程可以为自己和子进程设置进程组ID，但当子进程调用exec之后，就不能再设置了。

# 会话（session）

会话是一个或者多个进程组的集合，通常是shell的管道编排在一起的。

建立新会话是通过调用setsid来完成的：

```c++
#include <unistd.h>
pid_t setsid(void);
```

建立一个新会话时，如果这个进程不是一个进程组的组长：

* 这个进程变为会话leader。
* 自动创建一个他为进程组组长的进程组。
* 如果有控制终端将与之切断。
* 如果已经是一个进程组组长，将返回出错。

getsid返回会话ID，其实也就是创建这个会话的进程的ID：

```c++
#include <unistd.h>
pid_t getsid(pid_t pid);
```

出于安全考虑，很多操作系统规定只有属于这个会话的进程才可以调用这个函数取得会话ID。

# 控制终端

* 一个会话可以有一个控制终端，这个终端通常是终端设备或者伪终端设备。
* 建立与控制终端连接的会话首进程将被称为控制进程。
* 一个会话中的几个进程组可以被分为一个前台进程组和几个后台进程组。
* 如果一个会话有一个控制终端，那么一定有一个前台进程组，其他都是后台进程组。
* 无论何时键入控制终端的Ctrl+C和DELETE和Ctrl+\都将被发送给前台进程组的所有进程。
* 如果终端检测到网络断开连接，将会把挂断信号发送给会话首进程。

使用下面的函数来获取其中哪个进程组是前台进程组：

```c++
#include <unistd.h>
pid_t tcgetpgrp(int filedes);
int tcsetpgrp(int filedes, pid_t, pid_t pgrpid);
```

第一个输入的参数是打开的关联终端。

也可以直接获得会话首进程的进程组ID：

```c++
#include <termios.h>
pid_t tcgetsid(int filedes);
```

# 作业控制

这个通常是shell下使用的，一个进程组被执行时被分配一个作业号，我们可以通过作业控制控制哪些进程组在后台执行，哪些进程组可以访问终端。

**不太清楚是否真的需要作业控制，等待补充。**

# shell执行程序

# 孤儿进程组