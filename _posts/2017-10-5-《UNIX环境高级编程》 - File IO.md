---
layout: post
title: 《UNIX环境高级编程》 - File IO
author: 宋强
tags: UNIX
date: 2017-10-05 22:05 +0800
---

# open

```c++
#include <fcntl.h>
int open(const char *pathname, int oflag, ...);
```

## oflag
下面这些参数都定义在<fcntl.h>头文件中。

下面三个参数有且只有一个：

| oflag |      |
|----------|------|
| O_RDONLY | 只读 |
| O_WRONLY | 只写 |
| O_RDWR   | 读写 |

下面的选项是可以选择的：

| oflag   |                                                                                |
|------------|--------------------------------------------------------------------------------------------------------|
| O_APPEND   | 每次写文件都追加到尾端。                                                                               |
| O_CREAT    | 如果此文件不存在，则创建，但是必须制定第三个参数mode，mode指定了访问权限。                             |
| O_EXCL     | 如果指定了 O_CREAT，并且文件已经存在，则会出错，如果没有则创建，用于将查询是否存在和创建两个动作分开。 |
| O_TRUNC    | 如果此文件存在，并且以包含写的方式打开，则将其长度截为0。                                              |
| O_NOCTTY   | 如果打开的是一个终端设备，则不将此终端作为此进程的控制终端。                                           |
| O_NONBLOCK | 如果打开的是一个FIFO、快特殊文件或者是字符特殊文件，则以非阻塞模式打开。                               

POSIX和Single Unix Specification还增加了三个可选参数：

| oflag | |
|---------|-------------------------------------------------------------------------------------------------|
| O_DSYNC | 每次write操作等待物理磁盘操作完成，但是如果写操作不影响读取刚写入的数据，则不等待文件属性更新。 |
| O_RSYNC | 使每一个以文件描述符作为参数的read操作等待，直至任何对文件同一部分进行的未决写操作都完成。      |
| O_SYNC  | 使每次write都等到物理IO完成，包括文件属性更新。                                                 |

# create

```c++
#include <fcntl.h>
int create(const char *pathname, mode_t mode);
```

等价于

```c++
#include <fcntl.h>
open(pathname, O_WRONLY | O_CRATE | O_TRUNC, mode);
```

早期的操作系统无法使用open创建文件，现在的操作系统create用的不多了。

# close

```c++
#include <fcntl.h>
int close(int filedes);
```

当进程终止时，内核会自动关闭所有他打开的文件。

# lseek

```c++
#include <fcntl.h>
off_t lseek(int fileds, off_t offset, int whence);
```

成功返回当前偏移量，失败返回-1。

offset的含义取决于whence（定义在<unistd.h>）：

| whence |     |
|----------|----------------------------------------|
| SEEK_SET | 文件偏移量为从文件开始offset个字节     |
| SEEK_CUR | 文件偏移量为当前值加上offset，可以为负 |
| SEEK_END | 文件偏移量为文件长度加上offset         |

可以使用lseek来判断所涉及的文件是否为管道、FIFO、套接字，因为这些东西不可以设置偏移量，所以使用lseek的时候会返回错误。

例如：

```c++
#include <fcntl.h>
lseek(fd, 0, SEEK_CUR);
```

实质上文件偏移量是一个变量，一直存储着，并没有随关闭而清零。使用lseek只是设置这个变量，也没有IO操作。

# read

```c++
#include <fcntl.h>
ssize_t read(int filedes, void *buf, size_t nbytes);
```

返回读取到的字节数。（void *用于表示通用指针，C89）

# write

```c++
#include <fcntl.h>
ssize_t write(int fileds, const void *buf, size_t nbytes);
```

写操作

# 原子操作

为了避免共享文件表而出现的操作中文件偏移量被其他人修改的问题，可以采用原子操作将write/read和lseek合并在一起，Single UNIX Specification提供了两个函数来完成这样的功能：

```c++
#incldue <unistd.h>
 
ssize_t pread(int filedes, void *buf, size_t nbytes, off_t offset);
ssize_t pwrite(int filedes, const void *buf, size_t nbytes, off_t offset);
```

在原有的基础上机上一个offset，指明偏移量。

现在的很多时候使用在打开的时候直接使用O_APPEND标志，然后每次写都会原子的写到文件尾。

# dup

```c++
#include <fcntl.h>
int dup(int filedes);
```

复制一个文件描述符，文件描述符实质是一个结构体，之后两个文件描述符指向同一个文件，共享所有标志和文件偏移量，返回一个最小的文件描述符标号。

# dup2

```c++
#include <fcntl.h>
int dup2(int filedes, int filedes2);
```

和dup不同的是这个可以通过第二个选项指定新的文件描述符标号，如果已经存在，那么会关闭那个文件之后把这个描述符分配给返回值，如果指定的值和filedes的值相同，返回他，不做任何操作。

# sync

```c++
#include <fcntl.h>
void sync(void);
```

将所有修改过的快缓冲区加入写队列，然后返回，不等待磁盘IO。

# fsync

```c++
#include <fcntl.h>
int fdatasync(int filedes);
```

只对filedes指定的文件有效，但是会一直等待到磁盘IO结束。

# fdatasync

```c++
#include <fcntl.h>
int fdatasync(int filedes);
```

类似于fsync，但是只是对文件的数据部分有效，同时还会更新文件的属性。

# fcntl

```c++
#include <fcntl.h>
int fcntl(int filedes, int cmd, .../* int arg */);
```

fcntl的功能由cmd的值决定（定义于<fcntl.h>）：

| cmd                    |    |
|----------------------------|-------------------------|
| F_DUPFD                    | 复制一个现有的描述符    |
| F_GETFD, F_SETFD           | 获得/设置文件描述符标记 |
| F_GETFL, F_SETFL           | 获得/设置文件状态标记   |
| F_GETOWN, F_SETOWN         | 获得/设置异步IO所有权   |
| F_GETLK, F_SETLK, F_SETLKW | 获得/设置记录锁         |

# ioctl

```c++
#include <fcntl.h>
int ioctl(int filedes, int request, ...);
```

ioctl是一个各种剩余函数的杂物箱，扩展设备驱动的时候会需要。

# /dev/fd目录

这个目录下一般都有0、1、2三个文件夹，打开这些文件夹相当于使用dup复制描述符表。
（补充）