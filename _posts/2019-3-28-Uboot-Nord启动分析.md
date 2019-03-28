---
layout: post
title: "Uboot-Nord启动分析"
subtitle: Nordflash启动过程分析
author: "404"
header-style: text
tags:
  - Linux
  - Arm
  - Uboot
  - SDRAM
---

# 1. 基础介绍

　　基于前篇文章，我们知道了Uboot启动的的两个阶段以及两种不同介质Nandflash、Nordflash启动的不同之处，我们这篇文章就来分析Nordflash启动。

　　在Stage1、Stage2两个启动阶段中，其实主要不同之处就是Stage1阶段，Stage2阶段可以作为一个公共部分，两种不同介质启动方式均可用。而Stage1阶段不同之处就在于：
- Nordflash里面可以直接运行可执行代码，所以stage1可以直接运行在Nordflash中，关闭中断、关看门狗之类的，之后就负责将整个Uboot代码拷贝到SDRAM中
- Nardflash由于不可以直接执行，且不能直接和SDRAM交互，即不能直接拷贝到SDRAM中，所以有S3C2440芯片里面有一个SRAM作为Steeppingstone中，然后在SRAM中再做设置，拷贝Uboot执行程序到SDRAM中

　　基于此，我们这里就值分析不同之处，即Stage1中拷贝Uboot到SDRAM中。

# 2. Noarflash中的拷贝
```c
/*
 * we do sys-critical inits only at reboot,
 * not when booting from ram!
 */
#ifndef CONFIG_SKIP_LOWLEVEL_INIT
bl	cpu_init_crit
#endif

#ifndef CONFIG_SKIP_RELOCATE_UBOOT
relocate:				/* relocate U-Boot to RAM	    */
adr	r0, _start		/* r0 <- current position of code   */
ldr	r1, _TEXT_BASE		/* test if we run from flash or RAM */
cmp     r0, r1                  /* don't reloc during debug         */
beq     stack_setup

ldr	r2, _armboot_start
ldr	r3, _bss_start
sub	r2, r3, r2		/* r2 <- size of armboot            */
add	r2, r0, r2		/* r2 <- source end address         */

copy_loop:
ldmia	r0!, {r3-r10}		/* copy from source address [r0]    */
stmia	r1!, {r3-r10}		/* copy to   target address [r1]    */
cmp	r0, r2			/* until source end addreee [r2]    */
ble	copy_loop
#endif	/* CONFIG_SKIP_RELOCATE_UBOOT */
```
　　`cpu_init_crit`这部分我们已经在以前的一篇文章中分析过，所以现在不做过多介绍。紧接着我们来到`relocate`部分。

　　首先，`adr	r0, _start`,`ldr	r1, _TEXT_BASE`,然后接着一个cmp，就是比较加载地址是否等于链接地址，如果加载地址等于链接地址，说明不需要拷贝代码，现在就是在SDRAM中执行，我们来看看不相等的时候，事实上一般也就是不相等。

　　现在`ldr r2，_armboot_start`，这个`_armboot_start`是做什么的呢？我们在start.s中找到标号定义：
```c
.globl _armboot_start
_armboot_start:
	.word _start
```
　　实际上他就是等于程序的入口地址，紧接着将bss段开始加载到r2中，然后确定了需要拷贝的代码段大小即`r2=_bss_start - _armboot_start`，然后再用一个add制定，确定在加载位置的结束处，这里能知道加载位置和链接位置，就能轻轻松松理解。

　　然后利用寄存器，使用指令`ldmia`和`stmia`,将物理内存`0 ~ _bss_start-_armboot_start`拷贝到`0x33F80000 ~ _bss_start-_armboot_start`中。是的，在Nordflash中拷贝就是这么简单，这么任性！

　　由此Nordflash中拷贝部分到此完成，下一篇讲解Nardflash的拷贝。
　　
　


　　









　　
