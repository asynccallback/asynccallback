---
layout: post
title: "Linux-DTS Part 2"
subtitle: '承接Linux-DTS Part 1'
author: "404"
header-style: text
tags:
  - Linux
  - DTS
---


#### 设备如何寻址  
---
设备寻址地址在设备树中通过如下的属性信息来表示：  
- `reg`  
- `#address-cells`
- `#size-cells`

每一个可寻址设都会有一个reg属性，该属性有一个或多个元素组成，其基本格式为：
`reg = <address1 length1 [address2 length2] [address3 length3] ... >`
上面的每一个元素都代表设备的寻址地址及其寻址大小，每一个元素中的address值可以是一个或者多个无符号32位整形数据类型cell来表示，元素中的length可以为空也可以使一个或者多个无符号32位整形数据类型cell。
既然每个可寻址设备都会有reg属性可变的，而且reg属性元素也是灵活可选择的，那么谁来制定reg属性元素中每个元素也就是address和length的个数呢？
在这里，要关注到期父节点的两个属性，其中`#address-cells`表示reg中address元素的个数，`#size-cells`用来表示length元素的个数。
为了展示这些如何起作用，那么现在做一个Demo，首先从cpu节点开始演示：
##### CPU如何寻址
相对于其他节点的寻址地址，cpu节点的寻址地址是最简单的。之前介绍，而每个cpu节点都包含一个标记ID，但是没有其他的描述信息，这里填充一些其他的属性：
```
cpus {
    #address-cells = <1>;
    #size-cells = <0>;
    cpu@0 {
        compatible = "arm,cortex-a9";
        reg = <0>;
    };
    cpu@1 {
        compatible = "arm,cortex-a9";
        reg = <1>;
    };
};
```
在cpus节点中，`#address-cells` 赋值 1, `#size-cells` 赋值 0。这意味着在其子节点的reg只有一个地址元素值，没有长度元素值。在上述例子中，两颗cpu核的地址分别被分配成0和1。因为每个cpu只分配了地址，所以节点 #size-cells元素被设置为 0。
你肯定也注意到了`reg`属性值和cpu节点名称中的值相匹配`cpu@0`、`reg = <0>`,`cpu@1`、`reg = <1>`。在上述例子中，如果一个节点拥有`reg`属性，那么cpu节点中必须包含改属性节点的第一个值，即`#address-cells`。
##### 内存映射设备
需要内存映射的设备不同于cpu节点，这类的设备需要一段内存而不是单一的内存地址，因此不仅需要包含内存的基地址还而且还需要映射地址的长度。我们需要使用 #size-cells属性来表示reg属性元素中表示地址长度元素的个数。在下面的例子中，每一个节点的address值有一个32位无符号整形数据而且length值也是用一个32位无符号整形数据来表示。因此在32的系统中`#address-cells`和`#size-cells`都要设置为1，但是在64位系统中`#address-cells`就要设置成2了。具体设置如下：
```
/dts-v1/;

/ {
    #address-cells = <1>;
    #size-cells = <1>;

    ...

    serial@101f0000 {
        compatible = "arm,pl011";
        reg = <0x101f0000 0x1000 >;
    };

    serial@101f2000 {
        compatible = "arm,pl011";
        reg = <0x101f2000 0x1000 >;
    };

    gpio@101f3000 {
        compatible = "arm,pl061";
        reg = <0x101f3000 0x1000
               0x101f4000 0x0010>;
    };

    interrupt-controller@10140000 {
        compatible = "arm,pl190";
        reg = <0x10140000 0x1000 >;
    };

    spi@10115000 {
        compatible = "arm,pl022";
        reg = <0x10115000 0x1000 >;
    };

    ...

};
```
上述每个内存映射设备都设置了基地址以及地址大小。GPIO被分配了两个地址范围，分别是 0x101f3000…0x101f3fff 以及0x101f4000..0x101f400f。  
有一些设备可能有不同的寻址方案，比如一个设备挂载到总线上连接一个片选信号线，可以通过片选信号选择不同的设备。由于父节点可以定义了其子节点的地址映射域，所以可以选择最适合的一项来描述硬件设备。下面的代码就是把片选号码编入地址码挂载到外部总线上的一个设备。
```
external-bus {
       #address-cells = <2>;
       #size-cells = <1>;

       ethernet@0,0 {
           compatible = "smc,smc91c111";
           reg = <0 0 0x1000>;
       };

       i2c@1,0 {
           compatible = "acme,a1234-i2c-bus";
           reg = <1 0 0x1000>;
           rtc@58 {
               compatible = "maxim,ds1338";
           };
       };

       flash@2,0 {
           compatible = "samsung,k8f1315ebm", "cfi-flash";
           reg = <2 0 0x4000000>;
       };
   };
```
上面代码中，`#address-cells` 属性为2，则表示`reg`属性的`address`有两个地址域，其中一个表示片选号，另一个表示设备到片选基地址的偏移量，`#size-cells`为1，其地址范围量的个数还是一个32位的无符号整数。所以最后reg有三个属性值，分别表示片选号、偏移量、地址范围。
##### 无内存映射设备
有一些设备，他们在处理器总线上没有内存映射。他们拥有地址范围但是他们不被cpu直接的访问，而是被父设备驱动替代cpu进行访问。
举个例子，对于i2c设备，每个设备都会有一个指定的访问地址，但是这些设备不会有相关联的范围或者地址长度，这有点类似cpu节点地址分配。如下：
```
i2c@1,0 {
    compatible = "acme,a1234-i2c-bus";
    #address-cells = <1>;
    #size-cells = <0>;
    reg = <1 0 0x1000>;
    rtc@58 {
        compatible = "maxim,ds1338";
        reg = <58>;
    };
};
```
##### 地址转换（Range）
之前已经介绍过如何给设备分配地址了，但是这个地址都是设备的本地地址，我们并不知道如何将这些地址变成cpu可见的（可访问）。
根节点是从cpu的角度来描述地址空间的，根节点的子节点总是依赖于cpu的地址域，因此不需要显示地地址映射。比如，`serial@101f0000`设备描述的地址就是0x101f0000。
但是有些节点并不是根节点的直接子节点，因此其不使用cpu地址域。为了可以得到设备的映射地址，设备树必须说明一个域到另一个作用域的内存地址的转换关系，`range`属性就此诞生。
下面给出一个简单例子来介绍`range`属性:
```
/dts-v1/;

/ {
    compatible = "acme,coyotes-revenge";
    #address-cells = <1>;
    #size-cells = <1>;
    ...
    external-bus {
        #address-cells = <2>
        #size-cells = <1>;
        ranges = <0 0  0x10100000   0x10000     // Chipselect 1, Ethernet
                  1 0  0x10160000   0x10000     // Chipselect 2, i2c controller
                  2 0  0x30000000   0x1000000>; // Chipselect 3, NOR Flash

        ethernet@0,0 {
            compatible = "smc,smc91c111";
            reg = <0 0 0x1000>;
        };

        i2c@1,0 {
            compatible = "acme,a1234-i2c-bus";
            #address-cells = <1>;
            #size-cells = <0>;
            reg = <1 0 0x1000>;
            rtc@58 {
                compatible = "maxim,ds1338";
                reg = <58>;
            };
        };

        flash@2,0 {
            compatible = "samsung,k8f1315ebm", "cfi-flash";
            reg = <2 0 0x4000000>;
        };
    };
};
```
ranges属性是说明地址转换的一个列表，列表的每一个条目分别表示子节点的地址，父节点的地址，子节点地址空间的长度（内存空间大小），每个条目的个数分别是用子节点的`#address-cells`值,父节点`#address-cells`值,以及子节点`#size-cells`的值来表示。就上面出显得设备树代码来讲，子节点的`#address-cells`为2，父节点`#address-cells`为1，子节点`#size-cells`为1，则上面代码中程序中的`ranges`属性的含义就是：
- 片选号为0，偏移量为0的设备映射的地址为0x10100000..0x1010ffff
- 片选号为1，偏移量为0的设备映射的地址为0x10160000..0x1016ffff
- 片选号为2，偏移量为0的设备映射的地址为0x30000000..0x30ffffff

此外，假如父节点的地址空间与子节点的地址空间是相同的，可以添加一个空的ranges属性，当你看到一个空的ranges的时候，这意味着子节点的地址与父节点的地址空间是1:1映射的。
或许，我们会感到困惑，既然映射比例是1:1那么为什么还要地址转换。一些总线（比如PCI总线）用于完全不同的地址空间，然而这种总线需要把具体的地映射范围反应给操作系统。一些可DMA访问的总线设备，操作系统必须知道其真实地址。有些时候需要把一些共享相同软件可编程物理映射地址的设备分成一个组。是否1:1映射内存地址取决于操作系统需要获取的信息以及对硬件本身的描述。  
我们同时也注意到，在外部总线里面的i2c@1,0的节点中，并没有添加ranges属性信息。不同于外部总线，i2c中的设备不需要在cpu地址域中进行地址映射，cpu通过访问i2c@1,0来间接的访问rtc@58设备。一个节点缺少ranges属性意味着该设备不需要直接被除了父节点以外的设备访问。  
#### 中断如何工作
---
与依赖设备树结构的地址转换不同，中断信号可以由板子上任何设备产生与或者终止。对于一般的设备，其地址信息在设备树中自然的被表示出来，中断信号独立的在设备树节点中体现出来。下面的四个属性用来描述中断：
- `interrupt-controller` 一个空属性用来表示该节点描述的设备接受中断信号
- `#interrupt-cells` 这是一个中断控制器节点属性，该属性类似于`#address-cells`和`#size-cells`表示中断控制器包含多个中断描述符。
- `interrupt-parent` 该属性表示该节点有来自中断控制器的句柄，有些节点没有`interrupt-parent`属性，则表示继承其父节点的`interrupt-parent`属性。
- `interrupts` 该属性描述了设备节点包含的一系列的中断描述符，对应于该设备的中断输出信号。

一个中断描述符就是一个或者多个无符号32位整形数据的个数（具体的个数在#interrupt-cells中定义），重点描述符表示该设备接受哪些中断输入信号。就像下面设备树代码中的例子，大多数的设备仅仅有一个输出中断，但是有时候一个设备也有可能含有多个输出中断。这意味着一个中断描述符完全依赖绑定在设备上的中断控制器。每个中断控制器可以决定需要多少个cells来描述一个独一无二的输出中断。  
下面是 Coyote's Revenge 设备一个简单的中断例子：
```
/dts-v1/;

/ {
    compatible = "acme,coyotes-revenge";
    #address-cells = <1>;
    #size-cells = <1>;
    interrupt-parent = <&intc>;

    cpus {
        #address-cells = <1>;
        #size-cells = <0>;
        cpu@0 {
            compatible = "arm,cortex-a9";
            reg = <0>;
        };
        cpu@1 {
            compatible = "arm,cortex-a9";
            reg = <1>;
        };
    };

    serial@101f0000 {
        compatible = "arm,pl011";
        reg = <0x101f0000 0x1000 >;
        interrupts = < 1 0 >;
    };

    serial@101f2000 {
        compatible = "arm,pl011";
        reg = <0x101f2000 0x1000 >;
        interrupts = < 2 0 >;
    };

    gpio@101f3000 {
        compatible = "arm,pl061";
        reg = <0x101f3000 0x1000
               0x101f4000 0x0010>;
        interrupts = < 3 0 >;
    };

    intc: interrupt-controller@10140000 {
        compatible = "arm,pl190";
        reg = <0x10140000 0x1000 >;
        interrupt-controller;
        #interrupt-cells = <2>;
    };

    spi@10115000 {
        compatible = "arm,pl022";
        reg = <0x10115000 0x1000 >;
        interrupts = < 4 0 >;
    };

    external-bus {
        #address-cells = <2>
        #size-cells = <1>;
        ranges = <0 0  0x10100000   0x10000     // Chipselect 1, Ethernet
                  1 0  0x10160000   0x10000     // Chipselect 2, i2c controller
                  2 0  0x30000000   0x1000000>; // Chipselect 3, NOR Flash

        ethernet@0,0 {
            compatible = "smc,smc91c111";
            reg = <0 0 0x1000>;
            interrupts = < 5 2 >;
        };

        i2c@1,0 {
            compatible = "acme,a1234-i2c-bus";
            #address-cells = <1>;
            #size-cells = <0>;
            reg = <1 0 0x1000>;
            interrupts = < 6 2 >;
            rtc@58 {
                compatible = "maxim,ds1338";
                reg = <58>;
                interrupts = < 7 3 >;
            };
        };

        flash@2,0 {
            compatible = "samsung,k8f1315ebm", "cfi-flash";
            reg = <2 0 0x4000000>;
        };
    };
};
```
一些需要注意的事项：
- 这个设备只包含一个中断控制器，interrupt-controller@10140000.
- 标号为`intc:`添加到中断控制器节点中，这个标号主要是被用于根节点的`interrupt-parent`属性的赋值句柄，除非在根节点的子节点中明确的定义了`interrupt-parent`属性，否则所有根节点的子节点将继承根节点的`interrupt-parent`属性。
- 每个设备都会使用`interrupt`属性来描述一个独一无二的中断输入线.
- `#interrupt-cells`属性设置为2表示每个中断描述符包含2个cells，一般第一个cells表示表示中断线号，第二个cells表示一个标记号，比如表示该中断是高电平触发、是低电平触发还是电平触发等等。对于所有给定的中断控制器，请参考控制器绑定文档以便获取对象中断编码。  

#### 设备特定数据
---  
除了上面出现几个通用含义的属性，在子节点中可以添加任意属性信息。只要属性遵循下面的规矩，添加的任何属性都可被操作系统识别。
首先，新添加的自定义的属性名字需要加上制造商的名字作为前缀以免与标准通用的属性名发生冲突。
其次，每当定义了一个自定义属性都应该在相应的内核文档中加以说明，以便内核开发者可以明白该属性的具体含义。在相应的文档中需要有对该属性的说明，该属性值的含义，该属性可以赋予何值，以及该值的具体含义。
最后，邮件列表中devicetree-discuss@lists.ozlabs.org是收到的新的属性building，新的building审核可以获取到很多未来常见的错误。
#### 特殊节点
---
##### `aliases`节点
引用一个特定的节点通常是通过完整路径的方式，比如：/external-bus/ethernet@0,0，但是这样有不便，因为使用者根本不关心完整路径信息，他们关心的仅仅是哪个是eth0，`aliase`节点就是将一个全部路径心机简化为一个别名信息，比如：
```
aliases {
    ethernet0 = &eth0;
    serial0 = &serial0;
};
```
操作系统是欢迎你以这种方式来表述的。
这里你会发现一个新的语法`property = &label`;用字符串属性引用一个标号来替代一个完整的路径信息。但是这不同于上面介绍的`phandle =< &label >`，因为`phandle = < &label >`是把一个`phandle = < &label >`的值插入到一个cell.
##### `chosen`节点
`chosen`节点并不是代表一个真实的设备，仅仅是作为操作系统与固件之间传递数据的一个地方。相当于启动参数，在`chosen`节点的数据并不代表硬件。通常.Dts文件中`chosen`节点为空，在启动的时将会被填充：
```
chosen {
    bootargs = "root=/dev/nfs rw nfsroot=192.168.1.1 console=ttyS0,115200";
};
```
#### 进阶
---
##### 复杂一点的设备
到目前为止，我们了解了设备树的大概，为了提高普适性，我们新加入其他的一些稍微复杂难度的设备。
这里，在板子上添加一个PCI主桥，其内存映射地址为0x10180000，且BARS以0x8000000为起始地址。
考虑到我们已经了解了设备树，现在我们从如下的设备树节点来描述一个PCI主桥：
```
pci@10180000 {
    compatible = "arm,versatile-pci-hostbridge", "pci";
    reg = <0x10180000 0x1000>;
    interrupts = <8 0>;
};
```
##### PCI Host Bridge  
This section describes the Host/PCI bridge node.  
Note, some basic knowledge of PCI is assumed in this section. This is NOT a tutorial about PCI, if you need some more in depth information, please read[1]. You can also refer to either [ePAPR v1.1](https://www.power.org/documentation/power-org-standard-for-embedded-power-architecture-platform-requirements-epapr-v1-1-2/) or the [PCI Bus Binding to Open Firmware](http://playground.sun.com/1275/bindings/pci/pci2_1.pdf). A complete working example for a Freescale MPC5200 can be found here.
##### PCI总线编号
每个PCI总线都会被分配独立的编号，这些编号在`bus-ranges`属性中被定义，该属性包含两个cells，第一个表示当前节点PCI总线的标号，第二个表示子PCI总线的最大编号。  
下面给出只含有一个PCI总线的板级说明，因此bus-ranges两个属性值应该全是0：
```
pci@0x10180000{

        compatible= “arm,versatile-pci-hostbridge”, “pci”;

        reg =<0x10180000 0x1000>;

        interrupts = <8 0>;

        bus-ranges= <0 0>;

    };
```
##### PCI地址转换  
跟之前介绍的情况雷同，PCI地址空间与CPU地址空间是完全独立的，因此需要将PCI的地址映射到CPU的地址域。与之前一样，这里仍然使用`range`, `#address-cells`, 以及`#size-cells`这些属性。
```
pci@0x10180000 {
    compatible = "arm,versatile-pci-hostbridge", "pci";
    reg = <0x10180000 0x1000>;
    interrupts = <8 0>;
    bus-ranges = <0 0>;

    #address-cells = <3>
    #size-cells = <2>;
    ranges = <0x42000000 0 0x80000000 0x80000000 0 0x20000000
              0x02000000 0 0xa0000000 0xa0000000 0 0x10000000
              0x01000000 0 0x00000000 0xb0000000 0 0x01000000>;
};
```  

正如你所看到的，子地址（PCI地址）使用了3个cells，PCI地址范围使用了2个cells。或许你会有个疑问，为什么表示一个PCI地址需要3个cells，下面就这个问题我们做一个解释，首先地址有3个cells，我们把他分别定义为：phys.hi, phys.midand phys.low ，每一个cells为一个无符号32位整数，我们可以吧每一位用如下格式表示出来:  
- `phys.hi cell: npt000ss bbbbbbbb dddddfff rrrrrrrr`
- `phys.mid cell: hhhhhhhh hhhhhhhh hhhhhhhh hhhhhhhh`
- `phys.low cell: llllllll llllllll llllllll lllllll`

PCI地址是一个64位数据，所以分别用phys.hi及其phys.Low来表示高低32位，对于phys.hi，有如下的位域。具体含义如下：
 - n: 重定义标记 (在这里不起作用)
 - p: 可预区(缓存) 区域标记位
 - t: 地址别名标记位(在这里不起作用)
 - ss: 空间编码，具体含义如下:
   - 00: 配置空间
   - 01: I/O空间
   - 32 位地址空间
   - 64 为地址空间
 - bbbbbbbb:PCI总线编号，PCI总线可以分层结构，所以我们可能会在子总线定义一些PCI/PCI桥
 - ddddd: 设备编号，通常与初始化设备选择信号（IDSEL）有关系
 - fff: 功能选择号，用于多功能PCI设备
 - rrrrrrrr: 再配置周期使用的注册号

 为了达到地址转换的目的，最重要的域是p跟ss,这两个域直接决定访问哪个PCI空间。因此通过查看ranges属性，我们可以得到三个区域：
 - 一个32位可预取的存储区，从512 MB大小的PCI地址0x80000000开始，将映射到主机CPU上的地址0x80000000
 - 一个32位的非预取内存区域，从256 MB大小的PCI地址0xa0000000开始，将映射到主机CPU上的地址0xa0000000
 - 一个I / O区域，从16 MB大小的PCI地址0x00000000开始，将映射到主机CPU上的地址0xb0000000  

 为了完成这些操作，phys.hi位域的存在就意味着操作系统必须知道该节点代表了一个PCI桥，这样操作系统才能为了地址转换而忽略那些不相关的字段。为了判断应该忽略哪些额外的字段，操作系统需要在PCI总线节点中寻找\"pci\"字符串。  
##### 高级中断映射
下面我们开始介绍最有意思的部分——PCI中断映射，一个PCI设备可以被#INTA, #INTB, #INTC 以及 #INTD来触发。一个单功能的PCI设备显然使用#INTA来触发。一个多功能的设备可能使用#INTA单一指令，也可能同时使用#INTA和#INTB组合指令，等等。基于这些规则，#INTA指令显然比其他指令更频繁的使用。为了在四条IRQ线路上实现负载均衡，每个PCI插槽或设备通常以旋转方式（原文是 in rotating manner，应该是指的某种连线方式，感觉我翻译的不太准确）连接到中断控制器上的不同输入端，以避免所有INTA客户机都连接到相同的输入中断线路上。此过程称为混合中断。因此需要在设备树定义一种机制可以表示出PCI发出的信号与中断控制器之间的映射关系。`#interrupt-cells`, `interrupt-map`以及`interrupt-map-mask`就是用来描述这种映射关系。  
事实上，这种映射关系不仅仅存在于PCI总线，在其他的节点上也可以用此种方法来描述复杂的中断映射关系，只是这种关系在PCI设备上应用比较广泛。
```
pci@0x10180000 {
    compatible = "arm,versatile-pci-hostbridge", "pci";
    reg = <0x10180000 0x1000>;
    interrupts = <8 0>;
    bus-ranges = <0 0>;

    #address-cells = <3>
    #size-cells = <2>;
    ranges = <0x42000000 0 0x80000000  0x80000000  0 0x20000000
              0x02000000 0 0xa0000000  0xa0000000  0 0x10000000
              0x01000000 0 0x00000000  0xb0000000  0 0x01000000>;

    #interrupt-cells = <1>;
    interrupt-map-mask = <0xf800 0 0 7>;
    interrupt-map = <0xc000 0 0 1 &intc  9 3 // 1st slot
                     0xc000 0 0 2 &intc 10 3
                     0xc000 0 0 3 &intc 11 3
                     0xc000 0 0 4 &intc 12 3

                     0xc800 0 0 1 &intc 10 3 // 2nd slot
                     0xc800 0 0 2 &intc 11 3
                     0xc800 0 0 3 &intc 12 3
                     0xc800 0 0 4 &intc  9 3>;
};
```
首先可以注意到，PCI中断不同于系统的中断，系统中断号用2个cells来表示，一个代表IRQ中断号，另一个是标记号码，PCI中断只有一个cells，因为PCI中断总是低电平触发。  
在我们的实验板上，我们有两个PCI卡槽连接4个中断线，现在需要在中断控制器上映射8个中断线，此时就需要使用interrupt-map属性。  
仅仅通过中断号（#INTA等）是无法区分出单个PCI总线上的PCI设备的，但是我们还必须指出是哪个PCI设备触发了中断线。幸好每个设备都有一个独一无二的设备号，我们可通过该设备号识别不同的设备。为了辨别出到底是哪一个设备触发的中断线，我们需要一个元素，该元素需要包含PCI设备号以及PCI中断号，通俗的来说，我们需要构造一个中断说明单元，该单元包含如下四个元素：
- 3个 #address-cells 包含 phys.hi, phys.mid,phys.low
- 一个 #interrupt-cell (#INTA, #INTB, #INTC,#INTD)  

很多时候，我们仅仅需要PCI地址中的设备号部分，因此interrupt-map-mask 应运而生，interrupt-map-mask跟中断说明单元一样 也是拥有四个元素，第一个元素表示中断说明单元中哪一个部分应该被考虑。在我们的例子中，我们仅仅需要phys.hi中设备号部分且我们需要3个bit位来指明四个中断线（在计数的时候是从1开始的不是0）。  
现在我们可以构造interrupt-map 属性，这个属性是一个表格，其中表格中的每一个条目包含一个子（PCI总线）中断说明单元、一个父句柄（复制中断服务的中断控制器）以及一个父中断说明单元。所以我们在上面的代码的第一行就可以得出PCI中断#INTA被映射到IRQ9，低电平触发。  
现在，还有一点需要补充介绍一下PCI总线中断说明单元中的连接号，PCI总线中断说明单元重要组成部分就是pyhs.hi位域中的设备号部分，设备号是板级特定的，设备号具体的跟每个PCI主控制器如何激活各个设备的 IDSEL 管脚有关。就上面的例子来讲PCI卡槽1被分配的设备ID是24（0x18），PCI卡槽2被分配的设备ID是25（0x19）。对于每一个PCI卡槽的phys.hi中的值等与设备号左移11位到dddd位域得到的：
- phys.hi 插到PCI卡槽1的phys.hi 的值为 0xC000（0x18左移11位）
- phys.hi 插到PCI卡槽2的phys.hi 的值为 0Xc800（0x19左移11位）  

把所有的元素放到一起，则 interrupt-map属性为：
- 卡槽1#INTA对应主中断控制器的中断号为IRQ9, 低电平触发
- 卡槽1#INTB对应主中断控制器的中断号为IRQ10, 低电平触发
- 卡槽1#INTC对应主中断控制器的中断号为IRQ11, 低电平触发
- 卡槽1#INTD对应主中断控制器的中断号为IRQ12, 低电平触发  
同时
- 卡槽2#INTA对应主中断控制器的中断号为IRQ10, 低电平触发
- 卡槽2#INTB对应主中断控制器的中断号为IRQ11, 低电平触发
- 卡槽2#INTC对应主中断控制器的中断号为IRQ12, 低电平触发
- 卡槽2#INTD对应主中断控制器的中断号为IRQ9, 低电平触发

interrupts = <8 0>属性表示中断控制器PCI桥控制器本身可以触发中断，不要跟PCI设备使用INTA、INTB触发中断混淆。  
最后需要注意的事。就像interrupt-parent 属性一样，节点中 interrupt-map 属性的存在将改变子节点和孙节点的默认中断控制器。在这个 PCI 示例中，这意味着 PCI 主桥变成了默认中断控制器。如果一个通过 PCI 总线连接的设备同时还直接连接至另一个中断控制器，这时就需要指定它自己的 interrupt-parent 属性。
