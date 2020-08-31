---
layout: post
title:  "Linux memory addressing"
date: 2020-08-28 12:00:00 +0800
categories: jekyll update
---

1. 内存地址
2. 硬件中的分段
3. Linux中的分段
4. 硬件中的分页
5. Linux中的分页

我以前导师教导我们学习，首先要搞清楚概念定义是什么。

刘未鹏提出，学习一项知识，必须问自己三个重要问题：1. 它的本质是什么。2. 它的第一原则是什么。3. 它的知识结构是怎样的。
[一直以来伴随我的一些学习习惯(一)：学习与思考](http://mindhacks.cn/2008/07/08/learning-habits-part1/)

我自己摸索演化:

1.What 2.Why 3.How

1.它的概念定义是什么。2.为什么会出现它。3.它如何实现。

> 人与人学习的差距不在资质上，而在花在思考的时间和思考的深度上（后两者常常也是相关的）。
> 
> 获得的多少并不取决于读了多少，而取决于思考了多少、多深。
> 
> by刘未鹏

## 1. 内存地址

**logic address**

> Included in the machine language instructions to specify the address of an operand or of an instruction.

**linear address (virtual address)**

> A single 32-bit unsigned integer that can be used to address up to 4 GB that is, up to 4,294,967,296 memory cells.

**physical address**

> Used to address memory cells in memory chips.

为什么出现这三个概念，它为了解决什么问题。

每个机器指令都有个逻辑地址，物理地址解决物理内存访问。
线性地址（虚拟地址）是为了解决进程直接读写物理内存问题。

1、进程地址空间不能隔离，A进程的代码写溢出会影响到B进程，系统安全性会受影响。

2、内存使用效率低，为了支持多进程，物理内存有限，需要不时调换进程所用内存到磁盘。

3、进程每次运行，都需要足够大空闲内存，这块内存位置是不确定的，这会带来重定位问题，
也就是程序中变量和函数的地址重定位。

所以需要引入一个中间层来解决，于是virtual address
每个进程有自己独立的虚拟地址，这样所有进程地址隔离，也不存在重定位问题。

那么从逻辑地址怎么映射到线性地址（虚拟地址），引入了分段技术。
从线性地址（虚拟地址）映射到物理地址，引入了分页技术。
它由内存控制单元(MMU)控制。

接下来阐述How.

## 2. 硬件中的分段(Segmentation in Hardware)
2.1 逻辑地址由16位Segment Selector + 32位段内offset组成。

2.2 每个Segment由8-byte Segment Descriptor表示，段描述符放在GDT(Global Descriptor Table)或者LDT(Local Descriptor Table)中。

一般只定义一个GDT，而每个进程除了存放GDT中的段以外，还需要创建附加的段，于是有了LDT。
GDT在主存中的地址和大小存放在gdtr控制寄存器，当前正在被使用的LDT地址和大小放在ldtr控制寄存器。

2.3 快速访问段描述符

GDT和LDT放在内存中，为了加速逻辑地址到线性地址转换，于是需要快速访问段描述符。80x86处理器提供一种附加的非编程的寄存器。
每当一个段选择符被装入段寄存器时，相应的段描述符就由内存装入到对应的非编程CPU寄存器。

从那时起，针对那个段的逻辑地址转换就可以不访问内存中的GDT或者LDT，处理器只需直接引用存放段描述符的CPU寄存器即可。
仅当段寄存器内容改变，才有必要访问GDT或LDT。

![Fast Access to Segment Descriptors](https://williammuji.github.io/images/segment-descriptor.png)


2.4 分段单元(Segmentation Unit)

1、segment selector的TI字段决定GDT或LDT，分别由gdtr或ldtr寄存器得到线性基地址

2、segment selector头13位index字段计算段描述符的地址，index乘于8（段描述符大小），这个结果与第1步骤计算结果相加
获取段描述符数据

3、根据第2步骤所得段描述符Base字段的数值+逻辑地址段内offset就算出线性地址

![Segmentation Unit](https://williammuji.github.io/images/segment-unit.png)

## 3. Linux中的分段
比起分段，Linux更喜欢分页

所有进程使用相同的段寄存器值时，内存管理简单，它们共享同样一组线性地址

Linux目标是可以移植到大多数流行的处理器平台，分段支持有限。

2.6版的Linux只有在80x86结构下才需要支持分段。

运行在用户态和内核态的所有Linux进程都使用一对相同的段来对指令和数据寻址，即代码段和数据段。
它们的Base地址0x00000000，所以对Linux，逻辑地址与线性地址是一致的，即逻辑地址的offset与相应的线性地址值一致。

![four main Linux segments](https://williammuji.github.io/images/main-linux-segments.png)

3.1 Linux GDT

单处理器系统只有一个GDT，多处理器系统中每个CPU对应一个GDT。

3.2 Linux LDT

大多数用户态下的Linux程序不使用LDT。


## 4. 硬件中的分页

分页单元(paging unit)把线性地址转换城物理地址。

为了效率，线性地址被划分固定长度的页(page)。物理地址被划分固定长度的页框（page frame)。前者是个数据块，可以放在任何页框或磁盘中。
把线性地址映射到物理地址的数据结构称为页表(page table)。页表存放主存中，启用paging unit前需由内核对页表初始化。

4.1 常规分页

80386开始，Intel处理器的分页单元处理4KB的页。

![Paging by 80 x 86 processors](https://williammuji.github.io/images/8086-paging.png)

4.2 扩展分页(extened paging)

Pentium模型开始，80x86微处理器引入扩展分页，它允许页框大小为4MB。

![Extended paging](https://williammuji.github.io/images/extened-paging.jpg)

4.3 硬件高速缓存

动态RAM芯片的存取时间是微处理器时钟周期的数百倍。这意味着，从RAM取操作数或存放结果，CPU要等很久。

为了缩小CPU和RAM速度不匹配，引入硬件高速缓存内存(hardware cache memory)。

硬件高速缓存基于著名的局部性原理(locality principle)。程序最近常用的相邻地址在最近的将来又被用到
的可能性极大。因此引入小而快的内存来存放最近长使用的代码和数据变得很有意义。

4.4 转换后援缓冲器(TLB)(Translation Lookaside Buffer)

用于加快线性地址转换。当线性地址第一次被使用时，通过慢速访问RAM页表计算出物理地址。同时，物理地址被存放
在一个TLB表项(TLB entry)中，以便以后对同一个线性地址的引用可以快速的转换。

多处理器系统中，每个CPU都有自己的TLB。与硬件高速缓存相反，TLB对应项不必同步，因为运行在CPU上的进程，可以使用
同一个线性地址与不同物理地址发生联系。


## 5. Linux中的分页

Linux进程处理很大程度依赖于分页。事实上，线性地址到物理地址的自动转换使下面设计目标变得可行：
给每一个进程分配一块不同的物理地址空间，防止进程寻址错误。
区别页和页框，这就允许存放在某个页框的一个页，可以保存到磁盘中，之后重新装入该页，可以被装在不同的页框中。
这个是virtual address机制的基本要素。

5.1 物理内存布局

在初始化阶段，内核必须建立一个物理地址映射来指定那些物理地址违反对内核可用，而哪些不可用。内核讲下列页框标记为保留：

在不可用的物理地址范围内的页框

含有内核代码和已初始化的数据结构的页框

保留页框中的页不能被动态分配或交换到磁盘中。

![Kernel physical layout](https://williammuji.github.io/images/kernel-physical-layout.jpg)

5.2 进程页表

0x00000000-0xbfffffff，线性地址，无论进程运行在用户态还是内核态都可以寻址

0xc0000000-0xffffffff，线性地址，只有内核态的进程才能寻址


5.3 内核页表

内核初始化页表分为两个阶段

第一阶段，内核创建一个有限的地址空间，包括内核的代码段和数据段、初始页表和用于存放动态数据结构的共128KB大小的空间。

第二阶段，内核充分利用剩余的RAM适当的建立分页表。

当RAM小于896MB时的最终内核页表

当RAM896-4096MB时的最终内核页表

当RAM大于4096MB时的最终内核页表

5.4 固定映射的线性地址

内核线性地址第4个GB初始部分映射系统的物理内存。但至少128MB的线性地址总是留作他用，因为内核使用这些线性地址实现非连续
内存分配和固定映射的线性地址(fix-mapped linear address)。

5.5 处理硬件高速缓存和TLB

为了使高速缓存的命中率达到最大化，内核在下列决策中考虑体系结构：

一个数据结构中最常用的字段放在该数据结构内的低偏移部分，以便他们能够处于高速缓存的同一行中。

当为一大组数据结构分配空间时，内核试图把它们都存放在内存中，以便所有高速缓存行按同意方式使用。
