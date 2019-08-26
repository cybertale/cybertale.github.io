---
layout: post
title: GSoC 2019 Linux IIO subsystem AD5940 Driver Development
author: Song Qiang
tags:
date: 2019-08-26 15:19 +0800
---

# What I have done

* Write the driver for the AD5940 ADC under ADI Linux Kernel.

# Code that got merged

* dts, doc and basic code skeleton of the driver. [Github: Add driver skeleton support for AD5940](https://github.com/analogdevicesinc/linux/pull/427)
* Differential channel configuring system of the driver. [Github: Add differential ADC channel configuration of AD5940](https://github.com/analogdevicesinc/linux/pull/438/commits/0b1fb3affd74bed6b6143b3b46f8d46ad4c158ec)
* Regulator support of the driver. [Github: Add regulator support for AD5940](https://github.com/analogdevicesinc/linux/pull/438/commits/ffd2ef4676efb53405d0a4825b879554b8d82b6d)

# Code currently under review

* Real SPI read write code and interrupt support. [Github: Add SPI and irq support of AD5940](https://github.com/analogdevicesinc/linux/pull/515/commits/ab94a2b42b9109cbf6ecec5629fb99847281df33)

# Future work

* Add trigger, buffer and triggered buffer support of the driver.
* Send it to the mainline kernel as patches.
* Add DAC support of the device.
* Add all filter results read entrances for all channels.
* Power management of the system.
* Waveform generator driver of the device.
* mfd driver of the device.