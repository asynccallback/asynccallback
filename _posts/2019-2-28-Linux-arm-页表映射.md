---
layout: post
title: "Linux-arm-页表映射"
subtitle: '分析arm页表映射过程'
author: "404"
header-style: text
tags:
  - Linux
  - arm
  - Memory
---

>此篇文章为原创，转载请注明出处。

上篇文章中我们知道了`x86`体系结构下页表映射，现在我们来具体了解下`ARM`体系结构下的页表映射。

在32bit的Linux内核中一般采用3层的映射模型，分别为`PGD`、`PMD`、`PTE`,在arm32系统结构中，一般使用两层映射，即`PGD`、`PTE`,如下图所示：
![avatar](/img/in-post/Linux/201922802001.png)
其中PGD(bit[31:20])有12位，PTE(bit[19:12])有8位，其寻址过程跟`X86`一样。这个过程是在`ARM32`架构中的MMU中实现的。
