---
layout: post
title: 利用Mentor LP Wizard 10.5生成Allegro 17.2封装库
author: 宋强
tags: PCB
date: 2017-03-23 15:33 +0800
---

在使用Cadence最新的版本17.2的时候，发现LP Wizard生成的Allegro脚本无法使用，生成的一些LQFP的元器件没有焊盘，倒是0603的电容点阻生成的很正常，一直很奇怪，后来发现是17.2的版本相对于16.6的版本pad editor有了很大的更新，更新成为了新的软件，叫做padstack，LP Wizard的脚本无法成功生成焊盘，所以调用的时候产生的元件都没有焊盘，0603的电阻电容生成正常，是因为我之前手动制作过他们的焊盘，命名也是一样的，所以他直接就可以调用。

这里提供17.2版本下使用LP Wizard的办法，以LQFP64封装的STM32F407为例。

打开手册，找到基本参数，包括Pitch(e)，长(D)宽(E)高(A)，引脚数。

![LQFP-100](../../../images/LP&#32;Wizard/LQFP-100.jpg)

得到信息Pitch 0.5mm，D 12mm， E 12mm， A 1.6mm， 引脚数64，IPC7351标准命名为QFP50P1200X1200X160-64，找到相应的标准封装。

![LP-Selection](../../../images/LP&#32;Wizard/LP-Sellection.jpg)

打开之后再打开Wizard，CAD软件选择Allegro，勾选掉1处，在2处随便选择一个目录，这个一会还要删，再点击Create and Close。

![LP-Properties](../../../images/LP&#32;Wizard/LP-Properties.jpg)

打开上面2中文件夹里生成的文件夹。

![LP-Generated-Files](../../../images/LP&#32;Wizard/LP-Generated-Files.jpg)

查看psr文件，代表pad script，看看这两个焊盘你有没有做过，我的话做过了所以不用再做了，所以也推荐大家做的时候把标准名称的焊盘收集起来，如果没有的话就打开psr文件，就算不会allegro script肯定也能看懂，都有注释，如果没有的话就按照注释使用padstack制作相应的焊盘。

这个时候里面可能有ssr文件，是shape script，需要调用一下生成扩展名为ssm的shape script文件，有可能制作特殊形状的焊盘需要，例如QFN封装的焊盘。

最后用记事本打开allegroload文件，删除所有pad和shape生成的相关命令行，只执行最后一个allegro命令，就可以完成啦。

最后生成的封装并不是完全适合我们使用，推荐大家和我一样操作，他总共生成了两个REF，一个在Assambly Top，一个在Silkscreen Top，推荐大家删除Assambly Top层的，留下一个并且放在芯片旁边作为Ref，还生成了一个DEV在Silkscreen Top，这个实际制作PCB的时候包含的是Package的信息，文字非常多，把他挪到Assambly Top作为焊接和解释封装的注释。

再删除两个选中的圆形的焊盘，这个我们一般情况下也是不需要的。

做完这些工作之后把生成文件夹的dra文件和psm文件拷贝到你的PCB元件库中，这个文件夹剩下的东西就都可以删除啦。

![LP-Result](../../../images/LP&#32;Wizard/LP-Result.jpg)