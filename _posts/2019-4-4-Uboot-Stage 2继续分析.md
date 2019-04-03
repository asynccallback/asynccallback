---
layout: post
title: "Uboot-Stage 2继续分析"
subtitle: Uboot-Stage 2继续分析
author: "404"
header-style: text
tags:
  - Linux
  - Arm
  - Uboot
---
## 2.3 Norflash 初始化
```c
#ifndef CFG_NO_FLASH
	/* configure available FLASH banks */
	size = flash_init ();
	display_flash_config (size);
#endif /* CFG_NO_FLASH */

unsigned long flash_init(void)
{
    unsigned long size = 0;
    int i;

    /* Init: no FLASHes known */
    /* #define CONFIG_SYS_MAX_FLASH_BANKS    1 */
    /* include/configs/jz2440.h中有定义，为 1 */
    for (i = 0; i < CONFIG_SYS_MAX_FLASH_BANKS; ++i) {
        flash_info[i].flash_id = FLASH_UNKNOWN;

        /* Optionally write flash configuration register */
        cfi_flash_set_config_reg(cfi_flash_bank_addr(i),
                     cfi_flash_config_reg(i));

        /* 检测 flash
         * flash_detect_legacy 是旧的检测策略
         */
        if (!flash_detect_legacy(cfi_flash_bank_addr(i), i))
            flash_get_size(cfi_flash_bank_addr(i), i);
        size += flash_info[i].size;
        if (flash_info[i].flash_id == FLASH_UNKNOWN) {
#ifndef CONFIG_SYS_FLASH_QUIET_TEST
            printf("## Unknown flash on Bank %d ", i + 1);
            printf("- Size = 0x%08lx = %ld MB\n",
                   flash_info[i].size,
                   flash_info[i].size >> 20);
#endif /* CONFIG_SYS_FLASH_QUIET_TEST */
        }
    }

    flash_protect_default();    //flash的默认保护

    return (size);
}
```
　　这里会通过使用应用芯片的读写手册，读出Norflash相应的大小。

## 2.4 Nandflash 初始化
```c
#if (CONFIG_COMMANDS & CFG_CMD_NAND)
	puts ("NAND:  ");
	nand_init();		/* go init the NAND */
#endif
```
　　这里同理NorFlash

## 2.5 环境变量重定位
```c
env_relocate ();

void env_relocate (void)
{
	DEBUGF ("%s[%d] offset = 0x%lx\n", __FUNCTION__,__LINE__,
		gd->reloc_off);

#ifdef CONFIG_AMIGAONEG3SE
	enable_nvram();
#endif

#ifdef ENV_IS_EMBEDDED
	/*
	 * The environment buffer is embedded with the text segment,
	 * just relocate the environment pointer
	 */
	env_ptr = (env_t *)((ulong)env_ptr + gd->reloc_off);
	DEBUGF ("%s[%d] embedded ENV at %p\n", __FUNCTION__,__LINE__,env_ptr);
#else
	/*
	 * We must allocate a buffer for the environment，在sdram中开辟一开内存空间并使env_ptr指向它
	 */
	env_ptr = (env_t *)malloc (CFG_ENV_SIZE);
	DEBUGF ("%s[%d] malloced ENV at %p\n", __FUNCTION__,__LINE__,env_ptr);
#endif

	/*
	 * After relocation to RAM, we can always use the "memory" functions
	 */
	env_get_char = env_get_char_memory;

	if (gd->env_valid == 0) {
#if defined(CONFIG_GTH)	|| defined(CFG_ENV_IS_NOWHERE)	/* Environment not changable */
		puts ("Using default environment\n\n");
#else
		puts ("*** Warning - bad CRC, using default environment\n\n");
		SHOW_BOOT_PROGRESS (-1);
#endif

		if (sizeof(default_environment) > ENV_SIZE)
		{
			puts ("*** Error - default environment is too large\n\n");
			return;
		}

		memset (env_ptr, 0, sizeof(env_t));
		memcpy (env_ptr->data,
			default_environment,
			sizeof(default_environment));
#ifdef CFG_REDUNDAND_ENVIRONMENT
		env_ptr->flags = 0xFF;
#endif
		env_crc_update ();
		gd->env_valid = 1;
	}
	else {
    //如果，使用的非默认的env，也就是 nor中有，将norflash中的env拷贝到sdram
		env_relocate_spec ();
	}
	gd->env_addr = (ulong)&(env_ptr->data);

#ifdef CONFIG_AMIGAONEG3SE
	disable_nvram();
#endif
}

void env_relocate_spec (void)
{
//注意次函数从env_relocate()中调用时，env_ptr已经改变为新在内存中开辟的地址
//因此次函数功能：将Nor中的env，拷贝到sdram
  memcpy (env_ptr, (void*)flash_addr, CFG_ENV_SIZE);
}
```

## 2.6 初始化gd结构 ip mac

## 2.7 devices_init ()

## 2.8 jumptable_init ()

## 2.9 enable_interrupts 在 cpsr 中使能 irq
```c
/* enable exceptions */
enable_interrupts ();
```

## 2.10 网卡初始化
```c
#if defined(CONFIG_NET_MULTI)
	puts ("Net:   ");
#endif
	eth_initialize(gd->bd);
```

## 2.11 跳转到 uboot 菜单
```c
/* main_loop() can return to retry autoboot, if so just run it again. */
for (;;) {
  main_loop ();
}
```
