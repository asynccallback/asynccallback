---
layout: post
title: "NANDflash芯片分析"
subtitle: NANDflash芯片结构分析
author: "404"
header-style: text
tags:
  - Linux
  - Arm
  - Uboot
  - NANDflash
---

　　在写前面一篇NANDflash在Uboot启动过程中代码拷贝时，遇到一些硬件设置的时候，感觉很模糊，不知道代码这么设置的原因，不知道拷贝到底如何拷贝，现在就来写这篇文章来进行NANDflash芯片内部体系分析。

# 1 NAND Flash介绍和NAND Flash控制器使用

　　NANDFlash在嵌入式系统中的地位与PC上硬盘类似，用于保存存放系统运行所必须的操作系统、应用程序、用户数据、运行过程中产生的各类数据。与内存掉电后数据丢失不同，NANDflash中的数据在掉电后可永久保存。

## 1.1 Flash介绍

　　常用的Flash类型有NORFlash和NANDflash两种。NORflash由Intel公司在1988年发明，以代替当时在市场上占据主导地位的EPROM和$E^2PROM$。NANDflash由Toshiba公司在1989年发明。两者的主要差别如下：

　|NOR|NAND
容量　|1MB~3MB|16MB~512MB
XIP | YES | NO
擦写　|非常慢（5s）|快（3ms）
写　|慢|快
读　|快|快
可靠性 　|比较高，位反转比例小于NANDflash的10%|比较低，位反转比较常见，必须有校验措施，比如TNR必须有坏块管理措施
可擦除次数  |10000~100000|100000~1000000
生命周期 　|低于NANDflash的10%|是NORflash的10倍以上
接口|与RAM接口相同|I/O接口
访问方法|随机访问|顺序访问
易用性|容易|复杂
主要用途|常用于保存代码和关键数据|用于保存数据
价格|高|低

　　NORflash支持XIP,即代码可以直接在NORflash上执行，无需复制到内存中。这是由于NORflash的接口与RAM完全相同，可随机访问任意地址的数据。在NORflash上进行读操作的效率非常高，但擦除和写的效率很低；另外，NORflash的容量一般比较小。NANDflash进行擦除和写的效率更高，并且容量更大。一般而言，NORflash用于存储程序，NANDflash用于存储数据。基于NANDflash的设备通常也要搭配NORflash以存储程序。

　　Flash存储器件由擦除单元（也称为块）组成，当要写某个块时，需要确保这个块已经被擦除。NOR Flash的块大小范围是64KB ~ 128KB; NANDflash的块大小范围是8KB~64KB，擦/写一个NORflash块需4s，而擦/写一个NANDflash仅需2ms。NORflash的块太大，不仅增加了擦写时间，对于给定的写操作，Norflash也需要更多的擦除操作，特别是小文件，比如一个文件只有1KB，但是为了保存它却需要擦除大小为64KB ~128KB的NORflash块。

　　NORflash的接口与RAM完全相同，可以随意访问任意地址的数据。而NANDflash的接口仅仅包含几个I/O引脚，需要串行地访问。NANDflash一般以512字节为单位进行读写。这使得NORflash适合于运行程序，而NANDFlash更适合于存储数据。

　　容量相同的情况下，NANDflash的体积更小，对于空间有严格要求的系统，NANDflash可以节省更多空间。市场上NORflash的容量通常为1MB~4MB（也有32MB的NORflash），NANDflash的容量为8MB~512MB。容量的差别也使得NORflash多用于存储程序，NANDflah多用于存储数据。

　　对于Flash存储器件的可靠性需要考虑3点：位反转、坏块和可擦除次数。所有Flash器件都遭遇位反转的问题；由于Flash固有的电器特性，在读写数据过程中，偶然会产生一位或几位数据错误，而NANDflash出现的概率远大于NORflahs。当位反转发生在关键的代码、数据上时，有可能导致系统崩溃。当仅仅是报告位反转，重新读取即可；如果确实发生了位反转，则必须有相应的错误检测/恢复措施。当NANDflash上发生位反转的概率更高，推荐使用EDC/ECC进行错误检测和恢复。NANDflash上面会有坏块随机分布，在使用前需要将坏块扫描出来，确保不再使用它们，否则会使产品含有严重的故障。NANDflash没块的可擦除次数通常在10000次左右没事NORflash的10倍。另外，因为NANDflash的块大小通常是NORflash的1/8，所以NANDflash的寿命远远超过NORflash。

　　嵌入式Linux对Nor、NANDflash的软件支持都很成熟。在NORflash上常用jffs2文件系统，而在NANDflash上常用yaffs文件系统。在更底层，有MTD驱动程序实现对它们的读、写、擦除操作，它也实现了EDC/ECC校验。

## 1.2 NAND Flash的物理结构

　　以NAND Flash K9F1208U0M为例，K9F1208U0M是三星公司生产的容量为64MB的NAND Flash，常用于手持设备等消费电子产品。它的封装及外部引脚如下图：
![avatar](/img/in-post/Linux/201933001001.png)
K9F1208U0M的功能结构图如图所示：
![avatar](/img/in-post/Linux/201933001002.png)

　　K9F1208U0M的内部结构分为10个功能部件：
1. X-Buffers Latche & Decoders：用于行地址
2. Y-Buffers Latche & Decoders：用于列地址
3. Command Register：用于命令字
4. Control Logic & High Voltage Generator：控制逻辑及产生Flash所需高压
5. Nand Flash Array：存储部件
6. Page Register & S/A：页存储器，当读、写某页时，会将数据线读入/写入此寄存器，大小为528字节
7. Y-Gating
8. I/O Buffers & Latches
9. Global Buffers

　　NAND Flash存储单元组织结构如下图：
![avatar](/img/in-post/Linux/201933001003.png)

　　K9F1208U0M容量为528Mbit，分为131072行（页）、528列；每一页大小为512字节，外加16字节的额外空间，这16字节额外空间的列地址为512~527。

　　命令、地址、数据都通过8个I/O口输入/输出，这种形式减少了芯片的引脚个数，并使得系统很容易升级到更大的容量。写入命令、地址或数据时，都需要将WE#、CE#信号同时拉低。数据在WE#信号的上升沿被NAND Flash锁存；命令锁存信号CLE、地址锁存信号ALE用来分辨、锁存命令或地址。K9F1208U0M的64MB存储空间需要26位地址，因此以字节为单位访问Flash时需要4个地址序列：列地址、行地址的低位部分、行地址的高位部分。读/写页在发出命令后，需要4个地址序列，而查出块在发出命令后仅需要3个地址序列。

## 1.3 NAND Flash访问方法

### 1.3.1 硬件连接

　　NAND Flash与S3C2410/S3C2440的硬件连接如图：
![avatar](/img/in-post/Linux/201933001004.png)
　　NAND Flash与S3C2410/S3C2440的连线比较少：8个I/O引脚（$IO_0$~$IO_7$)、5个使能信号（nWE、ALE、CLE、nCE、nRE）、1个状态引脚（RDY/B）、1个写保护引脚（nWP）。地址、数据和命令都是在这些使能信号的配合下，通过8个I/O引脚传输。写地址、数据、命令时，nCE、nWE信号必须为低电平，它们在nWE信号的上升沿被锁存。命令锁存使能信号CLE和地址锁存信号ALE用来区分I/O引脚上传输的时命令还是地址。

### 1.3.2 命令字及操作方法

　　操作NAND Flash时，线传输命令，然后传输地址，最后读/写数据，期间要检查Flash的状态。对于K9F1208U0M,它的容量为64MB，需要一个26位的地址。发出命令后，后面要紧跟着4个地址序列。比如读Flash时，发出读命令和4个地址序列后，后续的读操作就可以得到这个地址及其后续地址的数据。相应的命令字和地址序列如下：
![avatar](/img/in-post/Linux/201933001005.png)
![avatar](/img/in-post/Linux/201933001006.png)
　　K9F1208U0M一页大小为528字节，而列地址$A_0$~$A_7$可以寻址的范围是256字节，所以必须辅以其他手段才能完全寻址这528字节。将一页分为A、B、C三个区：A区为0~255字节，B区为256~511字节，C区为512~527字节。访问某页时，需要选定特定的区，这称为“是地址指针指向特定的区”。这通过3个命令来实现：命令00h让地址指针指向A区、命令01h让地址指针指向B区、命令50h让地址指针指向C区。命令00h和50h会使得访问Flash的地址指针一直从A区或C区开始，除非发出了其他的修改地址指针的命令。命令01h的效果只能维持一次，当前的读、写、擦除、复位或者上电操作完成后，地址指针重新指向A区。写A区或C区的数据时，必须在发出命令80h之前发出命令00h或者50h；写B区的数据时，发出命令01h后必须紧接着就发出命令80h，下图更好的说明了这个特性：　
![avatar](/img/in-post/Linux/201933001007.png)
　　下面逐个讲解上表中的命令字：   
　　**Read 1：** 命令字为00h或01h   
　　如图所示，发出命令00h或01h后，就选定了读操作时从A区还是B区开始。列地址$A_0$~$A_7$可以寻址的范围是256字节，命令00h和01h使得可以在512字节大小的页内任意寻址——这相当于$A_8$被命令00h设为0，而被命令01h设为1。   
　　发出命令字后，依据表，发出4个地址序列，然后就可以检测R/nB引脚以确定Flash是否准备好。如果准备好了，就可以发起读操作依次读入数据。

　　**Read 2：** 命令字50h   
　　与Read 1命令类似，不过读取的时C区数据，操作序列为：发出命令字50h、发出4个地址序列、等待R/nB引脚为高，最后读取数据。不同的时，地址序列中$A_0$~$A_3$用于设定C区（大小为16字节）要读取的其实地址，$A_4$~$A_7$被忽略。

　　**Read ID：** 命令字90h   
　　发出命令字90h，发出4个地址序列（都设为0），然后就可以连续读入5个数据，分别表示：厂商代码（对于SAMSUNG公司为Ech）、设备代码（对于K9F1208U0M为76h）、保留的字节、多层操作代码

　　**Reset：** 命令字为FFh   
　　发出命令字FFh即可复位NAND Flash芯片。如果芯片正处于读、写、擦除状态，复位命令会终止这些命令。

　　**Page Program（TRUE)：** 命令字分为两阶段，80h和10h   
　　它的操作序列如下：
![avatar](/img/in-post/Linux/201933001008.png)
　　NAND Flash的写操作一般是以页为单位的，但是可以只写一页中的一部分。发出命令字80h后，紧接着是4个地址序列，然后向Flash发送数据（最大可以达到528字节），然后发出命令字10h启动写操作，此时Flash内部会自动完成写、校验操作。一旦发出命令字10h后，就可以通过读状态命令70h获知当前写操作是否完成、是否成功。

## 1.4 S3C2440 NAND Flash控制器介绍

　　NAND Flash控制器提供几个寄存器来简化对NAND Flash的操作。比如要发出读命令时，只需要往NFCMD寄存器中写入0即可，NAND Flash控制器会自动发出各种控制信号。

### 1.4.1 操作方法概述

　　访问NAND Flash时需要先发出命令，然后发出地址序列，最后读/写数据：需要使用各个使能信号来分辨是命令、地址还是数据。S3C2410的NAND Flash控制器提供了NFCONF、NFCMD、NFADDR、NFDATA、NFSTAT和NFECC等6个寄存器来简化这些操作。S3C2440的NAND Flash控制器则提供了NFCONF、NFCONT、NFCMMD、NFADDR、NFDATA、NFSTAT和其他的ECC有关的寄存器。对NAND Flash控制器的操作，S3C2410和S3C2440有一点小差别：有些寄存器的地址不一样，有些寄存器的内容不一样，这在实例程序中会体现出来。

　　NAND Flash的读写操作次序如下：
1. 设置NFCONF（对于S3C2440，还要设置NFCONT）寄存器，配置NAND Flash
2. 向NFCMD寄存器写入命令，这些命令字可参考表
3. 向NFADDR寄存器写入地址
4. 读/写数据：通过寄存器NFSTAT检测NAND Flash的状态，在启动某个操作后，应该检测R/nB信号以确定该操作是否完成、是否成功。

### 1.4.2 寄存器介绍

　　下面讲解这些寄存器的功能及具体用法。

**NFCONF：** NAND Flash配置寄存器   

　　这个寄存器在S3C2410、S3C2440上功能有所不同。   
1. S3C2410的NFCONF寄存器   
　　被用来使能/禁止NAND Flash控制器、使能/禁止控制引脚信号nFCE、初始化ECC、设置NAND Flash的时序参数等。   
　　TACLS、TWRPH0和TWRPH1这三个参数控制的时NAND Flash信号线CLE/ALE与写控制信号nWE的时序关系如下：
![avatar](/img/in-post/Linux/201933001009.png)
2. S3C24402的NFCONF寄存器   
　　被用来设置NAND Flash的时序参数TACLS、TWRPH0、TWRPH1，设置数据位宽；还有一些只读位，用来指示是否支持其他大小的页（比如一页大小为 256/512/1024/2048 字节）。   
　　它没有实现S3C2410的NFCONF寄存器的控制功能，这些功能在S3C2440的NFCONT寄存器里实现。

**NFCONT：** NAND Flash 控制寄存器，S3C2410没有这个寄存器   
　　被用来使能/禁止NAND Flash控制器、使能/禁止控制引脚信号nFCE、初始化ECC。它还有其他功能，在一般的应用中用不到，比如锁定NAND Flash。

**NFCMD：** NAND Flash命令寄存器      
　　对于不同型号的Flash，操作命令一般不一样。

**NFADDR：** NAND Flash地址寄存器   
　　只用到低８位，读、写此寄存器将启动对NAND Flash的读数据、写数据操作。

**NFSTAT：** NAND Flash状态寄存器   
　　只用到位0，0：busy，1：ready。

# 2. NAND Flash 控制器操作实例：读Flash

　　本实例讲述如何读取NAND Flash，擦除、写Flash的操作与读Flash类似。

## 2.1 读NAND Flash的步骤

　　下面讲述如何从NAND Flash中读出数据，假设读地址为addr。

### 2.1.1 设置NFCONF(对于S3C2440,还要设置NFCONT)
　　**对于S3C2410**   
 　　在本章实例中设置为0x9830——使能NAND Flash控制器、初始化ECC、NAND Flash片选信号 nFCE = 1（inactive，真正使用时再让它等于0），设置TACLS = 0， TWRPH0 = 3，TWRPH1 = 0。这些
 时序参数的含义为：TACLS = 1个HCLK时钟，TWRPH0 = 4个HCLK时钟，TWRPH1 = 1个HCLK时钟。
 　　K9F1208U0M的实践特性如下：
```c 
CLE setup Time = 0ns, CLE Hold Time = 10ns,
ALE setup Time = 0ns, ALE Hold Time = 10ns,
WE Pulse Width = 25ns
```
　　可以计算：即使在HCLK=100MHz的情况下，TACLS + TWRPH0 + TWRPH1 = 6/100uS = 60ns，也是可以瞒住NAND Flash K9F1208U0M的时序要求的。

　　**对于S3C2440**   
　　时间参数也设为：TACLS = 0，TWRPH0 = 3，TWRPH1 = 0。NFCONF寄存器的值如下：   
　　NFCONF = 0x300   
　　NFCONF寄存器的取值如下，表示使能NAND Flash控制器、禁止控制引脚信号nFCE、初始化ECC。
```c
　　NFCONT = （1<<4) | (1<<1) | (1<<0)
```

### 2.1.2 在第一次操作NAND Flash前，通常复位一下 NAND Flash

　　**对于S3C2410**   
```c 
NFCONF &= ~(1<<11) (发出片选信号）
NFCMD = 0xff （reset命令）
```
　　然后循环查询NFSTAT位0，知道它等于1.
　　最后禁止片选信号，在实际使用NAND Flash时再使能。
```c
NFCONT |= (1<<11) (禁止NAND Flash)
```

　　**对于S3C2440**   
```c 
NFCONF &= ~(1<<11) (发出片选信号）
NFCMD = 0xff （reset命令）
```
　　然后循环查询NFSTAT位0，知道它等于1.
　　最后禁止片选信号，在实际使用NAND Flash时再使能。
```c
NFCONT |= 0x2 (禁止NAND Flash)
```

### 2.1.3 发出读命令
　　先使能NAND Flash，然后发出读命令。   
　　**对于S3C2410**   
```c 
NFCONF &= ~(1<<11) (发出片选信号）
NFCMD = 0 （读命令）
```

　　**对于S3C2440**   
```c 
NFCONF &= ~(1<<11) (发出片选信号）
NFCMD = 0 （读命令）
```

### 2.1.4 发出地址信号

　　这步请注意，表中列出了在地址操作的４个步骤对应的地址线，没有用到$A_8$（它由读命令设置，当读命令为0时，$A_8$=0;当读命令为1时，$A_8$=1），如下：
```c
NFADDR = addr & 0xff
NFADDR = (addr>>9) & 0xff	(左移9位，不是8位）
NFADDR = (addr>>17) & 0xff	(左移17位，不是16位）
NFADDR = (addr>>25) & 0xff	(左移25位，不是24位）
```

### 2.1.5 循环查询NFSTAT位0，直到它等于1，这时可以读取数据了

### 2.1.6 连续读NFDATA寄存器512次，得到一页数据（512字节）

　　循环执行第3、4 、5、 6这4个步骤，直到读出所要求的所有数据。

### 2.1.7 最后，禁止NAND Flash的片选信号
o
　　**对于S3C2410**   
```c 
NFCONF |= (1<<11)
```

　　**对于S3C2440**   
```c 
NFCONF |= (1<<11) 
```

## 2.2 代码详解

　　实验代码用到的文件有head.S、init.c和main.c。本实例的目的是把一部分代码存放在NAND Flash地址4096之后，当程序启动后通过NAND Flash控制器将他们读出来、执行。以前的代码都小于
4096字节，开发板启动后他们被自动复制进“Steppingstone”中。

　　连接脚本nand.lds把他们分为两部分，nand.lds代码如下：
```makefile
first 0x00000000 : {head.o init.o nand.o}

second 0x300000000 : AT(4096) {MAIN.O}
}
```
 　　第二行表示head.o、init.o、nand.o这3个文件的运行地址为0，他们在生成的映像文件中的偏移地址也为0（从0开始存放）。

 　　第三行表示main.o的运行地址为0x30000000，它在生成的映像文件中的偏移地址我诶诶诶4096.

 　　head.S调用init.c中的函数来关WATCH DOG、初始化SDRAM；调用nand.c中的函数来初始化NAND Flash，然后将main.c中的代码从NAND Flash地址4096开始处复制到SDRAM中；最后跳到
 main.c中的main函数继续执行。

 　　由于S3C2410、S3C2440的NAND FLASH控制器并非而完全一样，这个程序要既能处理S3C2410，也能处理S3C3440，然后使用不同的函数进行处理。读取GSTATUS1寄存器，如果它的值为0x32410000或者
 0x32410002，就表示处理器是S3C2410，否则是S3C2440。

 　　nand.c向外引出两个函数：用来初始化NAND Flash的nand_init函数、用来将数据从NAND Flash读得到SDRAM的nand_read函数。

### 2.2.1 nand_init函数分析

　　代码如下：
```c
/* 初始化 NAND Flash */
void nand_init(void_)
{
#define TACLS 0
#define TWRPH0 3
#define TWRPH1 0

	if ((GSTATUS1 == 0x32410000) || (GSTATUS1 == 0X3241002)){
		nand_chip.nand_reeset = s3c2410_nand_reset;
		nand_chip.wait_idle = s3c2410_wait_idle;
		nand_chip.nand_select_chip = s3c2410_nand_select_chip;
		nand_chip.nand_deselect_chip = s3c2410_nand_deselect_chip;
		nand_chip.write_cmd = s3c2410_wirte_cmd;
		nand_chip.write_addr = s3c2410_write_addr;
		nand_chip.read_data = s3c2410_read_data;

		s3c2410nand->NFCONF = (1<<15)|(1<<12)|(1<<11)|(TACLS<<8)|(TWRPH0<<4)|(TWRPH1<<0);
	}
	else{
		nand_chip.nand_reeset = s3c2440_nand_reset;
		nand_chip.wait_idle = s3c2440_wait_idle;
		nand_chip.nand_select_chip = s3c2440_nand_select_chip;
		nand_chip.nand_deselect_chip = s3c2440_nand_deselect_chip;
		nand_chip.write_cmd = s3c2440_wirte_cmd;
		nand_chip.write_addr = s3c2440_write_addr;
		nand_chip.read_data = s3c2440_read_data;

		s3c2440nand->NFCONF = (TACLS<<12)|(TWRPH0<<8)|(TWRPH1<<4);


		s3c2440nand->NFCONF = (1<<4)|(1<<1)|(1<<0);
		}

	nand_reset();
	
}
```

### 2.2.2 nand_read函数分析

　　它的原型如下，表示从NAND Flash位置start_addr开始，将数据复制到SDRAM地址buf处，共复制size字节。
```c
int
nand_read(unsigned char *buf, unsigned long start_addr, int size)
{
	int i, j;

	if ((start_addr & NAND_BLOCK_MASK) || (size & NAND_BLOCK_MASK)) {
		return -1;	/* invalid alignment */
	}

	nand_select_chip();

	for(i=start_addr; i < (start_addr + size);) {
		/* READ0 */
		write_cmd(0);

		/* Write Address */
		write_addr(i);
		wait_idle();

		NAND_DETECT_RB;

		for(j=0; j < NAND_SECTOR_SIZE; j++, i++) {
			*buf = (NFDATA & 0xff);
			buf++;
		}
	}
	nand_deselect_chip();
	return 0;
}

　　可以看到，读NAND Flash的操作分为6步：   
1. 选择芯片
2. 发出读命令
3. 发出地址
4. 等待数据就绪
5. 读取数据
6. 结束后，取消片选信号





　
