---
layout: post
title: "Uboot-Nandflash进阶＜二＞"
subtitle: Uboot-Nandflash进阶分析
author: "404"
header-style: text
tags:
  - Linux
  - Arm
  - Uboot
  - NANDflash
---
>本篇文章转载自[【详解】如何编写Linux下Nand Flash驱动](https://www.crifan.com/files/doc/docbook/linux_nand_driver/release/html/linux_nand_driver.html),写的挺好

# 2. 硬件特性

## 2.9 Nand Flash引脚(Pin)的说明
![avatar](/img/in-post/Linux/201933101006.png)
　　上图是常见的Nand Flash所拥有的引脚（Pin）所对应的功能，简单翻译如下：
![avatar](/img/in-post/Linux/201933101007.png)
　　在数据手册中，你常会看到，对于一个引脚定义，有些字母上面带一横杠的，那是说明此引脚/信号是低电平有效，比如你上面看到的RE头上有个横线，就是说明，此RE是低电平有效，此外，为了书写方便，在字母后面加“＃”，也是表示低电平有效，比如我上面写的CE＃；如果字母头上啥都没有，就是默认的高电平有效，比如上面的CLE，就是高电平有效。
