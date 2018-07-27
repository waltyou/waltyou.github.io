---
layout: post
title: 《深入理解Java虚拟机：JVM高级特性与最佳实践--第二版》学习日志（五）：高效并发
date: 2018-07-11 20:26:04
author: admin
comments: true
categories: [Java]
tags: [Java，JVM]
---

这是《深入理解Java虚拟机》的最后一部分，分为两章，分别为Java内存模型与线程、线程安全与锁优化。

<!-- more -->
---

学习资料主要参考： 《深入理解Java虚拟机：JVM高级特性与最佳实践(第二版)》，作者：周志明

---
## 目录
{:.no_toc}

* 目录
{:toc}

---

# Java 内存模型与线程

## 1. 硬件的效率与一致性

IO很慢，CPU很快，如何协调呢？所以就引入了高速缓存。如下图：

[![](/images/posts/processor_cache.png)](/images/posts/processor_cache.png)

## 2. Java 内存模型

### 1）主内存与工作内存

Java 内存模型的主要目标是定义程序中各个变量的访问规则，即在虚拟机中将变量存储到内存和从内存中取出变量这样的底层细节。

此处变量，包含了实例字段、静态字段和构成数组对象的元素，但不包括局部变量与方法参数，因为后者是线程私有的。

Java 内存模型规定了所有的变量都存储在**主内存**中，内调线程还有自己的**工作内存**。

线程、主内存、工作内存三者的交互关系如下图：

[![](/images/posts/JMM.jpg)](/images/posts/JMM.jpg)

### 2）内存间交互操作

- lock（锁定）：作用于主内存的变量，把一个变量标识为一条线程独占状态。
- unlock（解锁）：作用于主内存变量，把一个处于锁定状态的变量释放出来，释放后的变量才可以被其他线程锁定。
- read（读取）：作用于主内存变量，把一个变量值从主内存传输到线程的工作内存中，以便随后的load动作使用
- load（载入）：作用于工作内存的变量，它把read操作从主内存中得到的变量值放入工作内存的变量副本中。
- use（使用）：作用于工作内存的变量，把工作内存中的一个变量值传递给执行引擎，每当虚拟机遇到一个需要使用变量的值的字节码指令时将会执行这个操作。
- assign（赋值）：作用于工作内存的变量，它把一个从执行引擎接收到的值赋值给工作内存的变量，每当虚拟机遇到一个给变量赋值的字节码指令时执行这个操作。
- store（存储）：作用于工作内存的变量，把工作内存中的一个变量的值传送到主内存中，以便随后的write的操作。
- write（写入）：作用于主内存的变量，它把store操作从工作内存中一个变量的值传送到主内存的变量中。

如果要把一个变量从主内存中复制到工作内存，就需要按顺寻地执行read和load操作，如果把变量从工作内存中同步回主内存中，就要按顺序地执行store和write操作。Java内存模型只要求上述操作必须按顺序执行，而没有保证必须是连续执行。也就是read和load之间，store和write之间是可以插入其他指令的，如对主内存中的变量a、b进行访问时，可能的顺序是read a，read b，load b， load a。

Java内存模型还规定了在执行上述八种基本操作时，必须满足如下规则：
- 不允许read和load、store和write操作之一单独出现
- 不允许一个线程丢弃它的最近assign的操作，即变量在工作内存中改变了之后必须同步到主内存中。
- 不允许一个线程无原因地（没有发生过任何assign操作）把数据从工作内存同步回主内存中。
- 一个新的变量只能在主内存中诞生，不允许在工作内存中直接使用一个未被初始化（load或assign）的变量。即就是对一个变量实施use和store操作之前，必须先执行过了assign和load操作。
- 一个变量在同一时刻只允许一条线程对其进行lock操作，lock和unlock必须成对出现
- 如果对一个变量执行lock操作，将会清空工作内存中此变量的值，在执行引擎使用这个变量前需要重新执行load或assign操作初始化变量的值
- 如果一个变量事先没有被lock操作锁定，则不允许对它执行unlock操作；也不允许去unlock一个被其他线程锁定的变量。
- 对一个变量执行unlock操作之前，必须先把此变量同步到主内存中（执行store和write操作）。

### 3）对于volatile型变量的特殊规则

volatile是Java虚拟机提供的最轻量级的同步机制，但是它并不容易完全被正确、完整地理解。

volatile关键字有如下两个作用：
- 保证被volatile修饰的共享gong’x变量对所有线程总数可见的，也就是当一个线程修改了一个被volatile修饰共享变量的值，新值总数可以被其他线程立即得知。
- 禁止指令重排序优化。

#### 可见性

我们必须意识到被volatile修饰的变量对所有线程总是立即可见的，对volatile变量的所有写操作总是能立刻反应到其他线程中，但是对于volatile变量运算操作在多线程环境并不保证安全性。因为对volatile变量的运算操作有可能不是原子性的。

只有在以下两种情况下，我们可以不使用加锁来保证原子性：
- 运算结果不依赖变量的当前值，或者能够确保只有单一的线程修改变量的值
- 变量不需要与其他的状态变量共同参与不变约束

#### 禁止重排优化

volatile关键字另一个作用就是禁止指令重排优化，从而避免多线程环境下程序出现乱序执行的现象。

这里主要简单说明一下volatile是如何实现禁止指令重排优化的。先了解一个概念，内存屏障(Memory Barrier）。

内存屏障，又称内存栅栏，是一个CPU指令，它的作用有两个，一是保证特定操作的执行顺序，二是保证某些变量的内存可见性（利用该特性实现volatile的内存可见性）。由于编译器和处理器都能执行指令重排优化。如果在指令间插入一条Memory Barrier则会告诉编译器和CPU，不管什么指令都不能和这条Memory Barrier指令重排序，也就是说通过插入内存屏障禁止在内存屏障前后的指令执行重排序优化。Memory Barrier的另外一个作用是强制刷出各种CPU的缓存数据，因此任何CPU上的线程都能读取到这些数据的最新版本。

### 4）对long和double型变量的特殊规则

Java内存模型要求lock、unlock、read、load、assign、use、store和write这8个操作都具有原子性，但是对于64位的数据类型long和double，在模型中特别定义了一条宽松的规定：允许虚拟机将没有被volatile修饰的64位数据的读写操作划分为两次32位的操作来进行。

### 5）原子性、可见性与有序性

#### 原子性

对基本类型的访问读写是具备原子性的。更大范围的原子性保证，需要加锁。

#### 可见性

指当多个线程访问同一个变量时，一个线程修改了这个变量的值，其他线程能够立即看得到修改的值。

#### 有序性

即程序执行的顺序按照代码的先后顺序执行。

### 6）先行发生原则

先行发生是Java内存模型中定义的两项操作之间的偏序关系，就是说如果操作A先行发生于操作B，那么操作A产生的影响能被操作B观察到。

一下是几种 Java 内存模型中天然的先行发生关系：
- 程序次序规则：同一个线程内，按照代码出现的顺序，前面的代码先行于后面的代码，准确的说是控制流顺序，因为要考虑到分支和循环结构。
- 管程锁定规则：一个unlock操作先行发生于后面（时间上）对同一个锁的lock操作。
- volatile变量规则：对一个volatile变量的写操作先行发生于后面（时间上）对这个变量的读操作。
- 线程启动规则：Thread的start( )方法先行发生于这个线程的每一个操作
- 线程终止规则：线程的所有操作都先行于此线程的终止检测。可以通过Thread.join( )方法结束、Thread.isAlive( )的返回值等手段检测线程的终止。
- 线程中断规则：对线程interrupt( )方法的调用先行发生于被中断线程的代码检测到中断事件的发生，可以通过Thread.interrupt( )方法检测线程是否中断
- 对象终结规则：一个对象的初始化完成先行于发生它的finalize（）方法的开始。
- 传递性：如果操作A先行于操作B，操作B先行于操作C，那么操作A先行于操作C。

## 3. Java 与线程

### 1）线程的实现

线程是比进程更轻量级的调度执行单位，线程的引入可以把一个进程的资源分配和执行调度分开，各个线程既可以共享进程资源(内存地址，文件IO等)，又可以独立调度（线程是CPU调度的基本单位）。

Thread类的所有关键方法都声明了native的，意味着这个方法没有使用或无法使用平台无关的手段来实现，也有可能是为了执行效率。

实现线程主要有以下三种方式。

#### 使用内核线程实现

内核线程（KLT，Kernel-Level Thread），直接由操作系统内核（Kernel，即内核）支持的线程。

由内核来完成线程切换，内核通过操纵调度器（Scheduler）对线程进行调度，并负责将线程的任务映射到各个处理器上。每个内核线程可以视为内核的一个分身，这样操作系统就有能力同时处理多件事情，支持多线程的内核叫做多线程内核。 

程序一般不会去直接使用内核线程，而是去使用内核线程的一种高级接口——轻量级进程（LWP），即通常意义上的线程。

由于每个轻量级进程都由一个内核线程支持，因此只有先支持内核线程，才能有轻量级进程。*轻量级进程与内核线程之间1:1关系称为一对一的线程模型。 

内核线程保证了每个轻量级进程都成为一个独立的调度单元，即时有一个轻量级进程在系统调用中阻塞了，也不会影响整个进程的继续工作。 

**局限**：基于内核线程实现，因此各线程操作等需要系统调用，系统调用代价高，需要在用户态和内核态来回切换，其次，每个轻量级进程都需要一个内核线程的支持，因此轻量级进程要消耗一定的内核资源，如内核线程的栈空间，因此一个系统支持轻量级进程的数量是有限的。

#### 使用用户线程实现

广义上，内核线程以外，就是用户线程。轻量级也算用户线程，但轻量级进程的实现始终是建立在内核上的，许多操作都要进行系统调度，效率会受到限制。 

狭义上，用户线程指完全建立在用户空间的线程库上。这种线程不需要切换内核态，效率非常高且低消耗，也可以支持规模更大的线程数量，部分高性能数据库中的多线程就是由用户线程实现的。这种进程与用户线程之间1：N的关系称为一对多的线程模型。 

用户线程优势在于不需要系统内核支援，劣势也在于没有系统内核的支援，所有的线程操作都是需要用户程序自己处理。阻塞处理等问题的解决十分困难，甚至不可能完成。所以使用用户线程会非常复杂。

#### 使用用户线程加轻量级进程混合实现

内核线程与用户线程混合使用。可以使用内核提供的线程调度功能及处理器映射，并且用户线程的系统调用要通过轻量级线程来完成，大大降低整个进程被完全阻塞的风险。用户线程与轻量级进程比例是N:M。 

#### Java线程的实现

JDK1.2之前，绿色线程——用户线程。JDK1.2——基于操作系统原生线程模型来实现。

Sun JDK, 它的Windows版本和Linux版本都使用一对一的线程模型实现，一条Java线程就映射到一条轻量级进程之中。

Solaris同时支持一对一和多对多。

### 2）Java 线程调度

线程调度是指系统为线程分配处理器使用权的过程。

主要调度方式分两种，分别是协同式线程调度和抢占式线程调度。 

协同式线程调度，线程执行时间由线程本身来控制，线程把自己的工作执行完之后，要主动通知系统切换到另外一个线程上。最大好处是实现简单，且切换操作对线程自己是可知的，没啥线程同步问题。坏处是线程执行时间不可控制，如果一个线程有问题，可能一直阻塞在那里。 

抢占式调度，每个线程将由系统来分配执行时间，线程的切换不由线程本身来决定（Java中，Thread.yield()可以让出执行时间，但无法获取执行时间）。线程执行时间系统可控，也不会有一个线程导致整个进程阻塞。 

Java线程调度就是抢占式调度。 

希望系统能给某些线程多分配一些时间，给一些线程少分配一些时间，可以通过设置线程优先级来完成。Java语言一共10个级别的线程优先级（Thread.MIN_PRIORITY至Thread.MAX_PRIORITY），在两线程同时处于ready状态时，优先级越高的线程越容易被系统选择执行。但优先级并不是很靠谱，因为Java线程是通过映射到系统的原生线程上来实现的，所以线程调度最终还是取决于操作系统。

### 3）状态转换

Java定义了5种线程状态，在任意一个点一个线程只能有且只有其中一种状态。无限等待和等待可以算在一起。所以共五种。 

- 新建(New)：创建后尚未启动的线程。 
- 运行(Runnable)：Runnable包括操作系统线程状态中的Running和Ready，也就是处于此状态的线程有可能正在执行，也有可能等待CPU为它分配执行时间。线程对象创建后，其他线程调用了该对象的start()方法。该状态的线程位于“可运行线程池”中，变得可运行，只等待获取CPU的使用权。即在就绪状态的进程除CPU之外，其它的运行所需资源都已全部获得。
- 无限期等待(Waiting)：该状态下线程不会被分配CPU执行时间，要等待被其他线程显式唤醒。如没有设置timeout的object.wait()方法和Thread.join()方法，以及LockSupport.park()方法。 
- 限期等待(Timed Waiting)：不会被分配CPU执行时间，不过无须等待被其他线程显式唤醒，在一定时间之后会由系统自动唤醒。如Thread.sleep()，设置了timeout的object.wait()和thread.join()，LockSupport.parkNanos()以及LockSupport.parkUntil()方法。
- 阻塞（Blocked）：线程被阻塞了。与等待状态的区别是：阻塞在等待着获取到一个排他锁，这个事件将在另外一个线程放弃这个锁的时候发生；而等待则在等待一段时间，或唤醒动作的发生。在等待进入同步区域时，线程将进入这种状态。 

---

# 线程安全与锁优化

## 1. 线程安全

Brian Goetz 给出的定义：
> 当多个线程访问一个对象时，如果不用考虑这些线程在运行时环境下的调度和交替执行，也不需要进行额外的同步，或者在调用方法进行任何其他的协调操作，调用这个对象的行为都可以获取正确的结果，那这个对象是线程安全的。

### 1）Java 语言中的线程安全

Java中各种操作共享的数据分为5类：不可变、绝对线程安全、相对线程安全、线程兼容和线程对立。

#### 不可变

不可变（Immutable）的对象一定是线程安全的，无论是对象的方法实现还是方法的调用者，都不需要再采取任何的线程安全保障措施。

Java中，如果共享数据是一个基本数据类型，那么只要在定义时使用final关键字就可以保证它不可变。如果是一个对象，那就需要保证对象的行为不会对其状态产生任何影响才行。如java.lang.String类的对象，它是典型的不可变对象，调用它的API不会影响原来的值，只会返回一个新构建的对象。

#### 绝对线程安全

绝对线程安全要完全满足Brain Goetz给出的线程安全的定义。

但在Java API中标注自己是线程安全的类，大多数都不是绝对的线程安全。

#### 相对线程安全

就是通常意义上的线程安全，需要保证对这个对象单独的操作是线程安全的，在调用时无需做额外的保障措施，但是对于一些特定顺序的连续调用，可能需要在调用端使用额外的同步手段来保证调用的正确性。在Java中，大部分的线程安全的类都是这种类型。

#### 线程兼容

对象本身并不是线程安全的，但是可以通过在调用端正确的使用同步手段来保证对象在并发环境中可以安全的使用，平时说的一个类不是线程安全的，指的大多数都是这一种情况。

#### 线程对立

无论调用端是否采用了同步措施，都无法在多线程环境下并发使用的代码。 


### 2）线程安全的实现方法

#### 互斥同步

互斥同步（Mutual Exclusion & Synchronization）是常见的一种并发正确性保障手段，同步是指在多个线程并发访问数据时，保证共享数据在同一时刻只被一个（或是一些）线程使用。而互斥是实现同步的一种手段，临界区（Critical Section）、互斥量（Mutex）和信号量（Semaphore）都是主要的互斥实现方式。互斥是因，同步是果；互斥是方法，同步是目的。

#### 非阻塞同步

互斥同步最主要的问题是进行线程阻塞和唤醒带来的性能问题，因此这种同步成为阻塞同步（Blocking Synchronization）。其属于悲观的并发策略，总是认为只要不去做正确的同步措施，那么肯定就会出现问题。

现在有了另一个选择，基于冲突检测的乐观并发策略，就是先进行操作，如果没有其他线程争用共享数据，那操作就成功了；如果共享数据有争用，产生了冲突，就再采取其他的补偿措施（重试直到成功为止），这种乐观的并发策略的许多实现都不需要把线程挂起，因此称为非阻塞同步（Non-Blocking Synchronization）。

#### 无同步方案

要保证线程安全，并不是一定就要进行同步，两者没有因果关系。 同步只是保证共享数据争用时的正确性的手段，如果一个方法本来就不涉及共享数据，那它自然就无须任何同步措施去保证正确性，因此会有一些代码天生就是线程安全的。比如下列两类：

1. 可重入代码（Reentrant Code）
    
    这种代码也叫做纯代码（Pure Code），可以在代码执行的任何时刻中断它，转而去执行另外一段代码（包括递归调用它本身），而在控制权返回后，原来的程序不会出现任何错误。 相对线程安全来说，可重入性是更基本的特性，它可以保证线程安全，即所有的可重入的代码都是线程安全的，但是并非所有的线程安全的代码都是可重入的。可重入代码有一些共同的特征，例如不依赖存储在堆上的数据和公用的系统资源、 用到的状态量都由参数中传入、 不调用非可重入的方法等。 我们可以通过一个简单的原则来判断代码是否具备可重入性：如果一个方法，它的返回结果是可以预测的，只要输入了相同的数据，就都能返回相同的结果，那它就满足可重入性的要求，当然也就是线程安全的。
2. 线程本地存储（Thread Local Storage）
    
    如果一段代码中所需要的数据必须与其他代码共享，那就看看这些共享数据的代码是否能保证在同一个线程中执行？如果能保证，我们就可以把共享数据的可见范围限制在同一个线程之内，这样，无须同步也能保证线程之间不出现数据争用的问题。符合这种特点的应用并不少见，大部分使用消费队列的架构模式（如“生产者-消费者”模式）都会将产品的消费过程尽量在一个线程中消费完，其中最重要的一个应用实例就是经典Web交互模型中的“一个请求对应一个服务器线程”（Thread-per-Request）的处理方式，这种处理方式的广泛应用使得很多Web服务端应用都可以使用线程本地存储来解决线程安全问题。Java语言中，如果一个变量要被多线程访问，可以使用volatile关键字声明它为“易变的”；如果一个变量要被某个线程独享，Java中就没有类似C++中__declspec（thread）[3]这样的关键字，不过还是可以通过java.lang.ThreadLocal类来实现线程本地存储的功能。 每一个线程的Thread对象中都有一个ThreadLocalMap对象，这个对象存储了一组以ThreadLocal.threadLocalHashCode为键，以本地线程变量为值的K-V值对，ThreadLocal对象就是当前线程的ThreadLocalMap的访问入口，每一个ThreadLocal对象都包含了一个独一无二的threadLocalHashCode值，使用这个值就可以在线程K-V值对中找回对应的本地线程变量。

## 2. 锁优化

### 2）自旋锁与自适应自旋

自旋锁是指当遇到互斥锁时不再阻塞，先自旋一段时间如果还无法获得同步锁 
再阻塞。

自适应自旋是指自旋的时间自适应。

### 2）锁消除 

锁消除是指虚拟机即时编译器在运行时，对一些代码上要求同步，但是被检测到不可能存在共享数据竞争的锁进行消除。 

锁消除的主要判定依据来源于逃逸分析的数据支持，如果判断在一段代码中，堆上的所有数据都不会逃逸出去从而被其他线程访问到，那就可以把它们当做栈上数据对待，认为它们是线程私有的，同步加锁自然就无须进行。

### 3）锁粗化 

原则上，我们在编写代码的时候，总是推荐将同步块的作用范围限制得尽量小——只在共享数据的实际作用域中才进行同步，这样是为了使得需要同步的操作数量尽可能变小，如果存在锁竞争，那等待锁的线程也能尽快拿到锁。

大部分情况下，上面的原则都是正确的，但是如果一系列的连续操作都对同一个对象反复加锁和解锁，甚至加锁操作是出现在循环体中的，那即使没有线程竞争，频繁地进行互斥同步操作也会导致不必要的性能损耗。所以要将互斥锁的范围扩大。

### 4）轻量级锁

轻量级锁是JDK 1.6之中加入的新型锁机制，它名字中的“轻量级”是相对于使用操作系统互斥量来实现的传统锁而言的，因此传统的锁机制就称为“重量级”锁。 首先需要强调一点的是，轻量级锁并不是用来代替重量级锁的，它的本意是在没有多线程竞争的前提下，减少传统的重量级锁使用操作系统互斥量产生的性能消耗。

### 5）偏向锁 

偏向锁也是JDK 1.6中引入的一项锁优化，它的目的是消除数据在无竞争情况下的同步原语，进一步提高程序的运行性能。

如果说轻量级锁是在无竞争的情况下使用CAS操作去消除同步使用的互斥量，那偏向锁就是在无竞争的情况下把整个同步都消除掉，连CAS操作都不做了。

偏向锁的“偏”，就是偏心的“偏”、 偏袒“偏”，它的意思是这个锁会偏向于第一个获得它的线程，如果在接下来的执行过程中，该锁没有被其他的线程获取，则持有偏向锁的线程将永远不需要再进行同步。
