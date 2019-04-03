---
layout: post
title: "Uboot-Stage 2最后阶段"
subtitle: Uboot-Stage 2最后阶段
author: "404"
header-style: text
tags:
  - Linux
  - Arm
  - Uboot
---
# 3. 在main_loop()中循环

```c
　　我们在Uboot中输入`bootm`命令，Uboot接收输入的字符串“bootm”，传递给`run_command`函数。`run_command`函数调用 `common/command.c`中实现的`find_cmd`函数在`__u_boot_cmd_start`与`__u_boot_cmd_end`间查找命令，并返回bootm命令的`cmd_tbl_t`结构。然后`run_command`函数使用返回的`cmd_tbl_t`结构中的函数指针调用bootm 令的响应函数`do_bootm`
int do_bootm (cmd_tbl_t *cmdtp, int flag, int argc, char *argv[])
{
	ulong	iflag;
	ulong	addr;
	ulong	data, len, checksum;
	ulong  *len_ptr;
	uint	unc_len = CFG_BOOTM_LEN;
	int	i, verify;
	char	*name, *s;
	int	(*appl)(int, char *[]);
	image_header_t *hdr = &header;

	s = getenv ("verify");
	verify = (s && (*s == 'n')) ? 0 : 1;

	if (argc < 2) {
		addr = load_addr;
	} else {
		addr = simple_strtoul(argv[1], NULL, 16);
	}

	SHOW_BOOT_PROGRESS (1);
	printf ("## Booting image at %08lx ...\n", addr);

	/* Copy header so we can blank CRC field for re-calculation */
#ifdef CONFIG_HAS_DATAFLASH
	if (addr_dataflash(addr)){
		read_dataflash(addr, sizeof(image_header_t), (char *)&header);
	} else
#endif
	memmove (&header, (char *)addr, sizeof(image_header_t));

	if (ntohl(hdr->ih_magic) != IH_MAGIC) {
#ifdef __I386__	/* correct image format not implemented yet - fake it */
		if (fake_header(hdr, (void*)addr, -1) != NULL) {
			/* to compensate for the addition below */
			addr -= sizeof(image_header_t);
			/* turnof verify,
			 * fake_header() does not fake the data crc
			 */
			verify = 0;
		} else
#endif	/* __I386__ */
	    {
		puts ("Bad Magic Number\n");
		SHOW_BOOT_PROGRESS (-1);
		return 1;
	    }
	}
	SHOW_BOOT_PROGRESS (2);

	data = (ulong)&header;
	len  = sizeof(image_header_t);

	checksum = ntohl(hdr->ih_hcrc);
	hdr->ih_hcrc = 0;

	if (crc32 (0, (uchar *)data, len) != checksum) {
		puts ("Bad Header Checksum\n");
		SHOW_BOOT_PROGRESS (-2);
		return 1;
	}
	SHOW_BOOT_PROGRESS (3);

#ifdef CONFIG_HAS_DATAFLASH
	if (addr_dataflash(addr)){
		len  = ntohl(hdr->ih_size) + sizeof(image_header_t);
		read_dataflash(addr, len, (char *)CFG_LOAD_ADDR);
		addr = CFG_LOAD_ADDR;
	}
#endif


	/* for multi-file images we need the data part, too */
	print_image_hdr ((image_header_t *)addr);

	data = addr + sizeof(image_header_t);
	len  = ntohl(hdr->ih_size);

	if (verify) {
		puts ("   Verifying Checksum ... ");
		if (crc32 (0, (uchar *)data, len) != ntohl(hdr->ih_dcrc)) {
			printf ("Bad Data CRC\n");
			SHOW_BOOT_PROGRESS (-3);
			return 1;
		}
		puts ("OK\n");
	}
	SHOW_BOOT_PROGRESS (4);

	len_ptr = (ulong *)data;

#if defined(__PPC__)
	if (hdr->ih_arch != IH_CPU_PPC)
#elif defined(__ARM__)
	if (hdr->ih_arch != IH_CPU_ARM)
#elif defined(__I386__)
	if (hdr->ih_arch != IH_CPU_I386)
#elif defined(__mips__)
	if (hdr->ih_arch != IH_CPU_MIPS)
#elif defined(__nios__)
	if (hdr->ih_arch != IH_CPU_NIOS)
#elif defined(__M68K__)
	if (hdr->ih_arch != IH_CPU_M68K)
#elif defined(__microblaze__)
	if (hdr->ih_arch != IH_CPU_MICROBLAZE)
#elif defined(__nios2__)
	if (hdr->ih_arch != IH_CPU_NIOS2)
#elif defined(__blackfin__)
	if (hdr->ih_arch != IH_CPU_BLACKFIN)
#elif defined(__avr32__)
	if (hdr->ih_arch != IH_CPU_AVR32)
#else
# error Unknown CPU type
#endif
	{
		printf ("Unsupported Architecture 0x%x\n", hdr->ih_arch);
		SHOW_BOOT_PROGRESS (-4);
		return 1;
	}
	SHOW_BOOT_PROGRESS (5);

	switch (hdr->ih_type) {
	case IH_TYPE_STANDALONE:
		name = "Standalone Application";
		/* A second argument overwrites the load address */
		if (argc > 2) {
			hdr->ih_load = htonl(simple_strtoul(argv[2], NULL, 16));
		}
		break;
	case IH_TYPE_KERNEL:
		name = "Kernel Image";
		break;
	case IH_TYPE_MULTI:
		name = "Multi-File Image";
		len  = ntohl(len_ptr[0]);
		/* OS kernel is always the first image */
		data += 8; /* kernel_len + terminator */
		for (i=1; len_ptr[i]; ++i)
			data += 4;
		break;
	default: printf ("Wrong Image Type for %s command\n", cmdtp->name);
		SHOW_BOOT_PROGRESS (-5);
		return 1;
	}
	SHOW_BOOT_PROGRESS (6);

	/*
	 * We have reached the point of no return: we are going to
	 * overwrite all exception vector code, so we cannot easily
	 * recover from any failures any more...
	 */

	iflag = disable_interrupts();

#ifdef CONFIG_AMIGAONEG3SE
	/*
	 * We've possible left the caches enabled during
	 * bios emulation, so turn them off again
	 */
	icache_disable();
	invalidate_l1_instruction_cache();
	flush_data_cache();
	dcache_disable();
#endif

	switch (hdr->ih_comp) {
	case IH_COMP_NONE:
		if(ntohl(hdr->ih_load) == addr) {
			printf ("   XIP %s ... ", name);
		} else {
#if defined(CONFIG_HW_WATCHDOG) || defined(CONFIG_WATCHDOG)
			size_t l = len;
			void *to = (void *)ntohl(hdr->ih_load);
			void *from = (void *)data;

			printf ("   Loading %s ... ", name);

			while (l > 0) {
				size_t tail = (l > CHUNKSZ) ? CHUNKSZ : l;
				WATCHDOG_RESET();
				memmove (to, from, tail);
				to += tail;
				from += tail;
				l -= tail;
			}
#else	/* !(CONFIG_HW_WATCHDOG || CONFIG_WATCHDOG) */
			memmove ((void *) ntohl(hdr->ih_load), (uchar *)data, len);
#endif	/* CONFIG_HW_WATCHDOG || CONFIG_WATCHDOG */
		}
		break;
	case IH_COMP_GZIP:
		printf ("   Uncompressing %s ... ", name);
		if (gunzip ((void *)ntohl(hdr->ih_load), unc_len,
			    (uchar *)data, &len) != 0) {
			puts ("GUNZIP ERROR - must RESET board to recover\n");
			SHOW_BOOT_PROGRESS (-6);
			do_reset (cmdtp, flag, argc, argv);
		}
		break;
#ifdef CONFIG_BZIP2
	case IH_COMP_BZIP2:
		printf ("   Uncompressing %s ... ", name);
		/*
		 * If we've got less than 4 MB of malloc() space,
		 * use slower decompression algorithm which requires
		 * at most 2300 KB of memory.
		 */
		i = BZ2_bzBuffToBuffDecompress ((char*)ntohl(hdr->ih_load),
						&unc_len, (char *)data, len,
						CFG_MALLOC_LEN < (4096 * 1024), 0);
		if (i != BZ_OK) {
			printf ("BUNZIP2 ERROR %d - must RESET board to recover\n", i);
			SHOW_BOOT_PROGRESS (-6);
			udelay(100000);
			do_reset (cmdtp, flag, argc, argv);
		}
		break;
#endif /* CONFIG_BZIP2 */
	default:
		if (iflag)
			enable_interrupts();
		printf ("Unimplemented compression type %d\n", hdr->ih_comp);
		SHOW_BOOT_PROGRESS (-7);
		return 1;
	}
	puts ("OK\n");
	SHOW_BOOT_PROGRESS (7);

	switch (hdr->ih_type) {
	case IH_TYPE_STANDALONE:
		if (iflag)
			enable_interrupts();

		/* load (and uncompress), but don't start if "autostart"
		 * is set to "no"
		 */
		if (((s = getenv("autostart")) != NULL) && (strcmp(s,"no") == 0)) {
			char buf[32];
			sprintf(buf, "%lX", len);
			setenv("filesize", buf);
			return 0;
		}
		appl = (int (*)(int, char *[]))ntohl(hdr->ih_ep);
		(*appl)(argc-1, &argv[1]);
		return 0;
	case IH_TYPE_KERNEL:
	case IH_TYPE_MULTI:
		/* handled below */
		break;
	default:
		if (iflag)
			enable_interrupts();
		printf ("Can't boot image type %d\n", hdr->ih_type);
		SHOW_BOOT_PROGRESS (-8);
		return 1;
	}
	SHOW_BOOT_PROGRESS (8);

	switch (hdr->ih_os) {
	default:			/* handled by (original) Linux case */
	case IH_OS_LINUX:
#ifdef CONFIG_SILENT_CONSOLE
	    fixup_silent_linux();
#endif
	    do_bootm_linux  (cmdtp, flag, argc, argv,
			     addr, len_ptr, verify);
	    break;
	case IH_OS_NETBSD:
	    do_bootm_netbsd (cmdtp, flag, argc, argv,
			     addr, len_ptr, verify);
	    break;

#ifdef CONFIG_LYNXKDI
	case IH_OS_LYNXOS:
	    do_bootm_lynxkdi (cmdtp, flag, argc, argv,
			     addr, len_ptr, verify);
	    break;
#endif

	case IH_OS_RTEMS:
	    do_bootm_rtems (cmdtp, flag, argc, argv,
			     addr, len_ptr, verify);
	    break;

#if (CONFIG_COMMANDS & CFG_CMD_ELF)
	case IH_OS_VXWORKS:
	    do_bootm_vxworks (cmdtp, flag, argc, argv,
			      addr, len_ptr, verify);
	    break;
	case IH_OS_QNX:
	    do_bootm_qnxelf (cmdtp, flag, argc, argv,
			      addr, len_ptr, verify);
	    break;
#endif /* CFG_CMD_ELF */
#ifdef CONFIG_ARTOS
	case IH_OS_ARTOS:
	    do_bootm_artos  (cmdtp, flag, argc, argv,
			     addr, len_ptr, verify);
	    break;
#endif
	}

	SHOW_BOOT_PROGRESS (-9);
#ifdef DEBUG
	puts ("\n## Control returned to monitor - resetting...\n");
	do_reset (cmdtp, flag, argc, argv);
#endif
	return 1;
}
```

　　这里，主要的`if(ntohl(hdr->ih_load) == addr)`,判断“uimage头部里指定的加载地址”与bootm指定的加载地址是否相等，不相等则需要移动,判断的方式有两种：
1. 判断 uimage头部里指定的加载地址 == bootm指定的加载地址 (hdr->ih_load == addr)，此时：   
```
实际存放的地址 == uimage加载地址 == uimage连接地址-64字节
bootm == 实际存放的地址
```   
例如：   
```
实际存放在 0x30007fc0
bootm 0x30007fc0
加载地址 0x30007fc0
连接地址	0x30008000
```   
- uboot根据Bootm指定的0x30007fc0去找image，实际地址为0x30007fc0，找到头部
- 读出头部里的加载地址，判断是否和Bootm相等，相等则不移动，启动

2. 判断 uimage头部里指定的加载地址 == bootm指定的加载地址 + 64字节 (hdr->ih_load == addr+64字节)
此时：   
```
实际存放地址+64字节 == uimage加载地址 == uimage连接地址
bootm == 实际存放的地址
```   
例子：   
```
实际存放在 0x30007fc0
bootm 0x30007fc0
加载地址 0x30008000
连接地址	0x30008000
```   
- uboot根据Bootm指定的0x30007fc0去找image，实际地址为0x30007fc0，找到头部
- 读出头部里的加载地址，判断是否和Bootm + 字节相等，相等则不移动，启动

首先bootm的地址要和我们 实际存放(不管它的加载地址是多少) 整个uimage的首地址吻合，这样就可以找到头部。
这里存在两种情况，我们可以看到 Uboot源码里:
1. `hdr->ih_load == addr`  
也就是说判断的是`bootm_addr`与uimage里的`ih_load`加载地址是否相等，这样的话，我们在制作uimage的时候就应该让`ih_load=bootmaddr`。那么uimage里的另一个参数，连接地址就应该等于`bootm+4K`.   
2. `hdr->ih_load == addr+64字节`   
那么，uimage里的Load地址和连接地址应该相等，等于`bootm+64`字节

　　然后会调用`do_bootm_linux`,在`do_bootm_linux`调用`theKernel (0, bd->bi_arch_number, bd->bi_boot_params);`,至此至此，uboot 的生命周期结束，控制权交给内核
