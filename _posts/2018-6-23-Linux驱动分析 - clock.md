---
layout: post
title: Linux驱动分析 - clock
author: 宋强
tags: Linux clock
date: 2018-06-23 22:20 +0800
---

现在的clk框架使用的是叫做common clock framework的框架，简称CCF。

![](../../../images/Linux&#32;clock/clock.jpg)

内核中CCF的使用，浅蓝色的部分是需要驱动实现的。

现代时钟管理主要是三种功能：时钟开关、速率调整和复用，CONFIG_COMMON_CLK选项可以使能这个时钟框架的使用。

框架主要分为两层：
* 第一层是对于clk结构体，是最高层的抽象。
* 第二层是对于clk.h中api的通用实现方法，调用的是用户定义的clk_ops结构体中的方法。

这里有一个命名约定，通常使用clk_foo作为foo时钟的clk_ops结构体。

第一部分中的clk结构体主要包含一个clk_core结构体指针和几个信息，最重要的就是clk_core结构体。

两个部分之间的连接是使用一个clk_hw结构体完成的，这个结构体定义在clk_foo当中，clk_ops又有指向clk_hw的一个指针，这样获取了clk_hw的驱动代码可以使用to_container_of获取整个我们自己定义的clk_foo结构体。

drivers/clk/clk.c中有一些用于使用clk_core结构体操作时钟的api，这些api的说明都在include/linux/clk.h当中，而且实际上是使用clk_core中的clk_ops指针指向的结构体的方法来操作时钟的，clk_ops结构体的声明在include/linux/clk-provider.h当中。

# CCF提供的时钟类型
* fixed-rate：永远运行并且提供固定频率的时钟。
* gate：拥有和父节点同样的频率只能够控制打开和关断。
* mux：能够从几个父时钟中选择一个作为输出。
* fixed-factor：可以在父时钟的基础上乘上或者处以固定常数之后输出。
* divider：可以以指定的分频系数对父时钟进行分频。
* clk-composite：新的正在讨论的时钟类型，可以任意组合基础时钟类型成为一个新的时钟类型。

实际的时钟由CCF提供的基础时钟类型组合而成，可能到来的clk-composite类型也满足这个理念。

# 硬件时钟操作实现

## 添加对自定义时钟的支持

首先要创建clk_foo结构体：

```c++
struct clk_foo {
    struct clk_hw hw;
    ... hardware specific data goes here ...
};
```

包含基类clk_hw结构体和一些我们自定义的属性。

然后要创建一个clk_ops结构体的实例提供具体操作：

```c++
struct clk_ops clk_foo_ops {
    .enable     = &clk_foo_enable;
    .disable    = &clk_foo_disable;
};
```

在实现上面这几个函数的时候可以使用to_clk_foo(_hw)宏，用于将clk_hw指针转换成clk_foo指针，例如：

```c++
int clk_foo_enable(struct clk_hw *hw)
{
    struct clk_foo *foo;
 
    foo = to_clk_foo(hw);
 
    ... perform magic on foo ...
 
    return 0;
};
```

内核文档还给出了哪些ops在我们具有哪些功能的情况下是必须实现的：

|                 | gate | change rate | single parent | multiplexer | root |
|-----------------|------|-------------|---------------|-------------|------|
| prepare         |      |             |               |             |      |
| unprepare       |      |             |               |             |      |
| enable          | y    |             |               |             |      |
| disable         | y    |             |               |             |      |
| is_enabled      | y    |             |               |             |      |
| recalc_rate     |      | y           |               |             |      |
| round_rate      |      | y\[1\]      |               |             |      |
| determine_rate  |      | y\[1\]      |               |             |      |
| set_rate        |      | y           |               |             |      |
| set_parent      |      |             | n             | y           | n    |
| get_parent      |      |             | n             | y           | n    |
| recalc_accuracy |      |             |               |             |      |
| init            |      |             |               |             |      |

标\[1\]的代表round_rate和recalc_rate至少需要实现一个。
最后一步就是使用clk_register函数注册clk驱动了。

# 时钟驱动框架中的锁

整个通用时钟框架中包含两个锁，prepare锁和enable锁。

给enable用的锁是一个自旋锁，在enable的三种操作期间全部都要获取，防止他们睡眠和被中断（Linux中规定自旋锁不可被中断，见LDD同步与互斥一节）。

prepare锁在其他的所有环节中使用，是一个互斥锁，可能进入睡眠，所以不可以再任何原子程序中使用。

由于使用的锁的不同通用始终框架中的操作被分成两部分，在任何一个部分中的驱动对资源共享的保护都是这个锁完成的，***但是如果有一个资源被两个拥有不同锁的ops竞争，那么驱动需要自己添加互斥方法保证互斥性***。

通用时钟框架代码是可重入的，这样的话很有可能在执行一个操作的时候执行另一个时钟的同样操作，驱动代码也尽量考虑可重入。

驱动代码外操作时钟也需要考虑使用锁，保证资源不会失去保护。

# 设备驱动中使用时钟驱动

具体的设备驱动中使用时钟驱动需要使用时钟驱动提供的API，包括：

```c++
struct clk *clk_get(struct device *dev, const char *id);
struct clk *devm_clk_get(struct device *dev, const char *id);
int clk_enable(struct clk *clk);
int clk_prepare(struct clk *clk);
void clk_unprepare(struct clk *clk);
void clk_disable(struct clk *clk);
static inline int clk_prepare_enable(struct clk *clk);
static inline void clk_disable_unprepare(struct clk *clk);
unsigned long clk_get_rate(struct clk *clk);
int clk_set_rate(struct clk *clk, unsigned long rate);
struct clk *struct clk_get_parent(struct clk *clk);
int struct clk_set_parent(struct clk *clk, struct clk *parent);
```

prepare的操作是后来加入的，由于某些硬件时钟的使能和禁止要求在原子上下文中进行，所以就把使能的过程分为了可以进入睡眠的prepare和原子过程的enable两部分。

# 参考

1. [Documentation/clk.txt](https://www.kernel.org/doc/Documentation/clk.txt)
2. [Common Clock Framework: How to use it - Bootlin](https://bootlin.com/pub/conferences/2013/elce/common-clock-framework-how-to-use-it/)
3. Linux设备驱动开发详解：基于最新的Linux4.0内核，宋宝华，572-577