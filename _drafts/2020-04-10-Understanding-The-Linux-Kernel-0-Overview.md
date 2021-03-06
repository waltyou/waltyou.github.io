---
layout: post
title: 深入理解 Linux 内核（零）：总览及绪论
date: 2020-04-10 18:11:04
author: admin
comments: true
categories: [Linux]
tags: [Linux, Linux Kernel]
---

Linux 作为主流且好评如潮的操作系统 ，内核是什么样子的呢？来了解一下吧。

<!-- more -->

---

* 目录
{:toc}
---



## 全书脑图

[![](/images/posts/UnderstandingLinuxKernel-Overview.png)](/images/posts/UnderstandingLinuxKernel-Overview.png)



可以看出这本书大概分为十一个部分，涵盖了Linux内存管理、文件系统、进程管理、进程通信等。

来一步步学习吧。



## 绪论

Linux 是 Unix-like 操作系统大家庭中的一名成员。

### 1. 与其他 Unix 内核的比较

Linux 内核2.6版本遵循IEEE POSIX 标准。

与其他内核相比：

- 单块结构的内核
- 编译并运行的传统 Unix 内核
- 内核线程
- 多线程应用程序支持
- 抢占式内核
- 多处理器支持
- 文件系统
- STREAMS

优势：

- 免费
- 所有成分都可以定制
- 可以运行在便宜、低档的硬件平台
- 强大
- 开发者都很出色
- 内核非常小，而且紧凑
- 与许多通用操作系统高度兼容
- 很好的技术支持

### 2. 操作系统基本概念

操作系统是一系列程序的集合。在这个集合中，最重要的程序是内核。它在操作系统启动时，直接被装入RAM中。

操作系统必须完成两个主要目标：

- 与硬件部分互交，为包含在硬件平台上的所有底层可编程部件提供服务
- 为运行在计算机系统上的应用程序(用户程序)提供执行环境

当程序想要使用硬件资源的时候，会通知内核，由内核来评估是否可以使用，然后和硬件资源交互。为了实现上述机制，现代操作系统依靠特殊的硬件特性来禁止用户直接与地层硬件互交，或者禁止直接访问任意的物理地址。硬件为CPU引入了至少两种模式:用户程序的**非特权模式**(User Mode)和内核的**特权模式**(Kernel Mode)。

#### 多用户系统

并发：竞争各种资源。

独立：每个应用程序执行自己的任务。

特点：

- 认证机制，核实用户身份
- 一个保护机制，防止有错误的用户程序妨碍其他应用程序在系统中运行
- 一个保护机制，防止有恶意的用户程序干涉或窥视其他用户的活动
- 记账机制，限制分配给每个用户的资源数

#### 用户和组

在多用户系统中，每个用户拥有自己私用的空间。用户互相之间不能访问彼此的文件。

用用户标识符（User ID）来标示一个用户。用组标识符（Group ID）来表示一个组。

root 超级用户（superuser），几乎无所不能。

#### 进程

进程（process）执行程序的一个实例。

由于资源都是有限的，所以多用户系统下的进程会被调度器进行调度，另外这些进程都必须是抢占式的。

#### 内核体系结构

大部分Unix是单块结构：每一个内核层都被集成到整个内核程序中，并代表当前进程在内核态下运行。操作系统的学术研究都是面向微内核的，但是这个样的操作系统一般比单块内核的效率低，因为操作系统不同层次之间显式的消息调用和传递会花费一定的时间。微内核程序模块化、便于移植、RAM利用率高、不需要执行的程序被调出。

Linux为了达到微内核理论上的优点而不影响性能，Linux内核提供了模块(module)，一个简单的目标文件；其代码在运行时链接到内核或从内核解除链接。**模块不是作为一个特殊的进程执行的，它代表当前进程在内核状态下执行**

模块的主要优点如下：

- 模块化方法
- 平台无关性
- 节省内存使用
- 无性能损失



### 3. Unix 文件系统概述

#### 文件

是字节序列组成的信息载体。

根目录（root directory）的表示是“/”。

“.” 表示当前目录，“..”表示父目录。

#### 硬链接和软链接

包含在目录内的一个文件，就是一个硬链接。它有两个限制：

- 不能给目录创建硬链接，避免出现环形图
- 只有同一文件系统的文件之间才能创建硬链接

为了克服以上限制，引入了软链接，也称符号链接。

#### 文件类型

- 普通文件
- 目录
- 符号链接
- 面向块的设备文件
- 面向字符的设备文件
- 管道和命名字符
- 套接字

#### 文件描述符和索引节点

文件系统处理文件所需要的信息，全都存在一个名为“索引节点（node）”的数据结构中。

文件索引节点基本属性和内容：

- 文件类型
- 与文件相关的硬链接个数
- 以字节为单位的文件长度
- 设备标识符(即包含文件的设备的标识符)
- 在文件系统中标识文件的索引节点
- 文件拥有者的UID
- 文件的用户ID
- 几个时间戳，标识索引节点状态改变的时间、最后访问时间及最后修改时间
- 访问权限和文件模式。

#### 访问权限和文件模式

文件的用户非为三类：

- 自己
- 同组用户
- 其他

有三种权限：

- 读
- 写
- 执行

#### 文件操作的系统调用

文件系统是硬盘分区物理组织的用户级视图，每个实际的文件必须在内核状态下运行。



### 4. Unix内核概述

#### 进程/内核模式

内核本身并不是一个进程，而是一个进程的管理者。

Unix系统还包括内核线程（Kernel Thread）的特权进程，他们具有一下特点：

- 以内核态运行在内核地址空间
- 不与用户直接互交，因此不需要终端设备
- 在系统启动时创建，然后一直处于活跃状态，直到系统关闭。

激活内核状态的几种方法：

- 进程调用系统调用
- 正在执行进程的CPU发出一个异常(exception)信号，异常是一些反常情况
- 外围设备发送一个CPU中断信号，通知一个事件的发生
- 内核线程被执行

#### 进程实现

每个内核有一个进程描述符，包有关进程当前状态的信息。方便内核管理进程。

当内核暂停一个进程的执行时，就把几个相关处理器寄存器的内容保存在进程描述符中。这些寄存器包括：

- 程序寄存器(PC)和栈指针(SP)寄存器
- 通用寄存器
- 浮点寄存器
- 包含CPU状态信息的处理控制寄存器(处理器状态字，Process Status Word)
- 用来跟踪进程对RAM访问的内存管理寄存器

内核重新来执行中断的程序时，它用进程描述符中适合的字段来装载CPU寄存器。

#### 可重入内核

所有的Unix内核都是可重入的，因此若干个进程可以同时在内核状态下执行。

当一个中断发生时，内核可以挂起当前执行的进程，转而处理其它事，这个是内核很重要的功能。

最简单的情况下，内核顺序执行指令，但当发生下述事件之一时，CPU交错执行内核控制路径：

- 运行在用户态下的进程调用一个系统调用,并且这个请求无法马上被满足
- 运行一个内核控制路径时，CPU检测到一个异常(缺页中断)。
- CPU正在运行一个启用了中断的内核控制路径时，一个硬件中断发生。第一个还没执行完，CPU开始执行另外一个内核控制路径，在这种情况下，两个内核控制路径运行在同一进程的可执行上下文中，所花费的系统CPU时间都算给这个进程。
- 支持抢占式的内核中，CPU正在运行，一个更高优先级的进程加入就绪队列，则中断发生。

#### 进程地址空间

每个进程拥有自己的私有地址空间，内核可以重入，每个内核控制路径都引用它自己的私有栈内核。

进程间可以共享部分地址空间，以实现一种进程间通信。

另外内核也会自动共享内存，比如在多个用户中共享指令内存，以达到节省内存空间的目的。

Linux支持mmap()系统调用，该系统调用允许存在在块设备上的文件或信息一部分映射到进程的部分地址空间。内存映射为为正常的读写传输数据方式提供了一个选择。

#### 同步和临界区

实现可重入内核需要同步机制：如果内核控制路径对某个内核数据结构操作时被挂起，那么其他内核控制路径就不应该再对此数据结构操作，除非它已经设为一致性状态。否则两个控制路径交互执行将破坏所存储的信息。

一般来说对全局变量的访问通过原子操作来保证。

以下是几种同步技术：

##### 1. 非抢占式内核

这与unix是具有抢占式进程的多处理操作系统并不矛盾。当进程在内核态执行时，他不能被任意挂起，也不能被另一个进程代替。因此，在单处理机上，中断或异常处理程序不能修改的所有内核数据结构，内核对他们的访问都是安全的。

如果内核支持抢占，那么在应用同步机制时，确保进入临界区前禁止抢占，退出临界区时启用抢占。

##### 2. 禁止中断

单处理机上的另一种同步机制是：在进入一个临界区之前禁止所有硬件中断，离开时再启用中断。这种机制比较简单，但不是最佳的，如果临界区比较大，那么在一个相当长的时间内，所有的硬件都将处于一个冻结状态。特别是在多处理机上，禁止本地中断是不够的。必须使用其他的同步技术。

##### 3. 信号量

广泛使用的一种机制是信号量，他在单处理器系统和多处理器系统都有效，信号量仅仅是与一个数据结构相关的计算器。所有的内核线程在试图访问这个数据结构之前，都要检查这个信号量。每个信号的组成如下：

##### 4. 自旋锁

在多处理器系统上，信号量并不是解决同步问题的最佳方案。为了检查信号量，内核必须把进程插入到信号量链表中，然后挂起他。因为这两种操作比较费时，完成这些操作时，其他的内核控制路径可能已经释放了信号量。

##### 5. 避免死锁

按规定的顺序来请求信号量来避免死锁。

#### 信号和进程间通信

Unix 信号提供了把系统事件报告给进程的一种机制。每种事件都有自己的信号编号，通常用一个符号常量来表示。

有两种系统事件：

- 异步通告：比如CTRL-C
- 同步错误或异常

#### 进程管理

Unix在进程和它正在执行的程序之间做出一个清晰的划分。

fork 函数用来创建一个进程，exit 函数用来终止一个进程，exec 则是装入一个新程序。



#### 内存管理

##### 1. 虚拟内存

即逻辑内存，处于应用程序的内存请求与硬件内存管理单元(MMU)之间，虚拟内存优点如下：

- 若干个进程可以并发的执行
- 应用程序所需内存大于可用物理内存时也可以运行
- 程序只有部分代码装入内存时进程可以执行它
- 允许每个进程访问物理内存的子集
- 进程可以共享函数或程序的一个单独内存印象
- 程序是可以重定位的，也就是说，可以把程序放在物理内存的任何地方
- 程序员可以编写与机器无关的代码，因为他们不必关心有物理内存的组织结构

##### 2. 随机访问存储器(RAM)的使用

将RAM划分为两个部分，存储内核映像(内核代码和内核静态数据结构)。其余部分使用虚拟内存系统来处理，并且存在一下可能的使用方面:

- 满足内核对缓冲区、描述符及其它动态内核数据结构的请求
- 满足进程对一般内存区的请求及对文件内存映射的请求
- 借助于高速缓存从磁盘及其它缓冲设备获得较好的性能

##### 3.  内核内存分配器(KMA)

是一个子系统，满足系统所有部分对内存的请求。基于不同的算法技术，提出了几种KMA

- 资源图分配算法(allocator)
- 2的幂次方空闲链表
- McKusick-Karels分配算法
- 伙伴(Buddy)系统
- Mach的区域(Zone)分配算法
- Dynix分配算法
- Solaris的Slab分配算法

##### 4. 进程虚拟地址空间处理

进程的虚拟地址空间包括了进程可以引用的所有虚拟地址内存地址。

##### 5. 高速缓存



#### 设备驱动程序

内核通过设备驱动程序和I/O设备交互。






## 参考资料

[深入理解LINUX内核(第三版)](https://book.douban.com/subject/2287506/)
