---
layout: post
title: "Linux-arm-内存布局"
subtitle: "从内核启动代码开始分析"
author: "404"
header-style: text
tags:
  - Linux
  - arm
  - Memory
---


#### Linux内核生成
Linux内核的生成是最先从生成vmlinux开始的，过程如下:

1. 依据arch/arm/kernel/vmlinux.lds 生成linux内核源码根目录下的vmlinux，这个vmlinux属于未压缩，带调试信息、符号表的最初的内核，大小约23MB；依据arch/arm/kernel/vmlinux.lds 生成linux内核源码根目录下的vmlinux，这个vmlinux属于未压缩，带调试信息、符号表的最初的内核，大小约23MB；   
```	c
命令：arm-linux-gnu-ld -o vmlinux -T arch/arm/kernel/vmlinux.lds  
arch/arm/kernel/head.o  
init/built-in.o  
--start-group   
arch/arm/mach-s3c2410/built-in.o   
kernel/built-in.o          
mm/built-in.o   
fs/built-in.o   
ipc/built-in.o   
drivers/built-in.o   
net/built-in.o  
--end-group .tmp_kallsyms2.o 
```
2. 将上面的vmlinux去除调试信息、注释、符号表等内容，生成arch/arm/boot/Image，这是不带多余信息的linux内核，Image的大小约3.2MB； 
```c
命令:arm-linux-gnu-objcopy -O binary -S  vmlinux arch/arm/boot/Image 
``` 
3. 将 arch/arm/boot/Image 用gzip -9 压缩生成arch/arm/boot/compressed/piggy.gz大小约1.5MB；
```c
命令:gzip -f -9 < arch/arm/boot/compressed/../Image > arch/arm/boot/compressed/piggy.gz 
```
4. 编译arch/arm/boot/compressed/piggy.S 生成arch/arm/boot/compressed/piggy.o大小约1.5MB，这里实际上是将piggy.gz通过piggy.S编译进piggy.o文件中。而piggy.S文件仅有6行，只是包含了文件piggy.gz; 
```c
命令:arm-linux-gnu-gcc -o arch/arm/boot/compressed/piggy.o arch/arm/boot/compressed/piggy.S 
```


5. 依据arch/arm/boot/compressed/vmlinux.lds 将arch/arm/boot/compressed/目录下的文件head.o 、piggy.o 、misc.o链接生成 arch/arm/boot/compressed/vmlinux，这个vmlinux是经过压缩且含有自解压代码的内核,大小约1.5MB; 
```c
命令:arm-linux-gnu-ld zreladdr=0x60008000 params_phys=0x60000100 -T arch/arm/boot/compressed/vmlinux.lds arch/arm/boot/compressed/head.o arch/arm/boot/compressed/piggy.o arch/arm/boot/compressed/misc.o -o arch/arm/boot/compressed/vmlinux 
```


6. 将arch/arm/boot/compressed/vmlinux去除调试信息、注释、符号表等内容，生成arch/arm/boot/zImage大小约1.5MB;这已经是一个可以使用的linux内核映像文件了； 
```c
命令:arm-linux-gnu-objcopy -O binary -S  arch/arm/boot/compressed/vmlinux  arch/arm/boot/zImage 
```


7. 将arch/arm/boot/zImage添加64Bytes的相关信息打包为arch/arm/boot/uImage大小约1.5MB; 
```c
命令: ./mkimage -A arm -O linux -T kernel -C none -a 0x60008000 -e 0x60008000 -n 'Linux-2.6.35.7' -d arch/arm/boot/zImage arch/arm/boot/uImage
```

