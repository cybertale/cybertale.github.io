---
layout: post
title: 《UNIX环境高级编程》 - System File and Information
author: 宋强
tags: UNIX
date: 2017-10-07 23:42 +0800
---

# 口令文件

获取口令文件的函数：

```c++
#include <pwd.h>
struct passwd *getpwuid(uid_t uid);
struct passwd *getpwnam(const char *name);
```

这两个用于查询特定用户的口令文件，也可以使用下面的函数来查询所有用户的口令文件：

```c++
#include <pwd.h>
struct passwd * getpwent(void);
void setpwent(void);
void endpwent(void);
```

setpwent用于定位当前读取的口令文件到最开始，getpwent用于获取下一个口令文件，但是由于getpwent只会打开口令文件，所以最后一定要调用endpwent来关闭所有打开的口令文件。

# 阴影口令

为了减少口令被猜测到的概率，又增加了一个阴影口令文件，其中存储了用户名和加密口令和一些控制口令衰老的信息。

加密口令存储在这个文件里面的话/etc/password文件就可以由任何人读取，通常阴影口令文件很少用，但是只有root用户可以访问，访问的函数为：

```c++
#include <shadow.h>
struct spwd *getspnam(const char *name);
struct spwd *getspent(void);
void setspent(void);
void endspent(void);
```

具体含义待补充。

# 组文件

/etc/group文件中存放了有关组的信息，可以通过下面的两个函数通过组名和组ID获取组信息：

```c++
#include <grp.h>
struct group *getgrgid(gid_t gid);
struct group *getgrnam(const char *name);
```

查看所有组信息采用几个类似的函数：

```c++
#include <grp.h>
struct group *getgernt(void);
void setgrent(void);
void endgrent(void);
```

# 附加组

附加组用于给予用户多个组的身份，操作附加组的几个函数为：

```c++
#include <unistd.h>
int getgroups(int gidsetsize, gid_t grouplist[]);
 
#include <gro.h>
#include <unistd.h>
int setgroups(int ngroups, const gid_t grouplist[]);
int initgroups(constchar *username, gid_t basegid);
```

# 其他系统文件

/etc下的系统文件实现大致相同，操作通常都是三个函数：

* get函数，获取下一个实例。
* set函数，回到开始。
* end函数，关闭所有打开的文件。

/etc下经典的几个如下：

| 说明 |    数据文件    |   头文件   |   结构   |
|:----:|:--------------:|:----------:|:--------:|
| 口令 |   /etc/passwd  |   \<pwd.h\>  |  passwd  |
|  组  |   /etc/group   |   \<grp.h\>  |   group  |
| 阴影 |   /etc/shadow  | \<shadow.h\> |   spwd   |
| 主机 |   /etc/hosts   |  \<netdb.h\> |  hostent |
| 网络 |  /etc/networks |  \<netdb.h\> |  netent  |
| 协议 | /etc/protocols |  \<netdb.h\> | protoent |
| 服务 |  /etc/services |  \<netdb.h\> |  servent |

# 登入账户记录

UNIX中，utmp文件记录了所有当前登陆进的账户，登录和注销的事件被记录在wtmp文件中，记录进去的是同一种数据结构：

```c++
struct utmp{
        char ut_line[8];
        char ut_name[8];
        long ut_time;
}
```

line代表登录的控制台号，time是间隔时间。

who程序读取的就是utmp文件。

# 系统标识

uname函数用于返回操作系统相关的一些信息：

```c++
#include <sys/utsname.h>
int uname(struct utsname *name);
```

utsname的基本结构为：

```c++
struct utsname{
       char sysname[];
       char nodename[];
       char release[];
       char version[];
       char machine[];
}
```

# 日期和时间

UNIX中使用time函数返回当前时间和日期：

```c++
#include <time.h>
time_t time(time_t *calptr);
```

时间作为函数值返回，如果参数不为空，也将时间存放在calptr中。

还有一个gettimeofday可以获取精细达微秒级的时间：

```c++
#include <sys/time.h>
int gettimeofday(struct timeval *restrict tp, void *restrict tzp);
```

timeval的结构为：

```c++
struct timeval{
        time_t tv_sec;
        long tv_usec;
}
```

tzp的唯一合法值为NULL。

上面获取的时间都是1970年1月1日00：00：00时开始的秒数，需要转换。

使用localtime和gmttime两个函数可以将这个时间编程年月日时分秒周几的形式，一个转换为本地时间，一个转换为标准时间：

```c++
#include <time.h>
struct tm *gmttime(const time_t *calptr);
struct tm *localtime(const time_t *calptr);
```

转换完成的结构体为：

```c++
struct tm{
        int tm_sec;             //秒[0-60]
        int tm_min;             //第几分钟[0-59]
        int tm_hour;            //午夜之后的第几个小时[0-23]
        int tm_mday;          //这个月的第几天[1-31]
        int tm_mon;            //一月开始的第几个月[0-11]
        int tm_year;            //1900开始的第几年
        int tm_wday;           //周天开始的一周的第多少天
        int tm_yday;            //一年的第多少天
        int tm_isdst;            //如果>0就是夏日时，=0不是，<0无效。
};
```

函数mktime将本地时间转换成time_t结构体：

```c++
#include <time.h>
time_t mktime(struct tm *tmptr);
```

asctime和ctime产生时间日期字符串，前者产生日期的，后者产生时间的，形式类似于：

```bash	
Tue Fed 10 18:27:38 2004\n\0
```

```c++
#include <time.h>
char *asctime(const struct tm *tmptr);
char *ctime(const time_t *calptr);
```

最后要介绍的时strftime函数，这个函数用于格式化输出时间信息到字符串：

```c++
#include <time.h>
size_t strftime(char *restrict buf, size_t maxsize, const char *restrict format, const struct tm *restrict tmptr);
```

如果buf足够长能够存下输出的字符串和一个终止符null，则返回存放字符的个数，否则返回0，这个里面的format形式比较容易理解：

| 格式 |                   说明                   |           实例           |
|:----:|:----------------------------------------:|:------------------------:|
|  %a  |               所写的周日名               |            Tue           |
|  %A  |                 全周日名                 |          Tuesday         |
|  %b  |                缩写的月名                |            Feb           |
|  %B  |                  全月名                  |         February         |
|  %c  |                日期和时间                | Tue Feb 10 18:27:38 2004 |
|  %C  |             年/100，取值0-99             |            20            |
|  %d  |             一月第几日，01-31            |            10            |
|  %D  |              日期\[MM/DD/YY\]              |         02/10/04         |
|  %e  | 一月第几日，一位数之前加上空格，取值1-31 |            10            |
|  %F  |       ISO 8601日期格式\[YYYY-MM-DD\]       |        2004-02-10        |
|  %g  |   ISO 8601基于周的年的最后两位数\[00-99\]  |            04            |
|  %G  |            ISO 8601基于周的年            |           2004           |
|  %h  |                 与%b相同                 |            Feb           |
|  %H  |         小时（24小时制），\[00-23\]        |            18            |
|  %I  |         小时（12小时制），\[01-12\]        |            06            |
|  %j  |              年日，\[001-366\]             |            041           |
|  %m  |                 月\[01-12\]                |            02            |
|  %M  |                 分\[00-59\]                |            27            |
|  %n  |                  换行符                  |                          |
|  %p  |                   AM/PM                  |            PM            |
|  %r  |            本地时间，12小时制            |         06:27:38         |
|  %R  |               与“%H:%M”相同              |           18:27          |
|  %S  |                秒，\[00-60\]               |            38            |
|  %t  |                水平制表符                |                          |
|  %T  |             与“%H:%M:%S”相同             |         18:27:38         |
|  %u  |            ISO 8601 周日\[1-7\]            |             2            |
|  %U  |            星期日周数，\[00-53\]           |            06            |
|  %V  |           ISO 8601周数，\[01-53\]          |            07            |
|  %w  |               周日，\[00-06\]              |            02            |
|  %W  |            星期一周数，\[00-53\]           |            06            |
|  %x  |                   日期                   |         02/10/04         |
|  %X  |                   时间                   |         18:27:38         |
|  %y  |          年的最后两位数，\[00-99\]         |            04            |
|  %Y  |                    年                    |           2004           |
|  %z  |          ISO 8601格式的UTC偏移量         |           -0500          |
|  %Z  |                  时区名                  |            EST           |
|  %%  |                转换为1个%                |             %            |