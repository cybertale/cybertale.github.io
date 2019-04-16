---
layout: post
title: Linux驱动分析 - pinctrl
author: 宋强
tags: Linux pinctrl
date: 2018-07-06 16:04 +0800
---

3.2版本开始加入pinctrl系统。

为引脚服用提供驱动框架，并为其他需要使用引脚复用的提供API。

大多数引脚复用配置采用设备树的方式进行传递。

pinctrl在设备树中也分为两部分，一部分是pincotroller，还有一部分是分布在各个设备节点中的引脚配置。

设备中的引脚配置使用两类属性：
* pinctrl-names：用于指定有几种状态，会被对应枚举0-n。这里面default这个名字比较特别，这个状态存在的话设备会被默认初始化为这个状态。
* pinctrl-\<id\>：names里面有几种状态，就有几个id，指定这个状态对应的控制器中的节点中的配置信息。

控制器信息形如：

```c++
&am33xx_pinmux {
        ...
        i2c2_pins: pinmux_i2c2_pins {
                pinctrl-single,pins = <
                                AM33XX_IOPAD(0x978, PIN_INPUT_PULLUP | MUX_MODE3)
                                /* (D18) uart1_ctsn.I2C2_SDA */
                                AM33XX_IOPAD(0x97c, PIN_INPUT_PULLUP | MUX_MODE3)
                                /* (D17) uart1_rtsn.I2C2_SCL */
                >;
        };
};
```

我们要使用的就是i2c2_pins这个节点配置。

i2c2设备中使用这个配置的设置如下：

```c++
&i2c2 {
        pinctrl-names = "default";
        pinctrl-0 = <&i2c2_pins>;
 
        status = "okay";
        clock-frequency = <400000>;
        ...
 
        pressure@76 {
                compatible = "bosch,bmp280";
                reg = <0x76>;
        };
};
```

# 通用驱动pinctrl-single

这个驱动有些类似led-gpio还有i2c-gpio，是一个linux内核提供的通用驱动，适用于通过给一个寄存器赋值来配置引脚复用的情况，AM335x就是采用的这一种驱动。

# 设备树需要的属性

前面几个到pinctrl-single,function-mask都是必须的，后面的是可选的。

### compatible

两种值：
* pinctrl-single：不支持pinconf功能。
* pinconf-single：支持pinconf功能。

### reg

寄存器的地址和长度。

### \#pinctrl-cells

除了index位以外需要赋值的位的个数，如果是1的话代表 pinctrl-single,pins 2的话代表 pinctrl-single,bits（***这个有点不确定***）。

### pinctrl-single, register-mask

复用寄存器的宽度，例如32.

### pinctrl-single, function-mask

使能复用使用的掩码，例如0x7f（***不确定***）。

## 可选的

### pinctrl-single, function-off

和function-mask相对，上面的是使能，这个是失能。
