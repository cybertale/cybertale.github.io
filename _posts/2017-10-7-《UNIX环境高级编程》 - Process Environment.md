---
layout: post
title: 《UNIX环境高级编程》 - Process Environment
author: 宋强
tags: UNIX
date: 2017-10-07 17:01 +0800
---

# 进程的启动和终止

Unix的C程序由main函数进入，五种正常终止方式：

* 从main返回。
* 调用exit。
* 调用_exit或者_Exit。
* 最后一个线程从其他例程中返回。
* 最后一个线程调用pthread_exit。

三种异常终止方式：

* 调用abort。
* 接收到一个信号并且终止。
* 最后一个线程对取消请求做出响应。

## 三个exit函数

```c++
#include <stdlib.h>
void exit(int status);
void _Exit(int status);
 
#include <unistd.h>
void _exit(int status);
```

这三个里面exit总是会执行一个标准IO流的清理工作，但是另外两个会直接陷入内核结束进程。

main函数返回整形值和使用exit函数退出给一个整形值在退出码方面时等价的，如果return不指明返回数值，那么将是一个随机数。

## atextit函数

根据ISO C规定，进程可以登记多达32个函数，在使用exit退出或者直接使用return退出的时候会自动调用他们，使用atexit来登记这些函数：

```c++
#include <stdlib.h>
int atexit(void (*func)(void));
```

登记函数的顺序与执行顺序相反，并且支持多次登记相同的函数。

# 循环命令行参数

由于ISO和POSIX都规定argv[argc]必须为NULL，所以如果想要循环所有参数的话，可以直接使用循环：

```c++
for(i = 0; argv[i] != NULL; i++)
```

# 环境表

历史上main函数还包括第三个参数，是一个指针数组，指向了操作系统传给他的环境变量表，而现在默认去掉了这个参数，并使用

```c++
extern char **environ;
```

这个变量直接指向环境表，环境表中的字符串遵循name=value的键值对形式，通常name都是大写的。

# 存储器分配

ISO C中指定了三个用于存储空间动态分配的函数：

* malloc：分配指定字节数的存储区，初始值不确定。
* calloc：为指定数量拥有指定长度的对象分配存储空间，这个空间中的每一位都被初始化为0.
* realloc：更改以前分配区的长度，增加的存储区部分的初始值将不确定。

```c++
#include <stdlib.h>
void *malloc(size_t size);
void *calloc(size_t nobj, size_t size);
void realloc(void *ptr, size_t newsize);
void free(void *ptr);
```

realloc中的newsize是新区长度，而不是差值。

通常需要和sbrk系统调用协作，这个系统调用可以改变程序堆的大小。

泄露（leak）：是指一个程序一直malloc而并没有free空间，会使分页空间过多，程序效率降低。

也绝不可free已经free过的块，这也会产生灾难性的错误。

## 一些替代的存储器分配库

由于存储器分配容易产生错误，人们就写了一些可以自己检查的库，比如libmalloc, vmallocm quick-fit, alloca。

可以调研其中一个选择使用。

# 环境变量

访问环境变量的时候，ISO C提供了一个函数getenv，并不推荐直接访问environ变量：

```c++
#include <stdlib.h>
char *getenv(const char *name);
```

这个函数可以直接获取到和name相关联的value。

Single UNIX Specification定义了一些环境变量：

|     变量    |                   说明                  |
|:-----------:|:---------------------------------------:|
|   COLUMNS   |                 终端宽度                |
|   DATEMSK   |          getdate模板文件路径名          |
|     HOME    |                 起始目录                |
|     LANG    |                  本地名                 |
|    LC_ALL   |                  本地名                 |
|  LC_COLLATE |                本地排序名               |
|   LC_CTYPE  |              本地字符分类名             |
| LC_MESSAGES |                本地消息名               |
| LC_MONETARY |              本地货币编辑名             |
|  LC_NUMERIC |              本地数字编辑名             |
|   LC_TIME   |           本地日期/时间格式名           |
|    LINES    |                 终端高度                |
|   LOGNAME   |                  登录名                 |
|   MSGVERB   | fmtmsg处理的消息组成部分（linux未支持） |
|   NLSPATH   |              消息类模板序列             |
|     PATH    |       搜索可执行文件的路径前缀列表      |
|     PWD     |         当前工作目录的绝对路径名        |
|    SHELL    |            用户首选的shell名            |
|     TERM    |                 终端类型                |
|    TMPDIR   |      在其中创建临时文件的目录路径名     |
|      TZ     |                 时区信息                |

可以使用一些函数来修改当前进程所使用的环境变量表，包括putenv,setenv,unsetenv和clearenv。

clearenv是linux自己提供的，用于清除所有的环境变量，前三个的函数声明为：

```c++
#include <stdlib.h>
int putenv(char *str);
int setenv(const char *name, const char *value, int rewirite);
int unsetenv(const char *name);
```

putenv输入参数的形式是一个name=value的字符串，如果已经存在，那么回自动删除原来的。

setenv中若rewrite为非0才会删除原有定义，否则什么也不发生。

unsetenv删除定义，即使不存在也不会出错。

# setjmp和longjmp

C语言中的goto是不可以跳转到函数外的，这样的情况需要用这两个jmp函数来实现。

```c++
#include <setjmp.h>
int setjmp(jmp_buf env);
void longjmp(jmp_buf env, int val);
```

使用方式是先定义一个全局jmp_buf变量，在需要返回到的地方调用setjmp函数，之后在出错的地方调用longjmp，longjmp的第二个值是返回值，可以用来表达是从哪里返回的。

**使用这两个函数的时候会存在自动变量跳回的时候没有回滚到原来的值的情况，这一点要注意。**

# getrlimit和setrlimit

```c++
#include <sys/resource.h>
int getrlimit(int resource, struct rlimit *rlptr);
int setrlimit(int resource, const struct rlimit *rlptr);
```

这两个函数用于管理进程的资源，rlimit的结构为：

```c++
struct rlimit
  {
    /* The current (soft) limit.  */
    rlim_t rlim_cur;
    /* The hard limit.  */
    rlim_t rlim_max;
  };
```

更改资源限制的时候，有三个规则：

* 任何一个进程都可以修改其软限制值到小于等于其硬限制值。
* 任何一个进程都可以降低硬限制值，但是必须大于等于软限制值，这种降低对普通用户来说是不可逆的。
* 只有超级用户可以提高硬限制值。


***不太懂他要说明的资源是什么？？$$P_{165}$$***