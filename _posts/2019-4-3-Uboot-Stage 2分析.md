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

　　Uboot的第二阶段就是要初始化第一阶段剩下的还没被初始化的硬件。主要是SoC外部硬件（譬如iNand、网卡芯片····）、Uboot本身的一些东西（Uboot的命令、环境变量等）。然后最终初始化完必要的东西后进入Uboot的命令行准备接受命令。

　　那么Uboot何时结束？如下：
1. Uboot启动后自动运行打印出很多信息（这些信息就是Uboot在第一和第二阶段不断进行初始化时，打印出来的信息）。然后Uboot进入了倒数bootdelay秒然后执行bootcmd对应的启动命令。
2. 如果用户没有干涉则会执行bootcmd进入自动启动内核流程（Uboot就死掉了）；此时用户可以按下回车键打断Uboot的自动启动进入Uboot的命令行下。然后Uboot就一直工作在命令行下。
3. Uboot的命令行就是一个死循环，循环体内不断重复：接收命令、解析命令、执行命令。这就是Uboot最终的归宿。
