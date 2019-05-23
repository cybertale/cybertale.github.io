---
layout: post
title: 《UNIX环境高级编程》 - Thread
author: 宋强
tags: UNIX
date: 2017-10-08 23:10 +0800
---

# 线程ID

进程ID从定义来说是一个非负整数，所以可以直接比较，但是线程ID在不同操作系统中是不同的，所以使用了结构体来表示，需要用pthread_equal来进行比较：

```c++
#include <pthread.h>
int pthread_equal(pthread_t tid1, pthread_t tid2);
```

想要获取线程的ID的话需要使用pthread_self函数：

```c++
#include <pthread.h>
pthread_t pthread_self(void);
```

# 线程的创建

线程所需要的一些信息包括线程ID、一组寄存器、栈、调度优先级和策略、信号屏蔽字、errno变量以及线程私有数据。

创建线程使用pthread_create：

```c++
#include <pthread.h>
int pthread_create(pthread_t *restrict tidp, const pthread_attr_t *restrict attr, void *(*start_rtn)(void), void *restrict arg);
```

执行成功时，tidp指向的内存单元被存储新线程的ID，attr参数用于定制线程属性。

线程的终止

线程不可用通过exit终止，这样整个进程都会退出，也绝对不可以通过给线程发送终止信号终止，终止一个线程的方式有三种：

* 线程从启动例程返回，返回退出码。
* 线程可以被同一进程的其他线程取消。
* 线程调用pthread_exit。

```c++
#include <pthread.h>
void pthread_exit(void *rval_ptr);
```

这个函数会结束线程，并且将要返回的信息存储到rval_ptr中，然后其他线程可以通过pthread_join访问这个变量：

```c++
#include <pthread.h>
int pthread_join(pthread_t thread, void **rval_ptr);
```

这个函数会阻塞等待目标线程终止，之后获取rval_ptr的值，但是如果对返回结果不感兴趣，可以直接使用NULL。

线程可以通过调用pthread_cancel函数来请求取消同一进程中的其他线程：

```c++
#include <pthread.h>
int pthread_cancel(pthread_t tid);
```

这个函数只是发出请求，被请求停止的进程相当于调用了pthread_exit(PTHREAD_CANCEL);

线程可以安排退出时需要调用的清理函数，类似于atexit：

```c++
#include <pthread.h>
void pthread_cleanup_push(void (*rtn)(void *), void *arg);
void pthread_cleanup_pop(int execute);
```

rtn的调用顺序是由push来安排的，当前程调用以下动作时将会执行清理函数：

* 调用pthread_exit。
* 响应取消请求。
* 是用非零参数调用pthread_cleanup_pop。

如果操作系统对于某个线程的终止状态不感兴趣，可以使用detach，这样的话线程结束之后操作系统会自动收回它所占的资源。

如果想要分离一个线程，可以使用pthread_detach:

```c++
#include <pthread.h>
int pthread_detach(pthread_t tid);
```

对比一下线程和进程的原语：

| 进程原语 |       线程原语      |             描述             |
|:--------:|:-------------------:|:----------------------------:|
|   fork   |    pthread_create   |          创建控制流          |
|   exit   |     pthread_exit    |        从控制流中推出        |
|  waitpid |     pthread_join    |    从控制流中得到退出状态    |
|  atexit  | pthread_cancel_push | 注册在退出控制流时调用的函数 |
|  getpid  |     pthread_self    |        获取控制流的ID        |
|   abort  |    pthread_cancel   |    请求控制流的非正常退出    |

# 线程同步

当一个线程修改变量，另一个线程读取变量时，修改动作大部分情况下是非原子的，所以需要让其变成原子的，这样就能保证两个进程看见的变量永远是一个，而没有过旧。

## 互斥量（mutex）

互斥量是一个线程的数字，当有线程访问的时候他被减少到零，另一个线程想访问由于0不可以再被减少而阻塞，直到之前的线程访问变量完成之后释放互斥量，第二个访问进程停止阻塞，继续执行。

互斥量的初始化和销毁函数：

```c++
#include <pthread.h>
int pthread_mutex_init(pthread_mutex_t *restrict mutex, const pthread_mutexattr_t *restrict attr);
int pthread_mutex_destroy(pthread_mutex_t *mutex);
```

加锁解锁函数：

```c++
#include <pthread.h>
int pthread_mutex_lock(pthread_mutex_t *mutex);
int pthread_mutex_trylock(pthread_mutex_t *mutex);
int pthread_mutex_unlock(pthread_mutex_t *mutex);
```

第一个是阻塞等待，第二个是非阻塞等待，第三个解锁。

注意，互斥量是一种工具，如果有线程访问数据而不使用lock，这种机制将不能保证互斥，所以要求按照mutex的使用规范来访问变量。

## 死锁

如果一个线程对某个互斥量加锁两次，那么将进入死锁状态，或者是两个线程都锁定了一个互斥量，还想要互相访问对方锁住的变量时也会导致死锁，可以使用trylock来避免死锁。

## 读写锁

读写锁是一种更高级的互斥量，互斥量是限制访问，同一个时间只能有一个线程访问，但是读写锁分为写锁和读锁，写锁只能同一个进程拥有，并且拥有时别的进程不可访问资源，但是读锁可以多个进程拥有。

读写锁的初始化和销毁函数：

```c++
#include <pthread.h>
int pthread_rwlock_init(pthread_rwlock_t *restrict rwlock, const pthread_rwlockattr_t *restrict attr);
int pthread_rwlock_destroy(pthread_rwlock_t *rwlock);
```

加锁和解锁：

```c++
#include <pthread.h>
int pthread_rwlock_rdlock(pthread_rwlock_t *rwlock);
int pthread_rwlock_wrlock(pthread_rwlock_t *rwlock);
int pthread_rwlock_runock(pthread_rwlock_t *rwlock);
```

非阻塞：

```c++
#include <pthread.h>
int pthread_rwlock_tryrdlock(pthread_rwlock_t *rwlock);
int pthread_rwlock_trywrlock(pthread_rwlock_t *rwlock);
```

## 条件变量

初始化和销毁：

```c++
#include <pthread.h>
int pthread_cond_init(pthread_cond_t *restric cond, pthread_condattr_t *restrict attr);
int pthread_cond_destroy(pthread_cond_t *cond);
```

使用pthread_cond_wait等待条件变为真，如果在给定的时间内条件得不到满足，那么回生成一个代表出错码的返回值：

```c++
#include <pthread.h>
int pthread_cond_wait(pthread_cond_t *restrict cond, pthread_mutex_t *restrict mutex);
int pthread_cond_timewait(pthread_cond_t *restrict cond, pthread_mutex_t *restrict mutex, const struct timespec *restrict timeout);
```

...

```c++
struct timespec{
        time_t tv_sec;
        long tv_nsec;
};
```

有两个函数通知线程条件已经满足，pthread_cond_signal函数将唤醒等待该条件的某个进程，而pthread_cond_broadcast函数将还行等待该条件的所有进程：

```c++
#include <pthread.h>
int pthread_cond_signal(pthread_cond_t *cond);
int pthread_cond_broadcast(pthread_cond_t *cond);
```

**条件变量到底是什么？？**