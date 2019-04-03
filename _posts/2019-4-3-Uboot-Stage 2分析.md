---
layout: post
title: "Uboot-Stage 2分析"
subtitle: Uboot-Stage 2分析
author: "404"
header-style: text
tags:
  - Linux
  - Arm
  - Uboot
---
　　在进行Stage 2阶段代码分析之前，我们先放置一张概貌图：
![avatar](/img/in-post/Linux/201940301001.png)

# 1. Uboot第二阶段做什么

　　Uboot的第二阶段就是要初始化第一阶段剩下的还没被初始化的硬件。进行单板级别的初始化，初始化 nandflash 、norflash 、初始化串口、设置环境变量、最终跳转到 main_loop 里，接收串口传递进来的各种命令。

　　那么Uboot何时结束？如下：
1. Uboot启动后自动运行打印出很多信息（这些信息就是Uboot在第一和第二阶段不断进行初始化时，打印出来的信息）。然后Uboot进入了倒数bootdelay秒然后执行bootcmd对应的启动命令。
2. 如果用户没有干涉则会执行bootcmd进入自动启动内核流程（Uboot就死掉了）；此时用户可以按下回车键打断Uboot的自动启动进入Uboot的命令行下。然后Uboot就一直工作在命令行下。
3. Uboot的命令行就是一个死循环，循环体内不断重复：接收命令、解析命令、执行命令。这就是Uboot最终的归宿。

# 2. Uboot 启动分析

## 2.1 Uboot主要数据结构

　　在分析Uboot代码前，我们先看看其中用到的两个主要的数据结构：
```c
/* 全局数据结构 */
typedef	struct	global_data {
	bd_t            *bd;            /* 指向板级信息结构 */
	unsigned long   flags;          /* 标记位 */
	unsigned long   baudrate;       /* 串口波特率 */
	unsigned long   have_console;   /* serial_init() was called */
	unsigned long   env_addr;       /* 环境参数地址 */
	unsigned long   env_valid;      /* 环境参数 CRC 校验有效标志 */
	unsigned long   fb_base;        /* fb 起始地址 */
#ifdef CONFIG_VFD
	unsigned char   vfd_type;       /* 显示器类型(VFD代指真空荧光屏) */
#endif
#if 0
	unsigned long   cpu_clk;        /* cpu 频率*/
	unsigned long   bus_clk;        /* bus 频率 */
	phys_size_t     ram_size;       /* ram 大小 */
	unsigned long   reset_status;   /* reset status register at boot */
#endif
	void            **jt;           /* 跳转函数表 */
} gd_t;

/* Global Data Flags */
#define	GD_FLG_RELOC	0x00001		/* 代码已经转移到 RAM */
#define	GD_FLG_DEVINIT	0x00002		/* 设备已经完成初始化 */
#define	GD_FLG_SILENT	0x00004		/* 静音模式 */
#define	GD_FLG_POSTFAIL	0x00008		/* Critical POST test failed */
#define	GD_FLG_POSTSTOP	0x00010		/* POST seqeunce aborted */
#define	GD_FLG_LOGINIT	0x00020		/* Log Buffer has been initialized	*/
#define GD_FLG_DISABLE_CONSOLE	0x00040		/* Disable console (in & out)	 */
/* 定义一个寄存器变量，占用寄存器r8，作为 gd_t 的全局指针 */
#define DECLARE_GLOBAL_DATA_PTR     register volatile gd_t *gd asm ("r8")

/* 板级信息结构 */
typedef struct bd_info {
    int                   bi_baudrate;	   /* serial console baudrate */
    unsigned long         bi_ip_addr;	   /* IP Address */
    struct environment_s  *bi_env;         /* 板子的环境变量 */
    ulong                 bi_arch_number;  /* 板子的 id */
    ulong                 bi_boot_params;  /* 板子的启动参数 */
    struct                                 /* RAM 配置 */
    {
        ulong start;
        ulong size;
    } bi_dram[CONFIG_NR_DRAM_BANKS];
} bd_t;

/**************************************************************************
 *
 * 每个环境变量以形如"name=value"的字符串存储并以'\0'结尾，环境变量的尾部以两个'\0'结束。
 * 新增的环境变量都依次添加在尾部，如果删除一个环境变量需要将其后面的向前移动
 * 如果替换一个环境变量需要先删除再新增。
 *
 * 环境变量采用 32 bit CRC 校验.
 *
 **************************************************************************
 */

/* 我们平台定义了8kB大小的环境变量存储区，地址空间位于4M ~ 6M */
#define CONFIG_ENV_IS_IN_NAND
#define CONFIG_ENV_OFFSET  0x00400000   /* 4M ~ 6M */
#define CONFIG_ENV_SIZE    8192         /* 8KB */
#define CONFIG_ENV_RANGE   0x00200000

#define ENV_SIZE (CONFIG_ENV_SIZE - ENV_HEADER_SIZE)
/* 环境变量结构 */
typedef	struct environment_s {
	uint32_t  crc;                 /* CRC32 over data bytes */
#ifdef CONFIG_SYS_REDUNDAND_ENVIRONMENT
	unsigned char flags;           /* active/obsolete flags */
#endif
	unsigned char data[ENV_SIZE];  /* Environment data */
} env_t;

```
　　这两个类型变量记录了刚启动时的信息，还将记录作为引导内核和文件系统的参数，如 bootargs 等，并且将来还会在启动内核时，由 Uboot 交由 kernel 时会有所用。

　　Uboot使用了一个存储在寄存器中的指针gd来记录全局数据区的地址：
```c
#define DECLARE_GLOBAL_DATA_PTR     register volatile gd_t *gd asm ("r8")
```
　　DECLARE_GLOBAL_DATA_PTR定义一个gd_t全局数据结构的指针，这个指针存放在指定的寄存器r8中。这个声明也避免编译器把r8分配给其它的变量。任何想要访问全局数据区的代码，只要代码开头加入“DECLARE_GLOBAL_DATA_PTR”一行代码，然后就可以使用gd指针来访问全局数据区了。
```c
gd = (gd_t*)(_armboot_start - CFG_MALLOC_LEN - sizeof(gd_t));
```

## 2.2 init_sequence初始化
```c
for (init_fnc_ptr = init_sequence; *init_fnc_ptr; ++init_fnc_ptr) {
  if ((*init_fnc_ptr)() != 0) {
    hang ();
  }
}

typedef int (init_fnc_t) (void);

int print_cpuinfo (void); /* test-only */

init_fnc_t *init_sequence[] = {
	cpu_init,		/* basic cpu dependent setup */
	board_init,		/* basic board dependent setup */
	interrupt_init,		/* set up exceptions */
	env_init,		/* initialize environment */
	init_baudrate,		/* initialze baudrate settings */
	serial_init,		/* serial communications setup */
	console_init_f,		/* stage 1 init of console */
	display_banner,		/* say that we are here */
#if defined(CONFIG_DISPLAY_CPUINFO)
	print_cpuinfo,		/* display cpu info (and speed) */
#endif
#if defined(CONFIG_DISPLAY_BOARDINFO)
	checkboard,		/* display board info */
#endif
	dram_init,		/* configure available RAM banks */
	display_dram_config,
	NULL,
};
```
### 2.2.1 cpu_init
```c
int cpu_init (void)
{
	/*
	 * setup up stacks if necessary
	 */
#ifdef CONFIG_USE_IRQ
	IRQ_STACK_START = _armboot_start - CFG_MALLOC_LEN - CFG_GBL_DATA_SIZE - 4;
	FIQ_STACK_START = IRQ_STACK_START - CONFIG_STACKSIZE_IRQ;
#endif
	return 0;
}
```
　　如果使用 irq 的话，将这两个宏指向之前分配的栈空间。由于我们在Stage1阶段已经做过CPU初始化，所以这里没啥事。

### 2.2.2 board_init
```c
int board_init (void)
{
  //获得时钟寄存器的地址，s3c24x0通用无需修改。
  S3C24X0_CLOCK_POWER * const clk_power = S3C24X0_GetBase_CLOCK_POWER();
  //获得GPIO寄存器的地址，s3c24x0不通用需要增加2440GPIO寄存器。
  S3C24X0_GPIO * const gpio = S3C24X0_GetBase_GPIO();
  /* to reduce PLL lock time, adjust the LOCKTIME register */
  clk_power->LOCKTIME = 0xFFFFFF;
  /* configure MPLL */
  /*MPLL初始化 参数位于./board/smdk2410/smdk2410.c中，2440应有自己的文件以及定义*/
  clk_power->MPLLCON = ((M_MDIV << 12) + (M_PDIV << 4) + M_SDIV);
  /* some delay between MPLL and UPLL */
  delay (4000);
  /* configure UPLL */
  clk_power->UPLLCON = ((U_M_MDIV << 12) + (U_M_PDIV << 4) + U_M_SDIV);
  /* some delay between MPLL and UPLL */
  delay (8000);
  /* set up the I/O ports */
  gpio->GPACON = 0x007FFFFF;
  gpio->GPBCON = 0x00044555;
  gpio->GPBUP = 0x000007FF;
  gpio->GPCCON = 0xAAAAAAAA;
  gpio->GPCUP = 0x0000FFFF;
  gpio->GPDCON = 0xAAAAAAAA;
  gpio->GPDUP = 0x0000FFFF;
  gpio->GPECON = 0xAAAAAAAA;
  gpio->GPEUP = 0x0000FFFF;
  gpio->GPFCON = 0x000055AA;
  gpio->GPFUP = 0x000000FF;
  gpio->GPGCON = 0xFF95FFBA;
  gpio->GPGUP = 0x0000FFFF;
  gpio->GPHCON = 0x002AFAAA;
  gpio->GPHUP = 0x000007FF;
  /* arch number of SMDK2410-Board */
  gd->bd->bi_arch_number = MACH_TYPE_SMDK2410;
  //#define MACH_TYPE_SMDK2410 193 位于./include/asm-asm/Mach-types.h中定义
  /* adress of boot parameters */
  gd->bd->bi_boot_params = 0x30000100; //内核启动参数存放地址
  //开数据 指令cache
  icache_enable();
  dcache_enable();
  return 0;
}
```
　　重点工作，向 gd 结构体中记录了 机器ID 以及 tag 的存放地址，这俩都是要传递给内核的。

### 2.2.3 interrupt_init
　　PWM的初始化，无关紧要

### 2.2.4 env_init 环境变量初始化
```c
typedef struct environment_s {
	ulong crc;			/* CRC32 over data bytes    */
	uchar flags;			/* active or obsolete */
	uchar *data;
} env_t;

env_t *env_ptr = (env_t *)CFG_ENV_ADDR;
static env_t *flash_addr = (env_t *)CFG_ENV_ADDR;

int  env_init(void)
{
#ifdef CONFIG_OMAP2420H4
	int flash_probe(void);

	if(flash_probe() == 0)
		goto bad_flash;
#endif
	if (crc32(0, env_ptr->data, ENV_SIZE) == env_ptr->crc) {
		gd->env_addr  = (ulong)&(env_ptr->data);
		gd->env_valid = 1;
		return(0);
	}
#ifdef CONFIG_OMAP2420H4
bad_flash:
#endif
	gd->env_addr  = (ulong)&default_environment[0];
	gd->env_valid = 0;
	return (0);
}
```
　　环境变量用一个 environment_s 结构来描述，它被放置在 norflash 的 0x70000 起始的地址处，crc 是环境变量的校验和，全部的环境变量都以字符串的形式存放在 data 数组中，两个环境变量之间用 “\0” 隔开。

　　现在env_ptr指向 Norfalsh 中的0x070000,进行校验，判断环境变量是否可用。
1、uboot 第一次启动，那么 norflash 这个地址处并没有任何东西，校验失败，则使用默认的环境变量，使全局指针 gd->env_addr 指向内存中的默认环境变量，并设置标志位 gd->env_valid 为 0 。
2、uboot 非第一次启动，那么校验成功，将全局指针 gd->env_addr 指向环境变量，并使标志位 gd->env_valid 置一。

　　默认环境变量的 定义 CONFIG_BOOTARGS 等宏在 Smdk2410.h (include\configs)

### 2.2.5 init_baudrate
　　设置波特率 首先从环境变量中读取，如果没有则设置为 115200

### 2.2.6 serial_init 串口初始化

```c
int serial_init (void)
{
	serial_setbrg ();

	return (0);
}

void serial_setbrg (void)
{
	S3C24X0_UART * const uart = S3C24X0_GetBase_UART(UART_NR);
	int i;
	unsigned int reg = 0;

	/* value is calculated so : (int)(PCLK/16./baudrate) -1 */
	reg = get_PCLK() / (16 * gd->baudrate) - 1;

	/* FIFO enable, Tx/Rx FIFO clear */
	uart->UFCON = 0x07;
	uart->UMCON = 0x0;
	/* Normal,No parity,1 stop,8 bit */
	uart->ULCON = 0x3;
	/*
	 * tx=level,rx=edge,disable timeout int.,enable rx error int.,
	 * normal,interrupt or polling
	 */
	uart->UCON = 0x245;
	uart->UBRDIV = reg;

#ifdef CONFIG_HWFLOW
	uart->UMCON = 0x1; /* RTS up */
#endif
	for (i = 0; i < 100; i++);
}
```
　　串口寄存器的设置。

### 2.2.7 console_init_f 控制台初始化，无关紧要

### 2.2.8 display_banner
　　打印代码段 BSS段等地址信息

### 2.2.9 dram_init
```c
int dram_init (void)
{
	gd->bd->bi_dram[0].start = PHYS_SDRAM_1;
	gd->bd->bi_dram[0].size = PHYS_SDRAM_1_SIZE;

	return 0;
}
```
　　在全局指针 gd 中标记内存范围。
