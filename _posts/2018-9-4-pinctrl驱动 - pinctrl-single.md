---
layout: post
title: pinctrl驱动 - pinctrl-single
author: 宋强
tags: Linux pinctrl
date: 2018-09-04 10:35 +0800
---

# pinctrl-single, pins=<>

可以使用寄存器地址和值对的方式来配置pinmux，这个时候使用pinctrl-single, pins来完成配置，例如：

```c++
pinctrl-single, pins=<0xdc 0x118>
```

但是如果一个寄存器中包含多个配置信息而我们配置一个引脚的时候不想影响其他人，可以添加一个mask，放在最后，例如：

```c++
pinctrl-single, pins=<0xdc 0x18 0xff>
```