---
layout: post
title: "Uboot-Stage 1最后阶段分析"
subtitle: Uboot Stage最后阶段
author: "404"
header-style: text
tags:
  - Linux
  - Arm
  - Uboot
---

# 1.Uboot Stage1最后阶段

　　前面我们分析了Uboot启动做的一些事：
1. 设置SPV模式
2. 关中断
3. 关MMU
4. SDRAM初始化

　　现在到了Stage1最后阶段，我们是从NorFlash的角度来分析，即拷贝镜像到连接地址，设置堆栈，清除BSS代码段。这里有一个问题，为什么要设置堆栈呢？因为在下一阶段，我们会用到C，C更方面我们进行一些逻辑操作，方便我们来进行编写代码。

## 1.1 代码重定位
```c
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

　　这段代码就是比较加载地址是否与连接地址相同，如果不同，则拷贝相应的镜像到连接地址。至于为什么要拷贝到连接地址去执行，这就涉及到Arm中各个不同区域放置的物理内存了，我们在_TEXT_BASE那块地方放置的是SDRAM，这个物理内存执行速度、容量等更适合来执行代码。

## 1.2 设置栈
```c
stack_setup:
	ldr	r0, _TEXT_BASE		/* upper 128 KiB: relocated uboot   */
	sub	r0, r0, #CFG_MALLOC_LEN	/* malloc area                      */
	sub	r0, r0, #CFG_GBL_DATA_SIZE /* bdinfo                        */
#ifdef CONFIG_USE_IRQ
	sub	r0, r0, #(CONFIG_STACKSIZE_IRQ+CONFIG_STACKSIZE_FIQ)
#endif
	sub	sp, r0, #12		/* leave 3 words for abort-stack    */

clear_bss:
	ldr	r0, _bss_start		/* find start of bss segment        */
	ldr	r1, _bss_end		/* stop here                        */
	mov 	r2, #0x00000000		/* clear                            */

clbss_l:str	r2, [r0]		/* clear loop...                    */
	add	r0, r0, #4
	cmp	r0, r1
	ble	clbss_l
```
　　`TEXT_BASE = 0x33F80000` 其它的宏在 Smdk2410.h (include\configs):
```c
#define CFG_MALLOC_LEN		(CFG_ENV_SIZE + 128*1024)
#define CFG_ENV_SIZE		0x10000	/* Total Size of Environment Sector */
#define CFG_GBL_DATA_SIZE	128
#define CONFIG_STACKSIZE_IRQ	(4*1024)	/* IRQ stack */
#define CONFIG_STACKSIZE_FIQ	(4*1024)	/* FIQ stack */
```
```c
0x34000000:
(512K)				存放 uboot
0x33F80000：	 TEXT_BASE
(64K+128K == 192K)  mallo区
0x33F50000:
(128bytes)			global data区，后边会提到主要放的gd、bd全局结构体
0x33F4FF80:			
(4*1024*2)			IRQ+FIQ的栈
0x33F4DF80:
(12byte)			abort-stack,栈溢出
0x33F4DF74:			sp

```
![avatar](/img/in-post/Linux/201940201001.png)

## 1.3 清BSS
```c
clear_bss:
	ldr	r0, _bss_start		/* find start of bss segment        */
	ldr	r1, _bss_end		/* stop here                        */
	mov 	r2, #0x00000000		/* clear                            */

clbss_l:str	r2, [r0]		/* clear loop...                    */
	add	r0, r0, #4
	cmp	r0, r1
	ble	clbss_l
```

## 1.4 进入第二阶段
```c
ldr	pc, _start_armboot

_start_armboot:	.word start_armboot
```
　
