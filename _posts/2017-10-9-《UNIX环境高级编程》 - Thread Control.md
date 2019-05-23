---
layout: post
title: 《UNIX环境高级编程》 - Thread Control
author: 宋强
tags: UNIX
date: 2017-10-09 21:13 +0800
---

# 线程属性

如果想要在线程创建的时候给予其属性，那么将使用pthread_attr_t型变量，初始化和销毁的函数如下：

```c++
#include <pthread.h>
int pthread_attr_init(pthread_attr_t *attr);
int pthread_attr_destroy(pthread_attr_t *attr);
```

结构体内主要需要的四种属性是：

* detachstate：线程的分离状态属性。
* guardsize：线程末尾的警戒缓冲区大小。
* stackaddr：线程栈的最低地址。
* stacksize：线程栈的大小。

如果在创建线程时就不需要了解线程的终止状态，可以在attr中设置detach，不过要通过两个函数对detach属性进行查询和设置：