---
layout: post
title: "Computer System-结构分析＜四＞"
subtitle: '计算机结构基本分析，内存进阶篇'
author: "404"
header-style: text
tags:
  - Computer System
  - Memory
  - SDRAM
---

SDRAM(Synchronous dynamic random access memory)，同步动态随机访问内存，通常包括 SDR (Single Data Rate) SDRAMs以及DDR (Double Data Rate) SDRAMs.在显卡中常用的是GDDR SDRAMs以及HBM。

下图就是PC系统中常用的内存条，该内存条是双通道2G内存(dual inline Memory Module),通常简称为DIMM。我们可以看到内存条上黑色的128MB内存芯片，这些内存芯片简称为IC。该内存条是双面内存，就是说正反两面都有8个IC，总共16个IC，16*128M=2GB。DIMM的单面称作rank，比如下图的2GB内存条，它就是由rank1，rank2两个单面组成，每个面有8个IC。
![avatar](/img/in-post/Linux/201922502001.png)

每个IC内部通常由8个bank组成(DDR3通常为8个bank，GDDR5通常有16个bank)，这些bank共享一个memory I/O controller, 但是在每个bank内部的读写可以并行进行。

每个bank内部包括行地址解码器，列地址解码器，传感放大器，以及DRAM内存阵列。如图2所示，这些内存阵列由行列组成，每个行列交叉的单元，表示n bit，通常是8bit或者16位【每一位都是由一个晶体管和一个电容组成,在GDDR5和HBM内存中，通常为32Byte】，表示一个字节或者一个word。bank中的每一行组成一个page，每一行又包括很多列(这儿列是指单个交叉单元)。内存读写的最小单位就是这些交叉单元，通常只有这些单元被放入传感放大器的时候，才能够被读写，所以通常要不断在行和传感放大器之间移动数据。
![avatar](/img/in-post/Linux/201922502002.png)

把一行放入传感放大器称作"activate”，因为这个操作会激活bank。把传感放大器的内容放入行，称作“precharge”。有时候Read或者write的时候会隐含着 precharge的操作，称作AP-read,或者AP-write,AP(auto precharge)。

 在上图中每个bank由16k的page组成，每个page包括1k的列，每列是8bit的byte，所以总共16,384 rows/bank x 1,024 columns addresses/row x 1 byte/column address x 8 stacked banks=128M。

 对于DDR3，我们通常说它是8n-prefetch(这儿n是指每个rank的bank数目)，因为DDR3，每个IC有8个bank，每个bank读取数据的最小单位是8bit，一个byte。每次数据读取request，都会读取8*8bit=64bitdata，而不管这些数据是否都是我们所需要的，比如我们只需要其中的某个byte，但读request会读取8个byte。

 如下图所示， SDRAM读写通常能用一个简单的状态机来描述，它的状态包括idle, active, precharging一个或多个bank。和任何其它状态机一样，从一个状态转换到另一个状态，并在新的状态开始数据操作，都需要一些最小等待时间，这些时延会影响SDRAM读写数据的性能，从而影响整个计算机系统的性能。
 ![avatar](/img/in-post/Linux/201922502003.png)

 SDARM bank中的内存单元行列交叉(通常称作cell)点，用来存储数据，它通常都是一些电容和放大器组成，由于电容的特性，它的电量会随着时间衰减，比如温度等因素都会影响它的衰减速度，所以需要周期性进行加电刷新操作，维持其中的数据。刷新频率通常依赖于内存die的工艺以及cell本身的设计。对内存cell的读写和内存刷新有相同的效果，但是在电容电量衰减到必须刷新之前，并不是所有的内存cell都有读写操作，所以定时刷新仍是需要的。通常刷新操作是按行或者说page进行的，刷新之后，该行cell的电容就会被充电。通常的刷新操作周期是几百clocks到几千clocks。

在刷新命令之前，每个bank必须要先precharged，然后处于idle状态，这需要消耗一个tRP时延(The minimum number of clock cycles required between the issuing of the precharge command and activating a different row within the same bank)。在一个刷新命令完成后，所有的bank处于precharge (idle)状态，在刷新命令和下一个activate命令(ACT)之间cycles数目必须大于等于tRFC(the Row Refresh Cycle Time )。

由于数据传输时候，都有一定的时延，所以有下面的一些符号描述bank内数据传输的各个阶段时延。

参数|符号|注释
Row Active Time|TRAS|The minimum number of clock cycles required between a bank active command and issuing the precharge command.
Row Address to Column Address Delay|TRCD|The minimum number of clock cycles required between the activation of a row and accessing columns within it.
CAS latency|CL|The time between sending a column address to the memory and the beginning of the data in response. This is the time it takes to read the first bit of memory from a DRAM with the correct row already open.
Row Precharge Time|TRP|The minimum number of clock cycles required between the issuing of the precharge command and activating a different row within the same bank.
Activate to Activate in same bank|TRC|The minimum number of clock cycles required between the activation of a row activting another row in the same bank.
Burst| |The number of data beats in a column access. This is usually 8 for recent DDR3/GDDR5 devices.

SDRAM在响应读写命令之前，bank必须处于激活状态，内存控制器通过发送activate命令，指定被访问的rank，bank以及page(row)。激活一个bank的时间称作tRCD,the Row-Column (or Command) Delay ,它表示激活发送active命令，program控制逻辑以及把内存行列单元读取到传感放大器中以便读写的cycles数目。

Bank激活之后，传感放大器中有完整page内容，这个时候，可以发射读写命令，指定从某列开始读写数据。从某个激活的page(放在传感放大器中)中读取一个byte数据消耗的时间称作， the Column Address Strobe (CAS) Latency ,通常间歇位CL 或者tCAS, 它包括在读写接口发送读写命令，program控制逻辑，把传感放大器的内容传输入到输入输出缓冲，并把数据的第一个word放在内存总线上总共消耗的时间。

一个bank每次只能打开一个page(这儿打开是指把page内容放入到传感放大器)，对于处于打开状态的page，我们可以进行读写操作，如果不需要再对该page进行读写操作，可以关闭该page, 把该page内容写入bank的行列单元对应的page中，以便对其它page进行读写操作。这个关闭操作通过发射一个Precharge命令实现，precharge命令可以关闭某一个bank，也可以关闭rank中所有打开的bank。

Precharge命令可以和bank中的上一个读写操作进行绑定，从而进行一个组合操作，这时发送一个Read with Auto-Precharge (RDA) 或 Write with Auto-Precharge (WRA)代替单独的读写操作命令。只要满足一定的条件，这将允许SDRAM控制逻辑自动的打开或者关闭bank。需要满足的条件包括：(1) A minimum of RAS Activation Time (tRAS) has elapsed since the ACT command was issued, and (2) a minimum of Read to Precharge Delay (tRTP) has elapse since the most recent READ command was issued。

Precharge命令把传感放大器中的数据写入bank中对应的page中，然后DRAM core能够准备下一个数据访问。 precharge一个打开的bank所消耗的时间称作the Row Access Strobe (RAS) Precharge Delay ，通过写作tRP。同一个bank两个activate命令之间所消耗的时间称作tRC，它等于tRAS+tRP。不同bank的ACT命令间隔时间称作the Read-to-Read Delay (tRRD)。

下面的时序图标出了各个阶段时延：
 ![avatar](/img/in-post/Linux/201922502004.png)
