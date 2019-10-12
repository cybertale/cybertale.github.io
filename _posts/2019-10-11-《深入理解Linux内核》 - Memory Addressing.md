---
layout: post
title: 《深入理解Linux内核》 - Memory Addressing
author: 宋强
tags: linux
date: 2019-10-11 9:08 +0800
---

The Linux GDT

Linux系统中每一个CPU对应一个GDT，全部存储在`cpu_gdt_table`这个数组中，所有GDT的地址和大小存储在`cpu_gdt_descr`这个数组中。


