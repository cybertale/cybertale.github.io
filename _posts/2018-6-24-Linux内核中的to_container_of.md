---
layout: post
title: Linux内核中的to_container_of
author: 宋强
tags: Linux内核
---

Linux的内核中使用了非常多的面向对象编程的思想，数据结构相对来说还是比较容易理解的，就拿时钟驱动来说，Linux提供的对某一个时钟的抽象是clk结构体，这个算作基类，但是对于时钟我们可能添加自己的属性，所以规定我们可以定义一个形如clk_foo的结构体：

```c++
struct clk_foo {
    struct clk_hw hw;
    ... hardware specific data goes here ...
};
```
第一个属性必须是clk_hw结构体。

这里要注意的是，clk_foo结构体基本都不相同，接收这个clk_foo结构体作为参数却是需要一样的参数，这里虽然可以使用void *指针，但是这样的话就必须根据传进来的不同结构进行强制转换，不具有通用性，而linux在实现这个多态输入的方面的做法非常巧妙，是使用clk_hw结构体的指针作为参数，to_container_of使用gcc链接记录的clk_foo和clk_hw的偏移地址计算出clk_foo的地址然后返回，这样只需要在我们实现的函数中使用to_container_of就可以将基类转换成子类指针，就获得了子类结构体了。

clk_foo结构体的地址和clk_hw的地址的偏移量只和他们的定义形式有关（***这里不应该就是0吗？？为了照顾clk_hw没有放在第一位的情况吗？？***）

是一种多态的实现方法。