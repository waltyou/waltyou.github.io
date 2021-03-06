---
layout: post
title: 深入理解 Linux 内核（二）：进程
date: 2020-05-08 18:11:04
author: admin
comments: true
categories: [Linux]
tags: [Linux, Linux Kernel]
---

进程的概念是任何多线程操作系统的基石。进程通常被定义为一个正在运行的程序实例；因此，如果16个用户在同一时刻运行vi，就会有16个独立的进程（虽然他们共享相同的可执行代码）。进程在linux源码里经常被叫做任务（tasks）或者线程（threads）。

<!-- more -->

---

* 目录
{:toc}
---

## 进程、轻量级进程和线程

进程是一个正在运行的程序实例。你可以把它看做一个数据结构集合，它完整的描述了程序的执行进度。

从内核的角度看，进程的作用是充当一个系统资源（cpu时间，内存等）被分配的实体。

旧版本的linux内核不提供多线程支持。从内核角度看，一个多线程应用仅仅是一个普通的进程，它的多个执行流完全是在用户态创建、处理、和调度的，通常是使用一个符合posix标准的pthread库来实现。

Linux使用轻量级进程来提供对多线程更好的支持。基本地，两个轻量级进程可能共享相同的资源，像地址空间、打开的文件等。每当其中一个修改共享资源时，另一个能立刻看到这个改变。当然，这两个进程必须在访问共享资源时相互同步。

一个简单粗暴的实现多线程应用的方式是使轻量级进程和每个线程相关联。这种方式下，通过共享相同的内存地址空间、相同的打开文件集等，多个线程可以访问一组相同的应用程序数据结构；同时，每个线程又可以被内核独立调度，因此一个睡眠时，另一个还能继续运行。

## 进程描述符

为了管理进程，内核必须对每个进程正在做什么有一个清晰的画面。比如，它必须知道进程的优先级、它是正在运行还是被某个事件阻塞、它的地址空间是多少、它被允许访问哪些地址等等。

进程描述符发挥了这些作用，它是一个task_struct类型的结构，包含与单个进程有关的所有信息。正因为存放了这么多信息，进程描述符是非常复杂的。除了大量的包含进程属性的字段，文件描述符还包含若干个指向其他数据结构的指针，这些数据结构又包含指向其他结构的指针。

### 1. 进程状态

运行状态：TASK_RUNNING

- 进程正在运行或者等待运行。

可中断的等待状态：TASK_INTERRUPTIBLE

- 进程被挂起（睡眠）直到某些条件变为true。产生一个硬件中断、释放一个进程正在等待的系统资源、或者传递一个信号，这些都会唤起一个进程（把它的状态恢复为TASK_RUNNING）。

不可中断的等待状态：TASK_UNINTERRUPTIBLE

- 类似于TASK_INTERRUPTIBLE，唯一的不同是当传递一个信号给挂起的进程时，它的状态保持不变。这个状态很少被使用，但却是有用的，在某些特殊的情况下，进程在等待一个事件时不能被中断。举个例子，当一个进程打开文件设备，相应的设备驱动程序开始探测对应的硬件设备，在探测结束之前这个设备驱动程序必须不能被中断，否则硬件设备将会停留在不可预知的状态。

暂停状态：TASK_STOPPED

- 进程已经被暂停运行（只是被暂停运行，并不是终止运行）；进程在收到这些信号后进入该状态：SIGSTOP、SIGTSTP、SIGTTIN、SIGTTOU。

跟踪状态：TASK_TRACED

- 进程被调试器暂停了运行。当一个进程被其他进程监视时（比如当调试器执行ptrace来监视一个test程序时），每一个信号都会使该进程进入TASK_TRACED状态。

另外，有两个状态可以被同时存储于进程描述符state和exit_state字段；从名字就可以看出，只有在进程终止了才会进入这两种状态：

僵死状态：EXIT_ZOMBIE

- 进程已终止，但是父进程还没有发起wait4()或waitpid()系统调用来获取僵尸进程的信息。在wait族函数被调用之前，内核不能丢弃僵尸进程的描述符中的任何数据，因为父进程可能会用到。（查看本章最后一节中的“进程移除”一段）。

僵死撤销状态：EXIT_DEAD

- 这是最后的状态：进程已经被系统移除了，因为父进程刚调用了wait族函数。从EXIT_ZOMBIE到EXIT_DEAD状态的转换避免了一种竞争态：当其他线程对同一个进程调用了wait族函数时（查看第5章）。

### 2. 标识进程

进程和进程描述符严格的一对一关系使得task_struct的32位地址成为内核标识进程的有效手段。这些地址被称为进程描述符指针。内核对进程的大部分引用都是通过进程描述符指针的。

另一方面，类unix操作系统允许用户通过一个叫做进程ID（PID）的数字来标识进程，这个数字被存储在进程描述符的pid字段里。PID按顺序编号：新创建进程的PID通常是在前一个创建的进程的PID上加1。当然，PID值有上限，当内核达到了这个上限，它必须回收更小的未使用的PID。PID的最大默认值为32767（`PID_MAX_DEFAULT - 1`）;系统管理员可以通过向`/proc/sys/kernel/pid_max`（/proc挂载了一个特殊的文件系统，请查看第12章“特殊文件系统”）文件写入一个更小的值来减小这个上限。在64位架构体系下，系统管理员可以把上限值扩大到4,194,303。

当回收PID数值时，内核维护了一个可以表示哪些PID正在使用、哪些是空闲的`pidmap_array`位图。因为一个页帧大小是32768比特，在32位体系架构上，`pidmap_array`存储在单个页面中。 但是，在64位架构上，当内核使用了一个超过当前页大小的PID数值时，更多的页会加到位图中。这些页面从来不被释放。

Linux给系统中每个进程或者轻量级进程关联一个不同的PID。（我们将在本章之后看到，在多处理器操作系统中有小小的例外。）这种方法允许最大的灵活性，因为每个系统中的执行上下文都可以被唯一标识。

另一方面，UNIX程序员们希望同一组中的线程具有相同PID。比如，给一个PID发送信号来影响组中的所有线程。实际上，POSIX 1003.1c标准已经声明一个多线程应用程序中的所有线程必须拥有相同的PID。

为了满足这个标准，Linux使用了线程组技术。线程共享的标识符是线程组leader的PID，也就是组里第一个轻量级进程的PID；它存储在进程描述符的tgid字段里。getpid()系统调用返回当前进程的tgid值，而不是pid，因此一个多线程应用程序中的所有线程共享相同的标识符。大多数进程属于只有一个成员的线程组（单线程），作为组leader，它们的tgid和pid具有相同的值，如此getpid()系统调用对这类进程也可以正常工作。

#### 进程描述符的处理

进程是一个动态的实体对象，它们的生命周期短至几毫秒，长至几个月。因此内核必须能够同时处理很多个进程，把进程描述符存储在动态内存中而不是永久分配给内核的内存区域。对于每个进程，Linux在一个进程内存区域封装了两个不同的数据结构：一个跟进程描述符关联的小数据结构，叫做`thread_info`结构体，和内核态进程栈。

#### 标识当前进程

刚才提到的thread_info结构和内核栈的紧密关系对提高效率大有益处：内核通过esp可以很方便的获得当前正在运行的进程的thread_info地址。实际上，如果thread_union大小为8KB（2^13字节），内核通过掩码计算出esp的13个最低有效位来得到thread_info的基地址；类似的，如果thread_union为4KB，则取出12个最低有效位。current_thread_info函数可以实现这些功能，它会产生类似于下面的汇编指令：
```assembly
movl $0xffffe000,%ecx /* or 0xfffff000 for 4KB stacks */
andl %esp,%ecx
movl %ecx,p
```

当这三条指令执行完毕，p指向当前运行进程的`thread_info`结构的地址。

多数情况下内核需要的是进程描述符的地址而不是`thread_info`的地址。为了得到一个cpu上正在运行的进程的描述符指针，内核使用current宏，它等同于`current_thread_info->task`，并产生以下汇编指令：

```assembly
movl $0xffffe000,%ecx /* or 0xfffff000 for 4KB stacks */
andl %esp,%ecx
movl (%ecx),p
123
```

因为task为`thread_info`结构的第一个字段，所以当执行完这些指令后，p刚好指向该CPU当前正在运行进程的描述符。

current宏经常出现在内核代码中，用来获取进程描述符中的字段。比如：current->pid返回当前进程的PID。

另一个把进程描述符和栈存放在一起的好处体现在多处理器系统上：仅仅通过检查stack就可以获取每个CPU上正在运行的进程。早期的Linux版本没有把这两个结构存储在一起，相反的，它们使用一个全局静态变量current来定义当前运行的进程。在多处理器系统上，就需要把current定义为每个CPU独有的变量。

#### 双向链表

每个链表都需要实现一系列的基础操作：初始化、插入、删除、遍历等等。对每个不同的链表都重复实现这些基础操作不仅浪费程序员的时间也浪费了内存。

因此，Linux内核定义了`list_head`结构，它只有next和prev字段名分别表示一个通用双链表的前一个和下一个元素。但是，有一点必须要注意:`list_head`存放的是其他`list_head`的地址，而不是整个包含`list_head`的结构的地址.

`LIST_HEAD(list_name)`宏用来新建一个链表。它声明一个`list_head`类型的变量`list_name`，这是个傀儡节点，它充当新链表的头节点占位符，并初始化`list_head`的prev和next字段使得它们指向`list_name`节点自身。

Linux2.6内核支持另一种类型的双向链表，与`list_head`链表的主要不同在于它不是循环的。它主要在内存比较宝贵的哈希表里会用到，而且不能在O(1)的时间内寻找最后一个元素。链表头部存储在`hlist_head`结构里面，它只是一个简单的指向链表第一个元素的指针（链表为空时值为NULL）。每个元素保存在`hlist_node`结构里，它包含一个指向下一个元素的next指针和指向前一个元素的next字段的pprev指针（pprev是个很巧妙的设计）。因为链表不是循环的，最后一个元素的next为NULL（注:原文说第一个元素的pprev也为NULL，应当是不正确的，事实上它指向head的first字段）。

#### 进程列表



我们将要研究的第一个双链表是进程列表，它把所有系统存在的进程描述符串在一起。每个`task_struct`结构包含一个`list_head`类型的tasks字段，它的prev和next字段分别指向前一个和后一个`task_struct`元素。

进程链表的头部是一个`task_struct`类型的`init_task`描述符,它是所谓的进程0或交换区的描述符（请查看本章之后的“内核线程”一节）。`init_task`的tasks->prev字段指向链表最后一个元素的tasks字段。

`SET_LINKS`和`REMOVE_LINKS`宏分别用来从进程列表中插入和删除进程描述符。这些宏也关系到进程间的父子关系（查看本章之后的“进程是怎么组织的”一节）。

另一个有用的宏，叫做`for_each_process`，遍历整个进程列表。它的定义如下：

```c
#define for_each_process(p) \
for (p=&init_task; (p=list_entry((p)-> tasks.next, \
                                    struct task_struct, tasks) \
                                   ) != &init_task; )
```

这个宏是循环控制语句，内核程序员把循环体添加在其后面。请注意`init_task`是怎样只充当链表头的。这个宏开始时移动`init_task`到下一个task，然后继续循环一直到重新遇到`init_task`时为止（多亏了循环链表）。每次迭代，传递给宏的参数包含的当前扫描的进程描述符，也就是`list_entry`宏返回的值。

#### `TASK_RUNNING`进程链表

当寻找一个新的进程来运行在CPU上时，内核只需要关注那些可运行的进程（即处于`TASK_RUNNING`状态的进程）。

早期的Linux版本把所有可运行的进程放在同一个runqueue链表里面。因为要按进程优先级来维护链表成本太高，早期的调度器被迫扫描整个链表，为了选出最好的进程来运行。

Linux2.6用不同的方式来实现这个运行队列。目的是为了能使调度器在常量的时间内选出最好的可运行进程，而与可运行进程的数量无关。我们会在之后的第7张详细描述这种运行队列，这里只提供一些基本信息。

实现提高调度速度的策略包括把运行队列分割成许多可执行进程链表，每个优先级一个链表。每个`task_struct`描述符包含一个`list_head`类型的`run_list`字段。如果一个进程优先级为k（k为0-139之间的一个值），`run_list`字段把这个进程描述符链接到优先级为k的可运行进程链表。此外，对于多处理器系统，每个cpu有自己的运行队列。这是一个典型的增加数据结构复杂性来提高性能的例子：为了更高效地调度，把运行队列分割成140个不同的链表。

我们将会看到，内核必须为每个运行队列保存许多数据；但是，运行队列的主要数据结构是多个进程描述符链表；



### 3. 进程之间的关系

一个程序和它创建进程之间有父子关系。当一个进程创建多个子进程时，这些子进程具有兄弟关系。为了表示这些关系，必须在进程描述符里引入几个字段，表3-3里列出了给定进程P的这些字段。进程0和进程1由内核创建；我们将在本章之后看到进程1是所有其他进程的祖先。

**用来表示父子关系的进程描述符字段**

| 字段名      | 描述                                                         |
| ----------- | ------------------------------------------------------------ |
| real_parent | 指向创建进程P的进程的描述符，若不存在则指向进程1（init）的描述符。（因此，当用户启动一个后台进程并退出shell后，这个后台进程会立刻变成init进程的子进程。） |
| parent      | 指向进程P的真实父进程（它是当该进程终止后必须通知的进程）；它和`real_parent`极为相似，只在某些情况下可能会不同，比如另一个进程对P进程发起ptrace系统调用(请查看第20章“跟踪执行”一节)。 |
| children    | 包含所有由P创建的所有子进程的链表的头                        |
| sibling     | 指向兄弟进程链表中的下一个和前一个元素，它们和P具有相同的父进程。 |

此外，进程之间还有其他关系：一个进程可以是一个进程组或一个登陆会话的leader，它也可以是一个线程组的leader，它也可以跟踪其他进程的执行.

**建立非父子进程之间关系的描述符字段**

| 字段名          | 描述                                                         |
| --------------- | ------------------------------------------------------------ |
| group_leader    | P所属进程组leader进程的描述符指针                            |
| signal->pgrp    | P所属进程组leader进程的PID                                   |
| tgid            | P所属线程组leader进程的PID                                   |
| signal->session | P所属登录会话leader进程的PID                                 |
| ptrace_children | 被调试器跟踪的P的子进程链表的头                              |
| ptrace_list     | 指向实际父进程的被跟踪进程列表的下一个和前一个元素（当P被跟踪时用到） |

#### PID哈希表和链表

在一些情况下，内核必须能够由PID导致相应的进程描述符。比如在kill()系统调用里面。当进程P1想要发送一个信号给另一个进程P2时，它调用kill()系统调用，以P2的PID作为参数。内核由P2的PID导出进程描述符指针，然后从进程描述里提取出记录未决信号的数据结构指针。

顺序扫描进程列表并校验pid字段是可行的方案但是非常低效。为了加速搜索，引入了四个哈希表。为什么有多个哈希表？原因很简单，因为进程描述符对不同类型的PID有不同的字段（见表3-5），每种类型的PID需要有自己的哈希表。

**四个哈希表和他们对应的进程描述符字段**

| 哈希表类型   | 字段名  | 描述              |
| ------------ | ------- | ----------------- |
| PIDTYPE_PID  | pid     | 进程的PID         |
| PIDTYPE_TGID | tgid    | 线程组leader的PID |
| PIDTYPE_PGID | pgrp    | 进程组leader的PID |
| PIDTYPE_SID  | session | 会话leader的PID   |

这四个哈希表在内核初始化的时候被动态分配，它们的地址存储在`pid_hash`数组里。单个哈希表的大小取决于可用的RAM；比如，对于512M内存的系统，每个哈希表存放在四个页帧中并包含2048条数据。

Linux使用链表来解决碰撞问题；每个表项存放的是碰撞的进程描述符的双向链表的头。

### 4. 进程之是怎么被组织的

运行队列链表把所有处于`TASK_RUNNING`状态的进程组织在一起。对于处在其他状态的进程，则需要不同的对待，Linux有以下两种选择。

- 处于`TASK_STOPPED`，`EXIT_ZOMBIE`或者`EXIT_DEAD`状态的进程没有用特殊的链表来连接。没有必要把任何处于这些状态的进程组织在一起，因为暂停、僵尸或者死亡的进程只通过PID或者通过某个特定进程的子进程链表来访问。
- 处于`TASK_INTERRUPTIBLE`或者`TASK_UNINTERRUPTIBLE`状态的进程被分成很多种类，每种对应一个特殊的事件。在这种情况下，只从进程状态不能快速获取到进程，所以必须要引入额外的进程链表。这些就是接下来将要探讨的等待队列。

#### 等待队列

等待队列在内核里有几个用处，尤其是中断处理，进程同步，和定时器。因为接下来的章节会讨论这些话题，这里我们只简单的说一个进程必定经常等待某些事件的发生，比如磁盘操作的完成，系统资源的释放，定时器的触发等。等待队列实现了事件上的条件等待：一个希望等待某个特殊事件的进程把自己放到合适的等待队列里面并让出控制权。因此，等待队存放了一组睡眠中的进程，这些进程将会在某个条件为true时被内核唤醒。

等待队列本质是一个双链表，存放包含进程描述符指针的元素。每个等待队列由一个`wait_queue_head_t`类型的队列头来定义：

```c
struct _ _wait_queue_head {
    spinlock_t lock;
    struct list_head task_list;
};
typedef struct _ _wait_queue_head wait_queue_head_t;
```

因为等待队列会被各种中断处理函数和内核函数修改，当并发访问这个双链表时必须对其加以保护，否则可能导致不可预知的结果。同步是用队列头中的自旋锁lock来实现的。`task_list`字段是等待进程链表的头部。

等待队列的数据元素类型为`wait_queue_t`：

```c
struct _ _wait_queue {
    unsigned int flags;
    struct task_struct * task;
    wait_queue_func_t func;
    struct list_head task_list;
};
typedef struct _ _wait_queue wait_queue_t;
```

等待队列中的每个元素都代表了一个睡眠中并等待某个事件发生的进程；它的描述符地址存储在task字段。`task_list`字段则保存了所有等待相同事件的进程链表。

然而，唤起等待队列中的全部进程并不总是合适的。举例来说，比如两个或者更多的进程正等待对某个待释放资源的互斥访问，这时候只唤起等待队列中的一个进程是有意义的。一个进程得到资源，其他进程继续睡眠。（这避免了有名的“惊群”问题，即多个竞争同一个资源的进程被同时唤醒，而这个资源只能同时被一个进程访问，结果导致其他进程又重新进入sleep状态。）

因此，sleep中的进程可以分为两种：互斥的进程（相应等待队列元素中的flag等于1）被内核有选择的唤醒，而非互斥的进程都被唤醒。如果进程等待的资源只能授予一个进程，则它们是互斥进程，相反，如果等待的事件与所有等待中的进程都相关，则它们是非互斥的。比如，考虑一组进程正等待磁盘数据块的传输，一旦传输完成，所有的进程必须被唤醒。我们接下来会看到，等待队列元素中的func字段定义了等待的进程是怎样被唤醒的。

#### 处理等待队列

可以用`DECLARE_WAIT_QUEUE_HEAD(name)`宏来定义一个新的等待队列，它声明一个队列头静态变量name，并初始化`lock`和`task_list`字段。`init_waitqueue_head()`函数可以用来初始化一个动态分配的等待队列头变量。



### 5. 进程资源限制

每个进程都有一组关联的资源限制，用于指定进程可用的系统资源数量。这些限制阻止了用户压垮系统（CPU，磁盘，等待）。

当前进程的资源限制存储在`current->signal->rlimit`字段，即进程的信号描述符中.



## 进程切换

为了控制进程的执行，内核必须能够挂起正在运行的进程并恢复运行其他之前被挂起的进程。这个活动通过进程切换，任务切换或上下文切换执行这种各样的操作。接下来的章节介绍Linux系统上的进程切换。

### 1. 硬件上下文

虽然每个进程拥有自己的地址空间，但是它们必须共享相同的CPU寄存器。因此恢复执行一个进程之前，内核必须保证用该进程先前被挂起时的寄存器值来初始化这些寄存器。

这组在进程恢复运行之前必须被加载到CPU寄存器的数据叫做硬件上下文。硬件上下文是进程执行上下文的子集，进程执行上下文包含进程执行所需要的所有信息。在Linux上，一部分的硬件上下文存储在进程描述符中，剩下的部分保存在内核栈中。

进程切换只发生在内核态。进程在用户态使用的寄存器内容在执行进程切换前就已经被保存到内核栈里（参考第4章）。这些内容包含指示用户态栈指针地址的ss和esp寄存器。

### 2. 任务状态段

80x86架构包含一个特殊的段类型叫任务状态段（TSS），用来存储硬件上下文。虽然Linux硬件上下文切换中不会用到，但还是强制地为每个CPU设置了一个TSS。主要原因如下：

- 当一个80x86 CPU从用户态切换到内核态时，它从TSS获取内核栈的地址（参考第4章的“中断和异常的硬件处理”一节和第10章的“使用系统指令发起一个系统调用”一节）。
- 当一个用户态进程尝试通过in或者out指令访问I/O端口时，CPU需要通过TSS中的权限位图来验证该进程是否有权限访问这个端口。

更精确地说，当进程在用户态执行一个in或者out指令时，CPU控制单元会执行以下操作：

1. 检查eflags寄存器（标志寄存器）中的2位的IOPL字段。如果值为3，控制单元执行这个指令，否则，执行下一个任务。
2. 访问tr寄存器（任务寄存器）来决定当前的TSS，然后获取适当的I/O权限位图。
3. 检查I/O访问指令中的I/O端口对应的权限位图位，如果被清除，指令可以执行，否则抛出一个“常规保护”异常。

#### thread字段

每次进程切换，被替换出来的硬件上下文必须被存储在某个地方。它不能存在TSS中，因为Linux的设计：多个处理器使用一个TSS，而不是每个进程一个。

因此，每个进程描述符包含一个`thread_struct`类型的`thread`字段，内核把被切换出的硬件上下文存储在其中。正如我们将要看到的，这个数据结构包含大部分的CPU寄存器字段，除了那些通用寄存器，比如eax，ebx，它们存储在内核栈中（译者注：进程切换中并不需要专门恢复这些寄存器的值）。

### 3. 执行进程切换

进程切换可能只发生在一个明确定义的点： schedule()函数，在第7章会详细讨论它。这里我们只关注内核是怎样执行一个进程切换的。

实质上，每个进程切换分为两步：

1. 切换全局页目录，并装载一个新的地址空间；我们将会在第9章详细描述这步。
2. 切换内核态栈和硬件上下文，它们提供了内核执行一个新进程所需要的所有信息，包括CPU寄存器。

同样的，我们假设prev指向被替换的进程的描述符，next指向被激活的进程的描述符。我们在第7章会按到，prev和next是schedule()函数的局部变量。

#### switch_to宏

进程切换的第二部使用`switch_to`宏来执行的。它是内核最依赖硬件的例程之一，并且需要付出一些努力才能理解它的功能。

首先，这个宏有三个参数，prev、next和last。你应该很容易就能猜出prev和next的功能：它们是局部变量prev和next的占位符（placeholders），即它们作为输入参数定义了被替换进程和新进程的描述符的内存地址。

那么第三个参数last是做什么的呢？所有的进程切换都会涉及三个进程，而不是只有两个。假设内核决定关闭进程A并激活进程B。这schedule()函数中，prev指向A的描述符，next指向B的描述符。一旦宏`switch_to`关闭A，A的执行就被冻结了。

之后，当内核又要重新激活A时，它必须调用`switch_to`关闭另一个进程C（通常和B不一样），prev指向C，next指向A。当A恢复执行时，它找到旧的内核栈，此时，prev指向A的描述符，next指向B的描述符（A挂起之前的状态）。这时调度器当前执行的进程A丢失了到C的任何引用。然而这个对C的引用却对进程切换特别有用（从第7章可以查看更多细节）。

`switch_to`宏的最后一个参数last是个输出参数，用来存储进程C的描述符地址（当然，这时在A恢复执行后做的）。进程切换之前，这个宏把A的内核栈中prev的值存储在eax寄存器中，进程切换之后，当A已经恢复执行，把eax寄存器的内容写入last参数指向的内存。因为eax寄存器在进程切换中不会改变，因此last指定的位置存储了C的描述符地址。在当前的schedule()实现中，最后一个参数是进程A的prev地址，因此prev被C的地址覆盖。

####   __switch_to()函数

\__switch_to()函数做了由switch_to()宏开始的进程切换的大部分工作。它作用于分别表示之前进程和新进程的prev_p和next_p参数。然而，这个函数又与一般的函数不同，因为\__switch_to()的参数prev_p和next_p分别取自eax和edx寄存器，而不是像一般的函数那样从栈取参数。为了强制使这个函数从寄存器取参数，内核使用_attribute\__和regpram关键字，它们是gcc编译器实现的c语言非标准扩展。\__switch_to()函数在include/asm-i386/system.h头文件里声明：
```c
_ _switch_to(struct task_struct *prev_p,
            struct task_struct *next_p)
   _ _attribute_ _(regparm(3));
```

### 4. 保存和加载FPU、MMX、XMM寄存器

从Intel 80486DX开始，浮点运算单元被集成到CPU中。名词“数学协处理器”曾经一直被使用，那时的浮点运算是通过一个昂贵的专用芯片执行的。然而，为了和旧模式兼容，浮点运算通过“ESCAPE”指令实现，这些指令的字节前缀从0xd8到0xdf。它们作用于CPU的一套浮点寄存器。显然，如果一个进程使用ESCAPE指令，属于硬件上下文的浮点寄存器的内容必须被保存。

在后来的奔腾模型里，因特尔引入了一套新的汇编指令到微处理器中。它们叫做MMX指令，用来加速多媒体应用程序的执行。MMX指令作用于FPU的浮点寄存器。选择这种架构的一个明显的缺点是程序员不能混淆浮点指令和MMX指令。优点是，操作系统设计人员可以忽略这套新的指令集，因为相同的用来保存浮点运算单元状态的任务切换代码也可以用来保存MMX的状态。

MMX指令加速了多媒体应用程序，因为它们在处理器内部引进一个单指令多数据（SIMD）管道。奔腾3模型扩展了SIMD性能：它引入了SSE扩展（流式SIMD扩展），它增加了一个能够处理8个128位寄存器（XMM寄存器）中的浮点数的设施。这些寄存器与FPU和MMX寄存器不重叠，因此SSE和FPU/MMX指令可以自由组合。奔腾4模型还引进另一个特性：SSE2扩展，它是一个能够支持更高精度浮点值的SSE扩展。SSE2使用和SSE相同的一套XMM寄存器。

80x86微处理器不会自动保存TSS中的FPU，MMX和XMM寄存器。但是，它们包含一些硬件来使得内核能够在需要的时候保存这些寄存器。这些硬件支持包含cr0寄存器中的TS（Task-Switching）标志，它遵循以下规则：

- 每次硬件上下文切换，TS标志都被设置。
- 当TS标志被设置时，每次ESCAPE、MMX、SSE、SSE2指令的执行都会产生一个“设备不可用”异常（参考第4章）。

TS标志只允许内核在正真需要的是才保存和恢复FPU、MMX、XMM寄存器。为了解释他是怎样工作的，假设进程A正在使用数学协处理器。当从A到B发生上下文切换时，内核设置TS标志，并保存浮点寄存器到进程A的TSS中。如果新的进程B不使用数学协处理器，内核不需要恢复浮点寄存器的内容。但是一旦B尝试执行一个ESCAPE或者MMX指令，CPU产生一个“设备不可用”异常，然后相应的处理函数会把进程B的TSS中保存的值加载到浮点寄存器。

#### 保存FPU寄存器

之前已经提到过，`switch_to()`函数执行`__unlazy_fpu`宏，并以被替换的prev进程的描述符作为参数。这个宏会检查prev的`TS_USEDFPU`标志。被设置说明prev使用了FPU，MMX，SSE或者SSE2指令；因此，内核必须保存相关的硬件上下文：

```assembly
if (prev->thread_info->status & TS_USEDFPU)
    save_init_fpu(prev);
```

#### 加载FPU寄存器

浮点寄存器的内容的恢复并不是紧跟在next进程恢复执行后。但是cr0的TS标志已经被`__unlazy_fpu()`设置。因此，当next进程第一次尝试执行一个ESCAPE，MMX或SSE/SSE2指令时，控制单元产生一个“设备不可用”异常，然后（更准确的说，异常处理函数是被该异常唤起的）内核运行`math_state_restore()`函数。next进程被标识为current。

```assembly
void math_state_restore( )
{
    asm volatile ("clts"); /* clear the TS flag of cr0 */
    if (!(current->flags & PF_USED_MATH))
        init_fpu(current);
    restore_fpu(current);
    current->thread.status |= TS_USEDFPU;
}
```

该函数清除cr0的CW标志，因此之后执行的FPU，MMX或SSE/SSE2指令不会触发“设备不可用”异常。如果thread.i387子字段没有意义，比如`PF_USED_MATH`标志位0，则调用`init_fpu()`来重置thread.i387自字段并设置current的`PF_USED_MATH`标志为1。然后调用`restore_fpu()`函数从thread.i387自字段中加载合适的值到FPU中。这一步会根据CPU是否支持SSE/SSE2扩展来选择使用fxrstor或者frstor汇编指令。最后，`math_state_restore()`设置`TS_USEDFPU`标志。

#### 在内核态使用FPU，MMX，和SSE/SSE2单元

即使内核也可以使用FPU，MMX，或SSE/SSE2单元。这样做当然应该避免干扰当前用户模式进程的计算。因此：

- 在使用协处理器之前，内核必须调用`kernel_fpu_begin()`，它实际上是在用户进程使用FPU（TS_USEDFPU标志）的情况下调用`save_init_fpu()`来保存寄存器的内容 ，然后重置cr0的TS标志。
- 结束使用协处理器后，内核必须调用`kernel_fpu_end()`来设置cr0的TS标志。

之后，当用户态进程执行一个协处理器指令时，`math_state_restore()`函数会恢复这些寄存器的内容，就像进程切换那样。

但是，我们应该注意，当当前进程正在使用协处理器时，`kernel_fpu_begin()`的执行是非常耗时的，以至于能抵消FPU，MMX或SSE/SSE2单元带来的性能提升。实际上，内核只在极少的地方使用它们，通常是在移动或清除较大的内存区域或计算校验和函数时。



## 创建进程

Unix 操作系统通过进程创建来满足用户需求。

传统的 Unix  操作系统以一种统一的方式对待所有进程：子进程复制父进程所拥有的资源。这种方法非常低效，因为要拷贝父进程整个地址空间，而实际上，子进程几乎不必修改或读取父进程的资源。

现在Unix 内核引进3种不同的机制来解决了这个问题：

- 写时复制技术允许父子进程读取相同物理页。
- 轻量级进程允许父子进程共享进程在内核的很多数据结构。
- vfork() 系统调用创建的子进程可以访问父进程的内存地址空间。



### 1. clone(),fork() vfork()系统调用

轻量级进程是由 clone 函数创建的。

实际上， clone 函数是 C 语言库中一个封装函数，它负责建立轻量级进程的堆栈并且调用对编程者透明的clone 系统调用。

do_fork() 函数负责处理系统调用。

copy_process() 函数创建进程描述符以及子进程执行所需要的其他数据结构。



## 内核线程

针对一些需要周期性运行的重要任务，现代操作系统将他们委托到内核线程，内核线程可以不受用户态的上下文拖累，可以获得更好的响应。

内核线程有以下几方面区分于其他进程：

- 内核线程只运行在内核态，而普通进程既可以运行在内核态，也可以运行在用户态。
- 它们只能使用大于 PAGE_OFFSITE 的线性地址空间

### 1. 创建一个内核线程

kernel_thread() 函数创建一个新的内核线程。

#### 进程 0

所有进程的祖先叫做进程0、idle进程或者swapper进程，它是在 Linux 的初始化阶段从无到有创建的一个内核线程。

#### 进程 1

由进程 0 创建的内核线程执行  init 函数，init 函数依次完成内核初始化。



## 撤销进程

### 1. 进程终止

do_group_exit() 函数杀死属于current线程组的所有进程。它接受进程终止代码作为参数，进程终止代号可能是系统调用exit_group()指定的一个值，也可能是内核提供的一个错误代号。

进程终止所要完成的任务都是由do_exit函数来处理。




## 未完待续。。。。。。





