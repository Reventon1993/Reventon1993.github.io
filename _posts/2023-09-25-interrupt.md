---
layout: post
title: "interrupt"
category: x86
tags: [x86,interrupt]
---
{% include JB/setup %}


本文探讨X86架构下中断的硬件机制。因为涉及硬件，某些细节的理解实在是超出能力范围。

## 8259A PIC

### 8259A芯片

早期的计算机系统使用两块级联的8259A PIC(programmable interrupt controller)芯片(master和slave)作为系统的中断控制器，级联示意图如下：

<img src="https://raw.githubusercontent.com/Reventon1993/pictures/master/typora/x86_interrupt_pic1.png" alt="image-20230922164234747" style="zoom: 33%;" />

下图为8259A的内部结构和与CPU交互的图：

<img src="https://raw.githubusercontent.com/Reventon1993/pictures/master/typora/x86_interrupt_pic2.png" alt="image-20230922165959820" style="zoom: 33%;" />



### 内部寄存器

每个8259A芯片包含两组可编程的(programmable)寄存器：ICW(initialization command word)和OCW(operational command word)。ICW寄存器组用于初始化中断控制器，OCW寄存器组用于操作中断控制器。通过这两个寄存器组，可以间接地操作IRR、PR、ISR、IMR。

在使用8259A中断前必须对ICWs进行初始化。OCWs在工作时可以随时进行设置(不清楚能不能在工作时对ICWs操作)。

计算机使用PIO的方式访问ICWs和OCWs。

ICWs包含ICW1、ICW2、ICW3、ICW4，它们都是8位的寄存器。主8259A的ICW1映射到端口0x20h，ICW2、ICW3、ICW4映射到端口0x21h。从8259A的ICW1映射到端口0xA0h，ICW2、ICW3、ICW4映射到端口0xA1h。

OCWs包含OCW1、OCW2、OCW3，它们都是8位的寄存器。主8259A的OCW1映射到端口0x21h，OCW2、OCW3映射到端口0x20h。

从8259A的OCW1映射到端口0xA1h，OCW2、OCW3映射到端口0xA0h。

多个ICW和OCW寄存器被映射到了同一个端口，计算机是怎么区分他们的，这是一个令人疑惑的问题。具体需要查阅芯片手册，也可以简单地参考下文：

https://www.eeeguide.com/8259-programmable-interrupt-controller/



总结来说，ICWs是在初始化时设置，运行时通过OCWs可以完成下面操作：    

1. OCW1寄存器用于发送EOI信号（End of Interrupt）以结束中断处理。
2. OCW2寄存器用于屏蔽和解除屏蔽中断。
3. OCW3寄存器用于设置8259A芯片的其他操作。例如：设置特殊优先级、查询中断请求（IRQ）线状态等

### 中断工作流程

1. IRR(Interrupt Request Register)、IMR(Interrupt Mask Register)、ISR(Interrupt Service Register)都是8位寄存器，IR0-IR7分别对应其一个位。
2. 中断发生后，置位IRR中的对应的位。
3. 如果中断没有被屏蔽(IMR)，会触发中断评估。PR(Priority Resolver)设置了中断的优先级策略。中断评估过程中会对IRR中已发生的中断和ISR中正在处理的中断进行优先级相关的评估，评估是否要"产生"了新中断。如果ISR中存在正在处理的中断，且评估结果为"产生"了新中断(优先级更高的中断)，那么就出现了嵌套中断的场景。
4. 如果评估结果是"产生"了新中断，那么发送INT信号给CPU。
5. 在打开中断时(sti)，处理器在执行完每条命令后，会检测是否受到中断信号。收到中断信号后，CPU发送一个INTA的应答信号给8259A。
6. 8259A收到应答信号后，值为ISR对应的位，复位IRR对应的位。
7. CPU随后会发送第二个INTA信号。
8. 8259A收到第二个INTA信号，会把中断向量号发送到数据总线上供CPU读取。如果8259A采用AEOI(Automatic End Of Intterrupt)，那么会复位ISR对应的位。
9. 中断处理程序开始处理中断，如果采用是非AEOI的方式，那么中断处理程序需要主动发送EOI信号给8259A表示中断处理结束。8259A收到EOI后复位ISR对应的位。

## APIC

8259A PIC只适用于单核CPU的架构。在多核处理器架构中，使用APIC，APIC不再使用中断引脚，而是使用总线进行通信。

APIC由Local APIC(每个逻辑CPU对应一个)和主板上的I/O APIC构成(可能会有多个)。

### Local APIC

#### 寄存器组&内部结构

<img src="https://raw.githubusercontent.com/Reventon1993/pictures/master/typora/x86_interrupt_pic3.png" alt="image-20230925102628559" style="zoom:33%;" />



#### Local APIC版本寄存器

local apic版本寄存器记录了local apic的版本，同时还记录了LVT的表项数。

<img src="https://raw.githubusercontent.com/Reventon1993/pictures/master/typora/x86_interrupt_pic6.png" alt="image-20230925110138350" style="zoom:50%;" />

早期的Local APIC在CPU外部。后来集成到了CPU中，称之位xAPIC控制器。APIC控制器和xAPIC控制器最大的不同在于使用不同的总线和I/O APIC通信。可以通过版本ID判断出是否为xAPIC控制器。

<img src="https://raw.githubusercontent.com/Reventon1993/pictures/master/typora/x86_interrupt_pic8.png" alt="image-20230925111940002" style="zoom:50%;" />



#### x2APIC

xAPIC中的基础操作模式为xAPIC模式，高级的x2APIC模式在xAPIC模式基础上进一步扩展。其最重要的新特性就是可以把local APIC寄存器组映射到MSR寄存器组中。

CPUID.01h:ECX[21]查看是否支持x2APIC模式。

#### IA32_APIC_BASE

Local APIC中的寄存器组采用MMIO访问。在x2APIC中，还可以以MSR寄存器方式访问。

IA32_APIC_BASE用于配置寄存器组的物理基地值(MMIO)，同时还控制这xAPCI、x2APIC的使用，记录BSP。

<img src="https://raw.githubusercontent.com/Reventon1993/pictures/master/typora/x86_interrupt_pic7.png" alt="image-20230925110437582" style="zoom: 50%;" />

注意:

1.只有支持x2APIC模式的情况下，EXTD才有意义。

2.上电后，默认物理基地值0xFEE0 0000h。

3.上电后，系统会自动选择BSP，并置位其IA32_APIC_BASE的BSP。

#### Local APIC ID寄存器

系统上电后，会自动对每个Local APIC分配一个APIC ID值，记录在Local APIC ID寄存器中。BIOS和OS一般会以此APIC ID作为CPU的ID号。不同版本的APIC ID位长不一样。

<img src="https://raw.githubusercontent.com/Reventon1993/pictures/master/typora/x86_interrupt_pic9.png" alt="image-20230925113340900" style="zoom: 50%;" />

CPUID.0Bh:EDX获取Local APIC ID寄存器的值。

#### 中断源

Local APIC可以认为是每个cpu的中断"代理"。它不仅仅接受外设的I/O中断，下面是其所有中断源：

<img src="https://raw.githubusercontent.com/Reventon1993/pictures/master/typora/x86_interrupt_pic4.png" alt="image-20230925103241873" style="zoom: 50%;" />

#### LVT

LVT(Local Vector Table)用于对中断源的控制。下图为各LVT寄存器的位功能说明图。

<img src="https://raw.githubusercontent.com/Reventon1993/pictures/master/typora/x86_interrupt_pic5.png" alt="image-20230925104030708" style="zoom: 50%;" />

更具体的信息查阅Intel SDM。

#### 其他重要寄存器

ESR:错误状态寄存器

TRP、PPR、CR8用于控制中断的优先级

IRR、ISR、TMR

EOI

SVR

更具体的信息查阅Intel SDM。



### I/O APIC

I/O APIC是外设的中断"代理"。

#### 寄存器组

<img src="https://raw.githubusercontent.com/Reventon1993/pictures/master/typora/x86_interrupt_pic10.png" alt="image-20230925115749016" style="zoom: 50%;" />

#### I/O APIC版本寄存器(RO)

<img src="https://raw.githubusercontent.com/Reventon1993/pictures/master/typora/x86_interrupt_pic11.png" alt="image-20230925115959705" style="zoom:50%;" />

#### I/O APIC ID寄存器(R/W)

<img src="https://raw.githubusercontent.com/Reventon1993/pictures/master/typora/x86_interrupt_pic12.png" alt="image-20230925120137273" style="zoom:50%;" />

#### I/O中断定向投递寄存器组

每个中断请求引脚都对应一个I/O中断定向投递寄存器(简称RTE寄存器)。RTE寄存器描述了中断请求的触发方式、中断投递模式、中断向量号等，和上述的LVT十分相似。

<img src="https://raw.githubusercontent.com/Reventon1993/pictures/master/typora/x86_interrupt_pic13.png" alt="image-20230925140237405" style="zoom:50%;" />

#### 间接访问寄存器组

上述的I/O APIC寄存器不能直接访问，需要通过间接访问寄存器组间接访问。间接访问寄存器组包含3个寄存器，用于索引I/O APIC寄存器组的地址(使用寄存器组的索引值)、向I/O APIC寄存器组传递数据、向I/O APIC发送EOI消息。

<img src="https://raw.githubusercontent.com/Reventon1993/pictures/master/typora/x86_interrupt_pic14.png" alt="image-20230925140646405" style="zoom:50%;" />

#### OIC寄存器

OIC是一个2B的中断控制寄存器，位于芯片组配置寄存器的31FEh-32FFh偏移处。OIC寄存器控制着I/O APIC的使能和上述间接寄存器的xy值。

<img src="https://raw.githubusercontent.com/Reventon1993/pictures/master/typora/x86_interrupt_pic15.png" alt="image-20230925141106861" style="zoom:50%;" />

配置寄存器的地址存在RCBA寄存器中。该寄存器如何访问与具体的主板有关，需要查询硬件手册。

**BIOS会置位I/O APIC的使能位。**

### 中断共享

APIC中虽然在I/O APIC和Local APIC采用总线架构通信，但是I/O设备和I/O APIC依旧采用引脚连接。这就导致了I/O设备受限于中断引脚数。对此，某些引脚允许多个设备共享。具体的机制尚不清楚。

TODO

## 系统中断模式

下图是系统整体的硬件架构图：

<img src="https://raw.githubusercontent.com/Reventon1993/pictures/master/typora/x86_interrupt_pic17.png" alt="image-20230925142614502" style="zoom: 50%;" />

上图损失了Local APIC和CPU之间关系的细节。具体细节可以在参照下图：

<img src="https://raw.githubusercontent.com/Reventon1993/pictures/master/typora/x86_interrupt_pic16.png" alt="x86_interrupt_pic17" style="zoom: 67%;" />

注意：8259A的INTR引出的线和LINTIN0是有交点的。

总结，系统中存在如下中断路线：

1. I/O->8259A->INTR(CPU)
2. I/O->8259A->LINTIN0(Local APIC)->CPU
3. I/O->8259A->I/O APIC->LINTIN0(Local APIC)->CPU
4. I/O->I/O APIC=>Local APIC->CPU
5. NMI->NMI(CPU)
6. NMI->LINTIN1(Local APIC)->CPU

这些中断路线中是有重复的，系统要对中断路线进行选择和配置，存在如下三种中断模式：

1. PIC模式：只使用PIC中断。
2. Virtual Wire中断模式：仅仅使用BSP的Local APIC。根据是否使用I/O APIC又可细分成两种子模式。
3. Sysmmetric I/O：所有CPU都可以参与中断，这种模式一定是所有的Local APIC和I/O APIC都参与中断。

显然，一般情况下都使用Sysmmetric I/O模式，它的配置其实很简单：

1.置位IMCR.E0。该位控制NMI、8259A的中断请求是直接发送给CPU还是发送给LOCAL APIC。

2.屏蔽8259A相关的中断路线。可以使用8259A的IMR实现，也可通过LVT屏蔽LINTIN0中断实现。

这样上述的中断路线只剩下4和6。

## MSI

TODO
