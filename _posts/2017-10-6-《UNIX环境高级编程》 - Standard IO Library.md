---
layout: post
title: 《UNIX环境高级编程》 - Standard IO Library
author: 宋强
tags: UNIX
date: 2017-10-06 22:19 +0800
---

# 流和FILE对象

## fwide

```c++
#include <stdio.h>
#include <wchar.h>
int fwide(FILE *fp, int mode);
```

fwide用于设置一个未定向的流的宽度，不能用于已定向的流。

| mode |                                        |
|:----:|:--------------------------------------:|
|  正  |                宽定向的                |
|  负  |               字节定向的               |
|  零  | 不设置流的定向，但是返回标识流定向的值 |

# 标准输入，输出和出错

这三个的预定义文件描述符宏为STDIN_FILENO、STDOUT_FILENO和STDERR_FILENO，预定义的流文件指针为stdin、stdout和stderr。

# 缓冲

缓冲有三种：

* 全缓冲：所有标准库输出的IO等到缓冲区满了之后进行实际IO操作。
* 行缓冲：遇到换行符或者缓冲区满的时候进行IO操作。
* 不缓冲：不进行缓冲，立刻进行IO操作。

通常不涉及交互的设备不需要进行行缓冲或者不缓冲，类似于命令行终端的设备使用行缓冲，保证交互性，标准错误输出使用不缓冲，这样能够使错误信息尽可能快的提示出来。

## 更改缓冲类型

```c++
#include <stdio.h>
void setbuf(FILE *restrict fp, char *restrict buf);
int setvbuf(FILE *restrict fp, char *restrict buf, int mode, size_t size);
```

setbuf用于打开关闭缓冲，当给buf一个缓冲区的时候，采用系统决定的缓冲类型，当给buf一个NULL指针的时候，缓冲被关闭。

setbuf和setvbuf的实际效果如下：

|   函数  |    mode    |  buf |        缓冲区及长度        |     缓冲类型     |
|:-------:|:----------:|:----:|:--------------------------:|:----------------:|
|  setbuf |  宽定向的  | 非空 | 长度为BUFSIZE的用户指定buf | 全缓冲或者行缓冲 |
|         | 字节定向的 | NULL |          无缓冲区          |      无缓冲      |
| setvbuf |   _IOFBF   | 非空 |   长度为size的用户指定buf  |      全缓冲      |
|         |            | NULL |    合适长度的系统缓冲区    |                  |
|         |   _IOLBF   | 非空 |   长度为size的用户指定buf  |      行缓冲      |
|         |            | NULL |    合适长度的系统缓冲区    |                  |
|         |   _IONBF   | 忽略 |          无缓冲区          |      无缓冲      |

# 强制清空流

流可以被强制冲洗，使用fflush函数：

```c++
#inlcude <stdio.h>
int fflush(FILE *fp);
```

将缓冲区内的所有数据进行IO操作，如果fp为NULL，那么将冲洗所有缓冲区。

# 打开和关闭流

有三个函数可以打开标准IO流：

```c++
#include <stdio.h>
FILE *fopen(const char * restrict pathname, const char *restrict type);
FILE *freopen(const char *restrict pathname, const char *restrict type, FILE *restrict fp);
FILE *fdopen(int filedes, const char *type);
```

区别：

* fopen：打开指定文件。
* freopen：在一个指定的流上打开指定文件，通常会将流关闭和重定向。
* fdopen：有些实际设备，例如socket，不能够使用fopen进行打开，必须要用dup等函数复制一个文件描述符之后，使用文件描述符来关联文件和流。

type有以下几种：

* r、rb：读打开
* w、wb：把文件截断至0大小之后为写而打开，或者为写而创建。
* a、ab：在文件尾进行添加，或者为写而创建。
* r+、r+b、rb+：读写打开。
* w+、w+b、wb+：截断至0，读写打开。
* a+、a+b、ab+：在文件尾读写，打开或者创建。

字符b是用于区分文件是文本文件还是标准二进制文件，不过在UNIX中不区分这两种数据，所以b在UNIX中没有作用。

## 使用fclose关闭一个流

```c++
#include <stdio.h>
int fclose(FILE *fp);
```

# 对打开的流进行读写

使用标准库对流读写有三种方式：

* 每次进行一个字符的IO：对于带缓冲的流，标准IO库将自动管理缓冲实际IO，对应的函数也就是fgetc()和fputc()。
* 每次进行一行的IO：对应的函数也就是fgets()和fputs()。
* 直接控制IO：标准库支持用户直接使用fwrite()和fread()进行IO。

## 字符形式读

```c++
#include <stdio.h>
int getc(FILE *fp);
int fgetc(FILE *fp);
int getchar(void);
```

getchar()相当于fgetc(stdin)，第一个getc是预定义宏，剩下两个是函数，返回值由unsigned char被强制转换成了int。

这三个函数也有些问题，就在于出错或者到达文件尾端，他们都将返回-1，但是可以使用下面的函数来获取一个流的错误状态：

```c++
#incldue <stdio.h>
int ferror(FILE *fp);
int feof(FILE *fp);
void clearerr(FILE *fp);
```

这样个函数如果返回零的话代表相应的事件发生。

每个流为错误留下了两个标志，一个出错标志和一个到达流末尾的标志，使用最后一个clearerr函数则可以清除这两个标志。

## 将字符送入到流中

流里面读取出字符之后可以使用ungetc函数将字符压回到流，类似于入栈操作，读取出来的时候也类似于栈使用的FILO形式。

```c++
#include <stdio.h>
int ungetc(int c, FILE *fp);
```

通常有的时候要根据一个流前几个字符是什么来决定要如何处理后面的，比如说帧头指定的后面压缩帧的长度，这样的话这个函数就会比较有用。

## 字符形式写

同样使用类似的三个函数进行写操作：

```c++
#include <stdio.h>
int putc(int c, FILE *fp);
int fputc(int c, FILE *fp);
int putchar(int c);
```

## 行读

两个函数可以提供读行的操作:

```c++
#include <stdio.h>
char *fgets(char *restrict buf, int n, FILE *restrict fp);
char *gets(char *buf);
```

gets相当于从标准输入读取，而且相比fgets，他并不讲换行符存入到缓冲区中，这个函数并不被推荐使用，因为他没有指定缓冲区，所以容易造成缓冲溢出。

## 行写

类似的，也有两个函数用于写行的操作：

```c++
#include <stdio.h>
int fputs(const char *restrict str, FILE *restrict fp);
int puts(const char *str);
```

fputs一直写到null，但并不输出null，并不保证一定有换行符，这个要自己处理，puts同样读取到null，但是他多输出一个换行符。

# 标准库的二进制IO

对于特殊结构的读写，我们希望能够自定制一次都写的字节数以及数据结构，这个时候就会希望直接使用write或者read，标准库提供了这样一个接口进行读写：

```c++
#include <stdio.h>
size_t fread(void *restrict ptr, size_t size, size_t nobj, FILE *restrict fp);
size_t fwrite(const void *restrict ptr, size_t size, size_t nobj, FILE *restrict fp);
```

size为单个元素长度，nobj为个数。

两种常用用法：

* 读写一个二进制数组。
* 读写一个结构体。

# 定位标准流

有三种方式定位标准流：

* ftell和fseek，传统，但是他们假定文件位置可以存放在一个长整形中。
* ftello和fseeko，使用off_t数据类型代替长整形。
* fgetpos和fsetpos，这两个函数使用一个抽象的fpos_t数据类型来记录文件位置，由于这个函数是IOS C引入的，所以这一种对跨平台的支持最好。

## ftell和fseek

```c++
#include <stdio.h>
long ftell(FILE *fp);
int fseek(FILE *fp, long offset, int whence);
void rewind(FILE *fp);
```

这个就和普通文件定位中的lseek一样的。

## ftello和fseeko

```c++
#include <stdio.h>
off_t ftello(FILE *fp);
int fseeko(FILE *fp, off_t offset, int whence);
```

特殊之处就是offset比32位更长了。

## fgetpos和fsetpos

```c++
#include <stdio.h>
int fgetpos(FILE *restrict fp, fpos_t *restrict pos);
int fsetpos(FILE *fp, const fpos_t *pos);
```

只是将存储位置的形式从一个变量变成了一个结构体。

# 格式化IO

## 输出

格式化输出使用的是四种printf函数：

```c++
#include <stdio.h>
int printf(const char *restrict format, ...);
int fprintf(FILE *restrict fp, const char *restrict format, ...);
int sprintf(char *restrict buf, const char *restrict format, ...);
int snprintf(char *restrict buf, size_t n, const char *restrict format, ...);
```

sprintf是一个我们常用的函数，他相比printf会在最后自动插入一个null，但是这个函数非常容易导致缓冲区溢出，所以推荐使用snprintf函数，这个函数显式指定了缓冲区的大小。

## format的格式

%\[flags\]\[fldwidth\]\[precision\]\[lenmodifier\]convtype

### flag

| 标志 |                         说明                         |
|:----:|:----------------------------------------------------:|
|   -  |                   字段内左对齐输出                   |
|   +  |               总是显示带符号转换的符号               |
| 空格 |    如果第一个字符不是括号，则在其前面加上一个空格    |
|   #  | 制定另一种转换形式，（例如对16进制，采用0x前缀形式） |
|   0  |                   添加前导0进行填充                  |

### fldwidth

说明转换的最小字段宽度，如果得到的少，就用空格填充，这个宽度是一个非负10进制数，或者是一个*号。

### precision

说明最少输出数字位数、浮点数转换后小数点后的最少位数、字符串转换后的最大位数。

### lenmodifier

说明参数长度，可取值如下：

| 长度修饰符 |         说明        |
|:----------:|:-------------------:|
|     hh     |     有无符号char    |
|      h     |    有无符号short    |
|      l     |     有无符号long    |
|     ll     |     有无符号long    |
|      j     | intmax_t或uintmax_t |
|      z     |        size_t       |
|      t     |      ptrdiff_t      |
|      L     |     long double     |

### convtype

| 转换类型 |                            说明                            |
|:--------:|:----------------------------------------------------------:|
|    d,i   |                        有符号十进制                        |
|     o    |                        无符号八进制                        |
|     u    |                        无符号十进制                        |
|    x,X   |                       无符号十六进制                       |
|    f,F   |                           double                           |
|    e,E   |                           double                           |
|    g,G   |            根据被转换的值，自动解释为f、F、e或E            |
|    a,A   |                  十六进制指数格式的double                  |
|     c    |                     字符（带l为宽字符）                    |
|     s    |                    字符串（带l为宽字符）                   |
|     p    |                       指向void的指针                       |
|     n    | 将到目前为止，所写的字符数写入到指针所指向的无符号整形数中 |
|     %    |                            %符号                           |
|     C    |                      宽字符，等效于lc                      |
|     S    |                      宽字符，等效于ls                      |

标准输出的printf还有四个变体，可变参数表...替换成了arg参数：

```c++
#include <stdarg.h>
#include <stdio.h>
int vprintf(const char *restrict format, va_list arg);
int vfprintf(FILE *restrict fp, const char *restrict format, va_list arg);
int vsprintf(char *restrict buf, const char *restrict format, va_list arg);
int vsnprintf(char *restrict buf, size_t n, const char *restrict format, va_list arg);
```

## 输入

```c++
#include <stdio.h>
int scanf(const char *restrict format, ...);
int fscanf(FILE *restrict fp, const char *restrict format, ...);
int sscanf(const char *restrict buf, const char *restrict format, ...);
```

### format

%\[*\]\[fldwidth\]\[lenmodifier\]convtype

### 前导*

前导*用于抑制转换，按照转换说明的其余部分进行转换，但是转换的结果并不存放在参数中。

### fldwidth

说明最大宽度。

### lenmodifier

说明用转换结果初始化的参数的大小。

### convtype

|        转换类型        |                            说明                            |
|:----------------------:|:----------------------------------------------------------:|
|            d           |            有符号十进制，基数为10（基数？？？）            |
|            i           |              有符号十进制，基数由输入格式决定              |
|            o           |                        无符号八进制                        |
|            u           |                   无符号十进制，基数为10                   |
|            x           |                       无符号十六进制                       |
| a、A、e、E、f、F、g、G |                           浮点数                           |
|            c           |                            字符                            |
|            s           |                           字符串                           |
|            [           |                 匹配列出的字符序列，用]终止                |
|           [^           |             匹配列出字符以外的所有字符，用]终止            |
|            p           |                       指向void的指针                       |
|            n           | 将到目前为止，所写的字符数写入到指针所指向的无符号整形数中 |
|            %           |                            %符号                           |
|            C           |                      宽字符，等效于lc                      |
|            S           |                      宽字符，等效于ls                      |

和printf一样，scanf也支持\<stdarg.h\>支持的可变参数表：

```c++
#include<stdarg.h>
#include <stdio.h>
int vscanf(const char *restrict format, va_list arg);
int vfscanf(FILE *restrict fp, const char *restrict format, va_list arg);
int vsscanf(const char *restrict buf, const char *restrict format, va_list arg);
```

# 一些支持

可以用fileno函数来获取一个文件流的文件描述符：

```c++
#include <stdio.h>
int fileno(FILE *fp);
```

# 临时文件

标准IO库提供了两个函数用于创建临时文件：

```c++
#include <stdio.h>
char *tmpnam(char *ptr);
FILE *tmpfile(void);
```

第一个tmpnam用于产生一个与现有文件名都不同的路径文件名字符串，调用的最多次数由TMP_MAX限定。

tmpfile产生一个临时文件，在程序结束的时候被自动删除。

还有两个扩展函数：

```c++
#include <stdio.h>
char *tempnam(const char *directory, const char *prefix);
int mkstemp(char *template);
```

第一个相比之下增加了可以指定存放目录和文件名前缀，第二个相比之下返回的是文件描述符，template放置的是一个路径名，后六个字符设置为XXXXXX，这样的话mkstemp会自动更改后六位，返回文件描述符，也获得了新的文件的名字。