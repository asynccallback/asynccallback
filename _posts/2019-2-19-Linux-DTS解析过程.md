---
layout: post
title: "Linux-DTS解析过程"
subtitle: 'DTS解析过程'
author: "404"
header-style: text
tags:
  - Linux
  - DTS
---

在分析DTS解析前，我们先假设系统中的DTS设备树已经生成，先不关心系统如何生成DTS设备树。

---
#### 基础数据结构
---
设备树是由节点和属性组成的的简单树形结构。属性是键值对，节点可以同时包含属性和子节点。以下是.dts格式的简单树：
```
/dts-v1/;

/ {
    node1 {
        a-string-property = "A string";
        a-string-list-property = "first string", "second string";
        // hex is implied in byte arrays. no '0x' prefix is required
        a-byte-data-property = [01 23 34 56];
        child-node1 {
            first-child-property;
            second-child-property = <1>;
            a-string-property = "Hello, world";
        };
        child-node2 {
        };
    };
    node2 {
        an-empty-property;
        a-cell-property = <1 2 3 4>; /* each number (cell) is a uint32 */
        child-node1 {
        };
    };
};
```
由于这颗设备树不描述任何东西，它没有任何价值，但它确实显示了节点和属性的结构。如下：   
- 一个根节点: \"/\"
- 一对子节点\"node1\" 和\"node2\"
- 子节点node1还有一对子节点中的子节点：\"child-node1\" 和\"child-node2\"
- 树形结构上的一堆属性

树形结构中的属性是简单的键值对，属性值可以为空，也可以是字节。虽然数据类型未编码到数据结构中，但有一些基本数据类型表示形式可以在设备树源文件中表示。
- 被双引号包含的文本字符串：
  - string-property = \"a string\";
- “cells”代表的是32位无符号整数数据，赋值的时候用“< >”
  - cell-property = <0xbeef 123 0xabcd1234>;
- 被“[ ]”包括的二进制数据
  - binary-property= [0x01 0x23 0x45 0x67];
- 混合数据类型，不同数据类型用逗号隔开：
 - mixed-property = \"a string\", [0x01 0x23 0x45 0x67], <0x12345678>;
- 逗号可以当做多个字符串的分隔符,用作字符数组：
 - string-list = \"red fish\", \"blue fish\";

#### 基础概念
---
为了更好理解设备树如何使用，我们选择一台简单的设备并为其建立设备树来描述他。  
##### 简单设备
现在有个arm平台的板子，假设制造商为“acme”我们给他命名为“Coyote's Revenge”首先，假设我们有如下的硬件平台  
- 一个32位ARM CPU  
- 处理器本地总线上映射了串口SPI控制器、I2C控制器、中断控制器以及外部总线桥  
- 基地址为0，大小为256MB的SDRAM  
- 基地址分别为0x101F1000 和0x101F2000的2个串口  
- 基地址为0x101F3000 GPIO控制器  
- 基地址为0x10170000的SPI 控制器上有如下的设备：
  - MMC 卡槽的SS引脚连接到了GPIO \#1  
- 外部总线桥上有如下的设备：
  - SMC公司生产的SMC91111 Ethernet，其基地址为0x10100000。
-  在基地址为0x10160000的 i2c 控制器有如下设备：  
   - Maxim公司的DS1338 实时时钟芯片.  slave address地址为0x58
   - 大小为64MB 的 NOR flash 其基地址为0x30000000     

##### 初始化树形结构
首先给设备树一个大致的结构框架。这个最简单的结构框架是设备树必须的，在此阶段，我们可以唯一的标识此机器。
```
/dts-v1/;

/ {
    compatible = "acme,coyotes-revenge";
};
```  
`compatible`唯一的确定了该设备的名称。我们通过使用\"\<制造商\>，\<模式\>\"。`compatible`对设备十分重要，可以唯一确定改设备，同时为了确保唯一性，我们加入该设备的制造商来避免命名名称冲突，可以参考编程语言库中的namespace。既然操作系统将会使用`compatible`来在设备树中寻找该设备，进而读取相应数据到操作系统中，所以一定要确保`compatible`值的正确性以及唯一性。　　
理论上，操作系统只能通过`compatible`来唯一确定一台设备。如果所有的设备细节都被硬编码，那么操作系统就可以在顶层的`compatible`属性中通过匹配`acme,coyotes-revenge `来找到该设备。

##### CPUs
下一步将来描述设备中的各个CPU。众多子CPU节点会被加入到一个\＂cpus\＂的容器节点中。在这个例子中，我们将会使用一块多核Cortex A9 的ARM开发板。  
```
/dts-v1/;

/ {
    compatible = "acme,coyotes-revenge";

    cpus {
        cpu@0 {
            compatible = "arm,cortex-a9";
        };
        cpu@1 {
            compatible = "arm,cortex-a9";
        };
    };
};
```  
在CPUs子节点中，各个cpu的`compatible`值（`<manufacturer>,<model>`）将会唯一确定该cpu。这个和上面姐扫的设备`compatible`一样。  
更多的属性值将会被加入到各个cpu节点中，不过我们现在还是先来谈谈更多的基础概念。  
##### 节点名称
为了更好的理解设备树，我们值得多花一点时间来讨论命名规则。每个节点名称的命名规则是`<name>[@<unit-address> `。  
`<name> ` 是一个简单的ASCII字符串，其长度不能超过31个字节。一般来说，节点名称能体现这个节点的用途。例如：一个3com Ethernet adapteer将被命名为etherneet，而不是3com509。  
`<unit-address>`描述一个设备是否需要设置访问地址。一般来说，`<unit-address>`描述的访问该设备的基地址，同时该地址也会在`reg`属性列出，稍后我们会讲解`reg`属性的设置。  
同一层次下的节点命名必须是唯一的，但是只要基地址不同，同一层次也可以存在相同的命名节点，比如：serial@101f1000 & serial@101f2000。  
如果想了解更多的节点命名规则，请参考ePAPR中2.2.1这部分。  
##### 设备
系统中的每个设备都将在设备树中以一个节点来体现。  
下一步我们将用每个设备的节点填充设备树。不过现在，新节点将保持为空，直到我们可以讨论如何处理地址范围和IRQ。  
```
/dts-v1/;

/ {
    compatible = "acme,coyotes-revenge";

    cpus {
        cpu@0 {
            compatible = "arm,cortex-a9";
        };
        cpu@1 {
            compatible = "arm,cortex-a9";
        };
    };

    serial@101F0000 {
        compatible = "arm,pl011";
    };

    serial@101F2000 {
        compatible = "arm,pl011";
    };

    gpio@101F3000 {
        compatible = "arm,pl061";
    };

    interrupt-controller@10140000 {
        compatible = "arm,pl190";
    };

    spi@10115000 {
        compatible = "arm,pl022";
    };

    external-bus {
        ethernet@0,0 {
            compatible = "smc,smc91c111";
        };

        i2c@1,0 {
            compatible = "acme,a1234-i2c-bus";
            rtc@58 {
                compatible = "maxim,ds1338";
            };
        };

        flash@2,0 {
            compatible = "samsung,k8f1315ebm", "cfi-flash";
        };
    };
};
```  
通过上面的设备数中，我们可以看出来，通过dts的层次关系，可以看到具体的板级的设备连接关系，比如外部总线上有ethercat、i2c和flash三个设备。上面的视图也就可以理解为cpu看到的设备关系。  
但上面的设备树并不是一个有效的设备结构图，它还缺少各个设备间的连接关系，这部分我们将在稍后加上。  
我们从上面的设备树中注意到以下几点：  
- 每个设备节点都包含`compatible`属性。
- flash节点的`compatible`属性中包含两个字符串，这个我们将在下面部分解释原因。

##### 理解`compatible`属性
设备数中每个节点都有`compatible`属性，操作系统通过`compatible`属性来决定设备驱动与设备的匹配。  
`compatible`属性由字符串数组组成，字符串数组中第一个字符串由`<manufacturer>,<model>`格式组成，来确定设备的唯一性；剩下的
字符串来描述的与该节点描述符相兼容的设备。  
举个例子，飞思卡尔MPC8349板上具有一个串行设备，实现了国家半导体NS16550寄存器接口标准。那么MPC8349的串行设备在设备数上`compatible`属性值为：`compatible = "fsl,mpc8349-uart", "ns16550"`。在这个例子中\"fsl,mpc8349-uart\"确定了改设备，
\"ns16550\"标明它是与国家半导体16550 UART兼容的寄存器级。  
这种做法允许将现有设备驱动程序绑定到较新的设备，同时仍然唯一地标识确切的硬件。  

剩下见后续[Linux-DST Part Ⅱ]()
