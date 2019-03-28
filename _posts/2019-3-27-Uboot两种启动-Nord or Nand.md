---
layout: post
title: "Uboot两种启动-Nord or Nand"
subtitle: Uboot的两种启动过程
author: "404"
header-style: text
tags:
  - Linux
  - Arm
  - Uboot
  - SDRAM
---

# 1. Uboot启动介绍

　　我们从前面的文章知道了，Uboot常见的启动方式有两种：Nordflash启动和Nardflash启动。其实我们就将其当成两种不同介质启动就行，那从这两种介质启动有什么不同呢？主要是从两种介质中，拷贝执行程序到SDRAM中不同。
- Nordflash有专用的地址引脚来寻址，较容易和其他芯片联接，还支持本地执行
- Nandflash的IO端口采用复用的数据线和地址线，必须先通过寄存器串行地进行数据存取
　　由于Nordflash可直接与SDRAM进行数据交互，Nandflash不行，由此引发了不同的拷贝方式。

# 2. Uboot启动阶段

　　以前我应该写过Uboot启动过程中两个阶段的过程，现在我们再回顾下：
- stage 1：初始化了SoC内部的一些部件（譬如看门狗、时钟），然后初始化DDR并且完成重定位
- stage 2：初始化剩下的还没被初始化的硬件。主要是SoC外部硬件（譬如iNand、网卡芯片...）、uboot本身的一些东西（uboot的命令、环境变量等）。然后最终初始化完必要的东西后进入uboot的命令行准备接受命令
　　
　　这两个阶段都对应相应的程序，我们现在称为BL1、BL2，BL1对应着stage1阶段，BL2对应着stage2阶段。再Noarflah中，我们可以直接将这Uboot程序包含内核映像拷贝到SDRAM中，然而在Nandflash中，由于不能将这些直接从Nardflash拷贝到SDRAM，所以我们中间要借助一些过程。

# 3. 从Nardflash如何启动
　　前面我们知道了不能直接拷贝，那么怎么办呢？
1. 在板子上电的一开始,首先自动判断是否是autoboot模式(这是由硬件设计阶段,由硬件工程师对mcu的引脚连线决定的),我所使用的s3c2410是带有nandflash的,并切被设置成autoboot,从nandflash开始启动
2. 在判断是autoboot模式后,mcu内置的nandflash控制器自动将nandflash的最前面的4k区域(这4k区域存放着bootloader的最前面4k代码)拷贝到samsung所谓的"steppingstone"里面(steppingstone是在S3C2440中，实际上是一块4k大小的SRAM)　
3. 在拷贝完前4k代码后,nandflash控制器自动将"steppingstone"映射到arm地址空间0x00000000开始的前4k区域
4. 在映射过程完成后.nandflash控制器将pc指针直接指向arm地址空间的0x00000000位置,准备开始执行"steppingstone"上的代码
5. 而"steppingstone"上从nandflash拷贝过来的4k代码,是程序员写的bootloader的前4k代码.这个bootloader在之前写好,并已经被烧写到nandflash的0x00000000开始的最前面区域。而这"steppingstone"上的4k代码就是bootloader的前4k代码
6. 在pc指向arm地址空间的0x00000000后,系统就开始执行指令代码.这4k代码的任务是:初始化硬件,设置中断向量表,设置堆栈,然后一个很重要的任务是,将nandflash的最前面区域的bootloader(包含4k启动代码)拷贝到SDRAM中去,bootloader代码的大小是写好bootloader就确定的。然后只需要确定bootloader想映射到SDRAM的起始位置就ok
7. 在完成对nandflash上的bootloader搬移后,找到4k代码的搬移代码最后一个指令的下一个指令在SDRAM的bootloader的地址,然后跳转到该位置,继续执行bootloader的剩余代码(引导系统)

# 4. Why 4K？
　　其中还有一点有意思的需要注意，那就是为什么是4K代码的拷贝？我猜测应该是：那时SRAM应该还挺贵，所以只配置了4K，由此程序员就只能在4K大小内做文章，这个应该是由硬件决定了软件如何配置。那么现在便宜了，是不是芯片内部配置的SRAM变大了？然后我们就可以将整个UBOOT放在里面执行？你还别说，三星公司还真的这样做了，如下图:
![avatar](/img/in-post/Linux/201932702001.PNG)
　　芯片换代升级，变成S5PV210，此时启动的方式不像2440那么单一了，如SD卡启动，emmc启动，USB启动，等等启动方式，其次外部的内存也不再是SDRAM这种简单的内存了，而是DDR这种内存。所以一开始需要做的准备工作大大增加。

　　所以，S5PV210内部多了一个叫做IROM的东西，他的内部固化了一下程序，在上电之后IROM会启动内部的程序，将外部的某些设备进行简单的初始化，如SD卡，emmc等。内部的SRAM也从4k升级到了96k.就是为了满足更复杂的配置要求。

　　三星为Stage1设置了一个新的SRAM大小：16K，那么还剩下80K，三星就打算留个BL2阶段。然而Uboo并没有使用这种方式，原因是随着Uboot的发展，Uboot的大小变得很大，远超过了96K。就是说Uboot无法整个都放入片内的SRAM。那么让就不能采取三星的方式。

　　所以Uboot给BL1和BL2重新分配任务：（当然BL1的大小还只能是16K，这是应为IROM里的程序是固定的，它只会将前16K的内容拷贝到SRAM。）

　　实际上，三星就是想让BL1、BL2都放在SRAM中运行，然而事与愿违，Uboot也变大了，Uboot还是只将Stage1放在SRAM中执行，然后拷贝Stage2到SDRAM中去执行。
　　








　　
