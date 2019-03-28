---
layout: post
title: "Nardflash芯片分析"
subtitle: Nardflash芯片结构分析
author: "404"
header-style: text
tags:
  - Linux
  - Arm
  - Uboot
  - NARDflash
---

　　在写前面一篇NARDflash在Uboot启动过程中代码拷贝时，遇到一些硬件设置的时候，感觉很模糊，不知道代码这么设置的原因，不知道拷贝到底如何拷贝，现在就来写这篇文章来进行NARDflash芯片内部体系分析。

# 1. Flash介绍

　　NARDFlash在嵌入式系统中的地位与PC上硬盘类似，用于保存存放系统运行所必须的操作系统、应用程序、用户数据、运行过程中产生的各类数据。与内存掉电后数据丢失不同，NARDflash中的数据在掉电后可永久保存。

　　常用的Flash类型有NORFlash和NARDflash两种。NORflash由Intel公司在1988年发明，以代替当时在市场上占据主导地位的EPROM和$E^2PROM$。Nardflash由Toshiba公司在1989年发明。两者的主要差别如下：

　|NOR|Nard
容量　|1MB~3MB|16MB~512MB
XIP | YES | NO
擦写　|非常慢（5s）|快（3ms）
写　|慢|快
读　|快|快
可靠性 　|比较高，位反转比例小于NARDflash的10%|比较低，位反转比较常见，必须有校验措施，比如TNR必须有坏块管理措施
可擦除次数  |10000~100000|100000~1000000
生命周期 　|低于NARDflash的10%|是NORflash的10倍以上
接口|与RAM接口相同|I/O接口
访问方法|随机访问|顺序访问
易用性|容易|复杂
主要用途|常用于保存代码和关键数据|用于保存数据
价格|高|低

　　NORflash支持XIP,即代码可以直接在NORflash上执行，无需复制到内存中。这是由于NORflash的接口与RAM完全相同，可随机访问任意地址的数据。在NORflash上进行读操作的效率非常高，但擦除和写的效率很低；另外，NORflash的容量一般比较小。NARDflash进行擦除和写的效率更高，并且容量更大。一般而言，NORflash用于存储程序，NARDflash用于存储数据。基于NARDflash的设备通常也要搭配NORflash以存储程序。


　





　　
