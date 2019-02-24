---
layout: post
title: "Uboot源码分析＜一＞"
subtitle: 'Uboot启动源码分析'
author: "404"
header-style: text
tags:
  - Linux
  - Uboot
  - BootLoader
---
由前几章Bootloader的文章可知，Bootloader启动后大致可以分为两个阶段
1. stage1：
- 硬件设备初始化。
- 为加载 Boot Loader 的 stage2 准备 RAM 空间。
- 拷贝 Boot Loader 的 stage2 到 RAM 空间中。
- 设置好堆栈。
- 跳转到 stage2 的 C 入口点。
2. stage2：
- 初始化本阶段要使用到的硬件设备。
- 检测系统内存映射(memory map)。
- 将 kernel 映像和根文件系统映像从 flash 上读到 RAM 空间中。
- 为内核设置启动参数。
- 调用内核。

现在我们从Uboot源码来分析如何实现这些阶段，由于Uboot启动和具体硬件相关，我们以S3C2440为例来分析，期间会穿插具体硬件的介绍。  
#### 1. CPU模式切换成`SVC`

Uboot启动后将从`\cpu\arm920t`中`start.S`开始执行，具体代码：
```c
/*
 * the actual start code
 */

start_code:
	/*
	 * set the cpu to SVC32 mode
	 */
	mrs	r0,cpsr
	bic	r0,r0,#0x1f //bic位清除指令，r0 <- r0&~0x1f，最后5位清零
	orr	r0,r0,#0xd3 //将CPSR修改指定状态，本来应该只有5位，此时将IRQ和FIQ也屏蔽掉，即将SPSR后8位设置成`1101 0011`,相当于16进制的`0xd3`
	msr	cpsr,r0

	bl coloured_LED_init
	bl red_LED_on
```

在此，我们需了解S3C2440中的一些寄存器：
![avatar](/img/in-post/Linux/2019223001.png)
根据S3C2440手册，我们知道它共有37个寄存器，其中31个32位通用寄存器和6个状态寄存器，这些寄存器不能在同一时刻可见，而是根据处理器状态和运行模式来决定那些寄存器对使用者（Coder）可见。

在ARM状态，16个通用寄存器和一两个状态寄存器在任何时刻都可见。在特权模式下，将切换到分组寄存器模式。上图显示了各个寄存器在哪些模式可见：分组寄存器被标记了阴影三角形。

在ARM状态下，寄存器中有16个可直接访问寄存器：R0-R15。这些寄存器中，除了R15寄存器，其他的都可以用于存储数据或者地址。此外，还有第17个寄存器用来保存状态信息。

上述代码中，我们使用了R0和CPSR（状态寄存器）两个寄存器，其中R0作为通用寄存器，在这里作用相当于编程中的临时变量，用以对CPSR寄存器中的值修改并返回到CPSR寄存器中，CPSR寄存器中各个位的作用如下：
![avatar](/img/in-post/Linux/2019223002.png)
由于此时我们需要在SVC状态下进行工作，所以这段代码的作用就是将CPSR修改指定状态，本来应该只有5位，此时将IRQ和FIQ也屏蔽掉，即将SPSR后8位设置成`1101 0011`,相当于16进制的`0xd3`，bic那段代码是将R0中的临时变量后5位清零。此时进入`SVC`运行模式。各个模式的bit位设置如下：
![avatar](/img/in-post/Linux/2019223003.png)
接下来两条跳转指令`bl`，跳转到的函数`coloured_LED_init`、`coloured_LED_init`均为空，可以暂时不用管。

#### 2. relocate execption table

接下来的代码如下：
```c
#if	defined(CONFIG_AT91RM9200DK) || defined(CONFIG_AT91RM9200EK)
	/*
	 * relocate exception table
	 * /
	ldr	r0, =_ start
	ldr	r1, =0x0
	mov	r2, #16
copyex:
	subs	r2, r2, #1
	ldr	r3, [r0], #4 //相当于*r3 = * r0，r0 = r0 + 4
	str	r3, [r1], #4
	bne	copyex
#endif
```
在这里源码标注为\"relocate exception table\"，我分析感觉是没有relocate的。因为根据链接地址知道`_start = 0x0`的,同时ldr给r1赋值也为0x0，相当于值仍然是给原地址，如果需要重新"relocate exception table"，那么则需要更改r1的初始化赋值`ldr r1，=0x0`。
同时解释下为什么`_start = 0x0`，在代码开头有：
```
OUTPUT_FORMAT("elf32-littlearm", "elf32-littlearm", "elf32-littlearm")
OUTPUT_ARCH(arm)  //设置输出文件的体系架构
ENTRY(_ start)  //将_start这个全局符号设置成入口地址
SECTIONS
{
  . = 0x00000000; //连接地址

  . = ALIGN(4);
```
由此知道`_start = 0x0`。

#### 3. 关闭关门狗、中断、ＭＭＵ
此阶段源码如下：
```
#if defined(CONFIG_S3C2400) || defined(CONFIG_S3C2410)
	/* turn off the watchdog */

# if defined(CONFIG_S3C2400)
#  define pWTCON		0x15300000
#  define INTMSK		0x14400008	/* Interupt-Controller base addresses */
#  define CLKDIVN	0x14800014	/* clock divisor register */
#else
#  define pWTCON		0x53000000
#  define INTMSK		0x4A000008	/* Interupt-Controller base addresses */
#  define INTSUBMSK	0x4A00001C
#  define CLKDIVN	0x4C000014	/* clock divisor register */
# endif

	ldr     r0, =pWTCON
	mov     r1, #0x0
	str     r1, [r0]

	/*
	 * mask all IRQs by setting all bits in the INTMR - default
	 */
	mov	r1, #0xffffffff
	ldr	r0, =INTMSK
	str	r1, [r0]
# if defined(CONFIG_S3C2410)
	ldr	r1, =0x3ff
	ldr	r0, =INTSUBMSK
	str	r1, [r0]
# endif

	/* FCLK:HCLK:PCLK = 1:2:4 */
	/* default FCLK is 120 MHz ! */
	ldr	r0, =CLKDIVN
	mov	r1, #3
	str	r1, [r0]
#endif	/* CONFIG_S3C2400 || CONFIG_S3C2410 */

	/*
	 * we do sys-critical inits only at reboot,
	 * not when booting from ram!
	 */
#ifndef CONFIG_SKIP_LOWLEVEL_INIT
	bl	cpu_init_crit
#endif
```
在此需要了解下看门狗，在ARM中，有一个硬件部分叫WATCH DOG。这个硬件，一直在做一件事情：就是，从某一数值，一直数，各一段时间减一，隔一段时间减一，直到减到0的时候将会触发重启或者中断。而有时候，为了预防死机，我们在操作系统跑起来的时候会有一个特定的程序来做一件事情：减到特定是值的时候数值将会重新置到100.这样，看门狗将会循环往复做一件事情：一直数数，而不会死机。这个程序叫做守护程序：又叫做`喂狗程序`。

看门狗程序有一个寄存器，寄存器地址及各个bit作用如下：
![avatar](/img/in-post/Linux/2019223004.png)
为了简单，我们将该寄存器置为0，而不去具体设置`bit[5]`,此时关门狗程序被关闭。
同理，IRQ寄存器设置如下，地址对应上诉`INTMSK`：
![avatar](/img/in-post/Linux/2019223005.png)
再就是设置时钟频率为120MHz。
接下来，如果定义了`CONFIG_SKIP_LOWLEVEL_INIT`则跳转到`cpu_init_crit`，那么`cpu_init_crit`是做什么的呢，如下：
```
cpu_init_crit:
	/*
	 * flush v4 I/D caches
	 */
	mov	r0, #0
	mcr	p15, 0, r0, c7, c7, 0	/* flush v3/v4 cache */
	mcr	p15, 0, r0, c8, c7, 0	/* flush v4 TLB */

	/*
	 * disable MMU stuff and caches
	 */
	mrc	p15, 0, r0, c1, c0, 0
	bic	r0, r0, #0x00002300	@ clear bits 13, 9:8 (--V- --RS)
	bic	r0, r0, #0x00000087	@ clear bits 7, 2:0 (B--- -CAM)
	orr	r0, r0, #0x00000002	@ set bit 2 (A) Align
	orr	r0, r0, #0x00001000	@ set bit 12 (I) I-Cache
	mcr	p15, 0, r0, c1, c0, 0

	/*
	 * before relocating, we have to setup RAM timing
	 * because memory timing is board-dependend, you will
	 * find a lowlevel_init.S in your board directory.
	 */
	mov	ip, lr

	bl	lowlevel_init

	mov	lr, ip
	mov	pc, lr
```
MCR 指令用于将ARM 处理器寄存器中的数据传送到协处理器寄存器中，格式为：
- MCR 协处理器编号，协处理器操作码1，源寄存器，目的寄存器1，目的寄存器2，协处理器操作码2。
- 其中协处理器操作码1 和协处理器操作码2 为协处理器将要执行的操作，
- 源寄存器为ARM 处理器的寄存器，目的寄存器1 和目的寄存器2 均为协处理器的寄存器。

第5~6行，使I/D cache失效： 协处理寄存器操作，将r0中的数据写入到协处理器p15的c7中，c7对应cp15的cache控制寄存器  
第7行，使TLB操作寄存器失效：将r0数据送到cp15的c8、c7中。C8对应TLB操作寄存器  
第12行，将c1、c0的值写入到r0中  
第17行，将设置好的r0值写入到协处理器p15的c1、c0中，关闭MMU  
第24行，将lr寄存器内容保存到ip寄存器中，用于子程序调用返回

然后跳转到`lowlevel_init`,`lowlevel_init`函数如下（此处用的是2410代码）：
```
lowlevel_init:
	/* memory control configuration */
	/* make r0 relative the current location so that it */
	/* reads SMRDATA out of FLASH rather than memory ! */
	ldr     r0, =SMRDATA
	ldr	r1, _TEXT_BASE
	sub	r0, r0, r1
	ldr	r1, =BWSCON	/* Bus Width Status Controller */
	add     r2, r0, #13*4
0:
	ldr     r3, [r0], #4
	str     r3, [r1], #4
	cmp     r2, r0
	bne     0b

	/* everything is fine now */
	mov	pc, lr

	.ltorg
/* the literal pools origin */

SMRDATA:
    .word (0+(B1_BWSCON<<4)+(B2_BWSCON<<8)+(B3_BWSCON<<12)+(B4_BWSCON<<16)+(B5_BWSCON<<20)+(B6_BWSCON<<24)+(B7_BWSCON<<28))
    .word ((B0_Tacs<<13)+(B0_Tcos<<11)+(B0_Tacc<<8)+(B0_Tcoh<<6)+(B0_Tah<<4)+(B0_Tacp<<2)+(B0_PMC))
    .word ((B1_Tacs<<13)+(B1_Tcos<<11)+(B1_Tacc<<8)+(B1_Tcoh<<6)+(B1_Tah<<4)+(B1_Tacp<<2)+(B1_PMC))
    .word ((B2_Tacs<<13)+(B2_Tcos<<11)+(B2_Tacc<<8)+(B2_Tcoh<<6)+(B2_Tah<<4)+(B2_Tacp<<2)+(B2_PMC))
    .word ((B3_Tacs<<13)+(B3_Tcos<<11)+(B3_Tacc<<8)+(B3_Tcoh<<6)+(B3_Tah<<4)+(B3_Tacp<<2)+(B3_PMC))
    .word ((B4_Tacs<<13)+(B4_Tcos<<11)+(B4_Tacc<<8)+(B4_Tcoh<<6)+(B4_Tah<<4)+(B4_Tacp<<2)+(B4_PMC))
    .word ((B5_Tacs<<13)+(B5_Tcos<<11)+(B5_Tacc<<8)+(B5_Tcoh<<6)+(B5_Tah<<4)+(B5_Tacp<<2)+(B5_PMC))
    .word ((B6_MT<<15)+(B6_Trcd<<2)+(B6_SCAN))
    .word ((B7_MT<<15)+(B7_Trcd<<2)+(B7_SCAN))
    .word ((REFEN<<23)+(TREFMD<<22)+(Trp<<20)+(Trc<<18)+(Tchr<<16)+REFCNT)
    .word 0x32
    .word 0x30
    .word 0x30

```

第5行，将保存设置内存控制器的参数的初始地址（连接地址），保存到r0寄存器中。  
第6行，将程序连接的入口地址存入到r1寄存器中  
第7行，r0=r0-r1，获取保存设置内存控制器的参数的初始地址（此处代码未进行定位，代码处于加载地址中，获取的是在存储器存放的物理地址）  
第8行，将内存控制器的第一个寄存器地址存到r1寄存器中  
第9行，获取保存设置内存控制器的参数的结束地址地址（此处代码未进行定位，代码处于加载地址中，获取的是在存储器存放的物理地址）  
第10~14行，将标号SMRDATA地址处存放的参数，写入到相应寄存器中，设置内存控制器工作方式  
第17行，程序调用返回，返回调用节点  

具体寄存器见此链接。
