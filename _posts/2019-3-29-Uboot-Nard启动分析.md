---
layout: post
title: "未完成-Uboot-Nard启动分析"
subtitle: Nardflash启动分析
author: "404"
header-style: text
tags:
  - Linux
  - Arm
  - Uboot
  - SDRAM
  - 未完成
---

# 1. Nardflash拷贝代码

　　通过前面的文章，我们都清楚了Uboot启动中拷贝的奥秘，同时既然是跟硬件打交道，那么肯定需要芯片手册(我这用的芯片手册是：K9F2G08U0C(NAND FLASH))。，这里我们就直接贴出Nardflash拷贝的源码：
```c
#ifdef CONFIG_S3C2440_NAND_BOOT
@
@ copy_myself: copy vivi to ram
@
copy_myself:
	mov	r10, lr

	@ reset NAND
	mov	r1, #NAND_CTL_BASE
	ldr	r2, =( (7<<12)|(7<<8)|(7<<4)|(0<<0) )
	str	r2, [r1, #oNFCONF]
	ldr	r2, [r1, #oNFCONF]

	ldr	r2, =( (1<<4)|(0<<1)|(1<<0) ) @ Active low CE Control
	str	r2, [r1, #oNFCONT] @ 这里是Control Register
	ldr	r2, [r1, #oNFCONT]

	ldr	r2, =(0x6)		@ RnB Clear
	str	r2, [r1, #oNFSTAT] @ 这里是NFSTAT Register
	ldr	r2, [r1, #oNFSTAT]

	mov	r2, #0xff		@ RESET command
	strb	r2, [r1, #oNFCMD] @ 这里是COMMOND Register
	mov	r3, #0			@ wait
1:	add	r3, r3, #0x1 @ 简简单单的等待一段时间
	cmp	r3, #0xa
	blt	1b
2:	ldr	r2, [r1, #oNFSTAT]	@ wait ready
	tst	r2, #0x4
	beq	2b

	ldr	r2, [r1, #oNFCONT]
	orr	r2, r2, #0x2		@ Flash Memory Chip Disable
	str	r2, [r1, #oNFCONT]


	@ get read to call C functions (for nand_read())
	ldr	sp, DW_STACK_START	@ setup stack pointer  @ 初始化栈以便后面调用c函数
	mov	fp, #0			@ no previous frame, so fp=0

	mov	r1, #GPIO_CTL_BASE
	add	r1, r1, #oGPIO_F
	mov	r2, #0xe0
	str	r2, [r1, #oGPIO_DAT]


	@ copy vivi to RAM
	ldr	r0, =VIVI_RAM_BASE
	mov     r1, #0x0  @ 设置需要开始拷贝的开始地址
	mov	r2, #0x20000  @ 设置需要开始拷贝的字节，即128K
	bl	nand_read_ll  @ 以上三个作为参数调用，传递给这个函数

#if 1
	mov	r1, #GPIO_CTL_BASE
	add	r1, r1, #oGPIO_F
	mov	r2, #0xb0
	str	r2, [r1, #oGPIO_DAT]
#endif


	tst	r0, #0x0
	beq	ok_nand_read
#ifdef CONFIG_DEBUG_LL
bad_nand_read:
	ldr	r0, STR_FAIL
	ldr	r1, SerBase
	bl	PrintWord
1:	b	1b		@ infinite loop
#endif

ok_nand_read:
#ifdef CONFIG_DEBUG_LL
	ldr	r0, STR_OK
	ldr	r1, SerBase
	bl	PrintWord
#endif

	@ verify

	mov	r0, #0
	ldr	r1, =0x33f00000
	mov	r2, #0x400	@ 4 bytes * 1024 = 4K-bytes
go_next:
	ldr	r3, [r0], #4
	ldr	r4, [r1], #4
	teq	r3, r4
	bne	notmatch
	subs	r2, r2, #4
	beq	done_nand_read
	bne	go_next
notmatch:
#ifdef CONFIG_DEBUG_LL
	sub	r0, r0, #4
	ldr	r1, SerBase
	bl	PrintHexWord
	ldr	r0, STR_FAIL
	ldr	r1, SerBase
	bl	PrintWord
#endif
1:	b	1b
done_nand_read:

#ifdef CONFIG_DEBUG_LL
	ldr	r0, STR_OK
	ldr	r1, SerBase
	bl	PrintWord
#endif

#if 1
	mov	r1, #GPIO_CTL_BASE
	add	r1, r1, #oGPIO_F
	mov	r2, #0x70
	str	r2, [r1, #oGPIO_DAT]
#endif

	mov	pc, r10

@ clear memory
@ r0: start address
@ r1: length
mem_clear:
	mov	r2, #0
	mov	r3, r2
	mov	r4, r2
	mov	r5, r2
	mov	r6, r2
	mov	r7, r2
	mov	r8, r2
	mov	r9, r2

clear_loop:
	stmia	r0!, {r2-r9}
	subs	r1, r1, #(8 * 4)
	bne	clear_loop

	mov	pc, lr

#endif @ CONFIG_S3C2440_NAND_BOOT

```

# 2. 源码分析

　　这个跟SDRAM寄存器初始化差不多，在使用前，都需要进行一些基本的初始化工作。`mov	r1, #NAND_CTL_BASE`这条指令中`NAND_CTL_BASE`定义如下，
```c
/* NAND Flash Controller */
#define NAND_CTL_BASE		0x4E000000

#define oNFCONF			0x00
```
　　我们可以看出，这里定义了一个地址，同时在S3C2440芯片手册中，我们可以看到对这个地址的定义，这个地址就是NANDflash控制器的地址。
![avatar](/img/in-post/Linux/201932901001.png)
　　同时也会将这个控制器初始化为`0x 0111 0111 0111 0000`，这些控制位对应如下：
![avatar](/img/in-post/Linux/201932901002.png)

　　接下来设置`CONTROL REGISTER`等设置见源码上我写的注释，一直到调用拷贝函数`nand_read_ll`,这个函数如下：
```c
/* low level nand read function */
int
nand_read_ll(unsigned char *buf, unsigned long start_addr, int size)
{
	int i, j;


	if ((start_addr & NAND_BLOCK_MASK) || (size & NAND_BLOCK_MASK)) {
		return -1;	/* invalid alignment */
	}
  /*到此应该可以明白s3c2440 nandflash 相关寄存器的确切含义了，就是说s3c2440里面已经集成了对nand flash操
作的相关寄存器，只要你的nand flash接线符合s3c2440 datasheet的接法，就可以随便使用s3c2440 对于nand
flash的相关寄存器，例如如果你想像nand flash写一个命令，那么只要对命令寄存器写入你的命令就可以了，s3c2440 可以自动帮你完成所有的时序动作，写地址也是一样。反过来说如果没有了对nand flash的支持，那么我们对nand falsh的操作就会增加好多对I/O口的控制，例如对CLE,ALE的控制。s3c2440已经帮我们完成了这部分工作了*/
	NAND_CHIP_ENABLE;

	for(i=start_addr; i < (start_addr + size);) {
		/* READ0 */
		NAND_CLEAR_RB;	    
		NFCMD = 0;

		/* Write Address */

/*下面这个送地址的过程可以说是这段程序里最难懂的一部分了，难就难于为什么送进nand flash的地址忽略了bit8，
纵观整个for(i) 循环，i并不是一个随机的地址，而应该是每一页的首地址。其实nand flash并不是忽略了bit 8这个
地址，而是bit 8早就被定下来了，什么时候定下来，就是上面的NFCMD = 0;语句，本flash (K9F1208U0B)支持从
半页开始读取，从而它有两个读的命令，分别是0x00(从一页的上半页开始读) 和 0x01(从一页的下半页开始读)，
当取0x00时，bit 8=0，当取0x01时 bit 8=1.*/
		NFADDR = i & 0xff;
		NFADDR = (i >> 9) & 0xff;
		NFADDR = (i >> 17) & 0xff;
		NFADDR = (i >> 25) & 0xff;

		NAND_DETECT_RB;

		for(j=0; j < NAND_SECTOR_SIZE; j++, i++) {
			*buf = (NFDATA & 0xff);
			buf++;
		}
	}
	NAND_CHIP_DISABLE;
	return 0;
}
```
　　这里还没有分析完，未完待续！！！









　　
