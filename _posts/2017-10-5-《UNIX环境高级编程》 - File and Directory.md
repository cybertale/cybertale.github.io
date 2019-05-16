---
layout: post
title: 《UNIX环境高级编程》 - File and Directory
author: 宋强
tags: UNIX
date: 2017-10-05 22:59 +0800
---

# stat, fstat和lstat函数

```c++
int stat(const char * restrict pathname, struct stat *restrict buf);
int fstat(int filedes, struct stat *buf);
int lstat(const char *restrict pathname, struct stat *restrict buf);
```

有关定义都定义在<sys/stat.h>头文件中。

stat函数用于返回文件相关的信息结构，fstat用于使用描述符货的文件，lstat和stat大部分功能相同，但是当要操作i的文件类型是一个符号链接时，他返回符号链接的信息，而不是符号链接所指向的文件的，也就是stat会返回符号链接所指向的最终文件的信息。

stat结构体的基础形式为：

```c++
struct stat{
    mode_t      st_mode;
    ino_t       st_ino;
    dev_t       st_dev;
    dev_t       st_rdev;
    nlink_t     st_nlink;
    uid_t       st_uid;
    gid_t       st_gid;
    off_t       st_size;
    time_t      st_atime;
    time_t      st_mtime;
    time_t      st_ctime;
    blksize_t   st_blksize;
    blkcnt_t    st_blocks;
};
```

UNIX总共包括七种文件类型：
* 普通文件：就是普通文件。
* 目录文件：就是目录文件。
* 块特殊文件：这种文件类型用于对块设备提供访问。
* 字符特殊文件：这种文件类型用于对字符型设备提供访问。
* FIFO：这种类型文件用于进程间通讯，有时也称为命名管道。
* 套接字：这种文件类型通常用于网络通讯。
* 符号链接：就是快捷方式。

在stat中使用以下几个宏判断文件类型：

|     宏     |   文件类型   |
|:----------:|:------------:|
|  S_ISREG() |   普通文件   |
|  S_ISDIR() |     目录     |
|  S_ISCHR() | 字符特殊文件 |
|  S_ISBLK() |  块特殊文件  |
| S_ISFIFO() |   FIFO文件   |
|  S_ISLNK() |   符号链接   |
| S_ISSOCK() |    套接字    |

# 访问权限

关于用户权限的说明  p75

## access

```c++
int access(const char *pathname, int mode);
```

按照实际用户ID和组用户ID来对文件进行读写权限测试：

mode有以下四种属性：

| mode |       属性       |
|:----:|:----------------:|
| R_OK |    测试读权限    |
| W_OK |    测试写权限    |
| X_OK |   测试执行权限   |
| F_OK | 测试文件是否存在 |

## umask

```c++
mode_t umask(mode_t cmask);
```

进程创建文件时，使用umask设置好的文件访问权限。

cmask是stat中定义的9个值：

|  cmask  |          |
|:-------:|:--------:|
| S_IRUSR |  用户读  |
| S_IWUSR |  用户写  |
| S_IXUSR | 用户执行 |
| S_IRGRP |   组读   |
| S_IWGRP |   组写   |
| S_IXGRP |  组执行  |
| S_IROTH |  其他读  |
| S_IWOTH |  其他写  |
| S_IXOTH | 其他执行 |

## chmod,fchmod

```c++
int chmod(const char *pathname, mode_t mode);
int fchmode(int filedes, mode_t mode);
```

这两个函数用于修改文件的访问权限，。

mode有15种选择：

|     mode     |          说明          |
|:------------:|:----------------------:|
|    S_ISUID   |    执行时设置用户ID    |
|    S_ISGID   |     执行时设置组ID     |
|    S_ISVTX   |   保存正文（粘住位）   |
| S_IRWXU      | 用户（所有者）读写执行 |
|      S_IRUSR |           读           |
|      S_IWUSR |           写           |
|      S_IXUSR |          执行          |
| S_IRWXG      |       组读写执行       |
|      S_IRGRP |           读           |
|      S_IWGRP |           写           |
|      S_IXGRP |          执行          |
| S_IRWXO      |    其他用户读写执行    |
|      S_IROTH |           读           |
|      S_IWOTH |           写           |
|      S_IXOTH |          执行          |

有关粘住位（sticky bit）的说明，详见p83。

## chown,fchown,lchown

```c++
#include <unistd.h>
int chown(const char *pathname, uid_t owner, gid_t group);
int fchown(int filedes, uid_t owner, gid_t group);
int lchown(const char *pathname, uid_t owner, gid_t group);
```

如果owner或者group的值设置为-1，则不变。

超级用户或者拥有此文件或者当前进程拥有修改此文件权限的时候才可以修改。

## truncate,ftruncate

```c++
#include <unistd.h>
int truncate(const char *pathname, off_t length);
int ftruncate(int filedes, off_t length);
```

这两个函数将文件缩短为length个字节，length之后的将不可以再访问，可以用于截短某个文件。

# 硬链接

## link,unlink,remove,rename

```c++
#include <unistd.h>
int link(const char *existingpath, const char *newpath);
int unlink(const char *pathname);
```

link与unlink创建与删除链接。（本质是目录？？这一节需要多对文件系统有所了解，p89）

这一节的链接指的是硬链接。

```c++
#include <stdio.h>
int remove(const char *pathname);
int rename(const char *oldname, const char *newname);
```

remove同样也可以用于解除一个对文件或者目录的链接。

rename用于对一个文件或者目录重命名。

# 符号链接

在使用路径文件名作为参数的一些文件操作函数里面，有些不支持符号链接，也就是使用符号链接作为参数时，操作的是符号链接本身，而不是其指向的文件，各个函数对符号链接的处理如下：

| 是否穿透到目标文件 |   函数   |
|:------------------:|:--------:|
|         是         |  access  |
|         是         |   chdir  |
|         是         |   chmod  |
|         否         |   chown  |
|         是         |   creat  |
|         是         |   exec   |
|         否         |  lchown  |
|         是         |   link   |
|         否         |   lstat  |
|         是         |   open   |
|         是         |  opendir |
|         是         | pathconf |
|         否         | readlink |
|         否         |  remove  |
|         否         |  rename  |
|         是         |   stat   |
|         是         | truncate |
|         否         |  unlink  |

有一个关于符号链接的经典错误，如果创建了不能打开的文件的符号链接，那么打开这个文件的时候就会提示不存在这个文件，让人感到迷惑：

```bash
$ln -s /no/such/file myfile
$cat myfile
>>cat: myfile: No such file or directory
```

## symlink,readlink

```c++
#include <unistd.h>
int symlink(const char *actualpath, const char *sympath);
ssize_t readlink(const char* restrict pathname, char * restrict buf, size_t bufsize);
```

symlink用于创建一个符号链接，实际文件不要求一定存在。
readlink用于打开符号链接本身，并且返回链接中指向的实际文件的路径文件名字符串。

# 文件的时间

有关于各种函数修改i节点和文件内容实体是否修改相应的三种文件时间。p94

## utime

```c++
#include <utime.h>
int utime(const char *pathname, const struct utimebuf *times);
```

用于修改文件的时间属性。

```c++
struct utimebuf{
    time_t  actime;
    time_t  modtime;
};
```

## mkdir,rmdir

```c++
include <sys/stat.h>
int mkdir(const char * pathname, mode_t mode);
```

创建一个空目录，创建目录时必须至少给出一个执行权限，否则将无法进入。

```c++
#include <unistd.h>
int rmdir(const char * pathname);
```

满足删除条件时（链接数为0，打开他的进程的数目也为0等），执行此命令将删除这个目录。

## chdir,fchdir,getpwd

```c++
#include <unistd.h>
int chdir(const char *pathname);
int fchdir(int filedes);
char *getcwd(char *buf, size_t size);
```

前两个用于切换目录，最后一个获取当前工作目录的绝对路径字符串，size是指定buf长度用的。