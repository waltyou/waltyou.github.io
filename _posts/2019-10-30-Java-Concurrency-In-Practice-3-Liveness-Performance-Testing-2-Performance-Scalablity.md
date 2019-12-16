---
layout: post
title: Java 并发编程实战-学习日志（三）2：性能与可伸缩性
date: 2019-10-30 18:01:04
author: admin
comments: true
categories: [Java]
tags: [Java, Concurrency, Java Concurrency In Practice]
---

线程的最主要目的是提高程序的运行性能，使程序更加充分地发挥系统的可用处理能力，从而提高系统的资源利用率，此外，还可以使程序在运行现有任务的情况下立即开始处理新的任务，从而提高系统的响应性。许多提升性能同样会增加复杂性，也会增加在安全性和活跃性上发生失败的风险。

<!-- more -->

---

* 目录
{:toc}
---

# 对性能的思考

对于一个给定的操作，通常会缺乏某种特定的资源，例如 CPU 时钟周期、内存、网络带宽、I/O 带宽。数据库请求、磁盘空间以及其他资源。当操作性能由于某种特定的资源而受到限制时，通常将该操作称为资源密集型的操作，例如，CPU 密集型、数据库密集型等。

通过并发来获得更好的性能：更有效地利用现有处理资源，以及在出现新的处理资源时使程序尽可能地利用这些新资源。从性能监视的视角看，CPU 需要尽可能保持忙碌状态。

## 1. 性能与可伸缩性

应用程序的性能可以采用多个指标来衡量，例如服务时间、延迟时间、吞吐率、效率、可伸缩性以及容量等。其中一些指标（服务时间、等待时间）用于衡量程序的 "运行速度"，即某个指定的任务单元需要 "多快" 才能处理完成。另一些指标（生产量、吞吐量）用于程序的 "处理能力"，即在计算资源一定的情况下，能完成 "多少" 工作。

可伸缩性指的是：当增加计算资源时（例如 CPU、内存、存储容量或 I/O 带宽），程序的吞吐量或者处理能力能相应地增加。

## 2. 评估各种性能权衡因素：

在几乎所有的工程决策中都会涉及某些形式的权衡。在作出正确的权衡时通常会缺少相应的信息，例如，"快速排序" 算法在大规模数据集上的执行效率非常高，但对于小规模的数据集，"冒泡排序" 实际上更高效。要实现一个高效的排序算法，那么需要知道被处理数据集的大小，还有衡量优化的指标，包括：平均计算时间、最差时间、可预知性等。大多数优化措施都不成熟的原因之一是通常无法获得一组明确的需求。

进行决策时，会通过增加某种形式的成本来降低另一种形式的开销（例如，增加内存使用量以降低服务时间），也会通过增加开销来换取安全性。很多性能优化措施通常都是以牺牲可读性或可维护性，或会破坏面向对象的设计原则，例如需要打破封装，或带来更高的错误风险，因为通常越快的算法就越复杂。

对性能的提升可能使并发错误的最大来源，例如采用双重检查锁来减少同步的使用，由于并发错误是最难追踪和消除的错误，因此对于任何可能会引入这类错误的措施，都需要谨慎实施。

对性能调优时，一定要有明确的性能需求（才能知道什么时候需要调优，以及什么时候应该停止），此外还需要一个测试程序以及真实的配置和负载等环境。在对性能调优后，需要再次测量以验证是否到达了预期的性能提升目标。在许多优化措施中带来的安全性和可维护性等风险非常高。免费的 perfbar 应用程序可以给出 CPU 的忙碌程度信息，而通常目标就是使 CPU 保持忙碌状态，因此可以有效地评估是否需要进行性能调优或者已实现的调优效果如何。



#  Amdahl 定律

如果可用资源越多，那么问题的解决速度就越快，例如，如果参与收割庄稼的工人越多，那么就能越快地完成工作。而有些任务本质上是串行的，例如，即使增加再多的工人也不可能增加作物的生长速度。如果使用线程主要是为了发挥多个处理器的处理能力，那么就必须对问题进行合理的并行分解，并使得程序能有效地使用这种潜在的并行能力。

​    大多数并发程序都是由一系列的并行工作和串行工作组成。Amdahl 定律描述的：在增加计算资源的情况下，程序在理论上能够实现最高加速比，这个值取决于程序中可并行组件与串行组件所占比重。假定 F 是必须被串行执行的部分，根据 Amdahl 定律，在包含 N 个处理器的机器中，最高的加速比：

​    **Speedup <= 1 /  (F + (1-F) / N)**

当 N 趋近于无穷大时，最大的加速比趋近于 1/F。因此，如果程序有 50% 的计算需要串行执行，那么最高的加速比只能是 2（不管有多少个线程可用）；如果在初中有 10% 的计算需要串行执行，那么最高的加速比将接近 10。Amdahl 定律还量化了串行化的效率开销。在拥有 10 个处理器的系统中，如果程序中有 10% 的部分需要串行化执行，那么最高的加速比为 5.3（53%的使用率），在拥有 100 个处理器的系统中，加速比可达到 9.2（9%的使用率）。即使拥有无限多的 CPU，加速比也不可能为 10。随着处理器的增加，即使串行部分所占的百分比很小，也会极大地限制当增加计算资源时能够提升的吞吐率。




[![](/images/posts/java-Amdahl.gif)](/images/posts/java-Amdahl.gif) 



## 1. 示例：在各种框架中隐藏的串行部分

串行部分是如何隐藏在应用程序的架构中，可以比较当增加线程时吞吐量的变化，并根据观察到的可伸缩性变化来推断串行部分中的差异。例如多个线程反复地从一个共享 Queue 中取出元素进行处理，尽管每次运行都表示相同的工作量，但改变队列的实现方式，就能对可伸缩性产生明显的影响。

ConcurrentLinkedQueue 的吞吐量不断提升，直到到达了处理器数量上限，之后将基本保持不变。另一方面，当线程数量小于 3 时，同步 LinkedList 的吞吐量也会有某种程度的提升，但是之后会由于同步开销的增加而下降。当线程数量达到 4 或 5 个时，竞争将非常激烈，甚至每次访问队列都会在锁上发生竞争，此时的吞吐量主要受到上下文切换的限制。

吞吐量的差异来源于两个队列中不同比例的串行部分。同步 LinkedList 采用单个锁来保护整个队列的状态，并且在 offer 和 remove 等方法的调用期间都将持有这个锁。ConcurrentLinkedQueue 使用了一种更复杂的非阻塞队列算法，该算法使用原子引用来更新各个链接指针。在第一个队列中，整个的插入或删除操作都将串行执行，而在第二个队列中，只有对指针的更新操作需要串行执行。



# 线程引入的开销

单线程程序既不存在线程调度，也不存在同步开销，而且不需要使用锁来保证数据结构的一致性。在多个线程的调度和协调过程中都需要一定的性能开销：对于为了提升性能而引入的线程来说，并行带来的性能提升必须超过并发导致的开销。

## 1. 上下文切换

如果主线程是唯一的线程，它基本上不会被调度出去。另一方面，如果可运行的线程数大于 CPU 的数量，那么操作系统最终会将某个正在运行的线程调度出来，从而使其他线程能够使用 CPU。这将导致一次上下文切换，在这个过程中将保存当前运行线程的执行上下文，并将新调度进来的线程的执行上下文设置为当前上下文。

切换上下文需要一定的开销，而在线程调度过程中需要访问由操作系统和 JVM 共享的数据结构。应用程序、操作系统以及 JVM 都使用一组相同的 CPU。在 JVM 和操作系统的代码中消耗越多的 CPU 时钟周期，应用程序的可用 CPU 时钟周期就越少。但上下文切换的开销并不只是包含 JVM 和操作系统的开销。当一个新的线程被切换进来时，它所需要的数据可能不在当前处理器的本地缓存中，因此上下文切换将导致一些缓存缺失，因而线程在首次调度运行时会更加缓慢。这就是调度器会为每个可运行的线程分配一个最小执行时间，即使有许多其他的线程正在等待执行：它将上下文切换的开销分摊到更多不会中断的执行时间上，从而提高整体的吞吐量。（以损失响应性为代价）。

当线程由于等待某个发生竞争的锁而被阻塞时，JVM 通常会将这个线程挂起，并允许它被交换出去。如果线程频繁发生阻塞，那么它们将无法使用完整的调度时间片。在程序中发生越多阻塞（包括 I/O 阻塞，等待获取发生竞争的锁，或者在条件变量上等待），与 CPU 密集型的程序就会发生越多的上下文切换，从而增加调度开销，并降低吞吐量。

UNIX 系统的 vmstat 命令和 Windows 系统的 perfmon 工具都能报告上下文切换次数以及在内核中执行时间所占比例等信息。如果内核占用率较高（超过 10%），那么通常表示调度活动发生得很频繁，这很有可能是由 I/O 或竞争锁导致的阻塞引起的。

## 2. 内存同步 

在 synchronized 和 volatile 提供的可见性保证中可能会使用一些特殊指令，即内存栅栏（Memory Barrier）。内存栅栏可以刷新缓存，使缓存无效，刷新硬件的写缓冲，以及停止执行管道。内存栅栏可能同样会对性能带来间接的影响，因为它们将抑制一些编译器优化操作。在内存栅栏中，大多数操作都是不能被重排序的。

在评估同步操作带来的性能影响时，区分由竞争的同步和无竞争的同步非常重要。synchronized 机制针对无竞争的同步进行了优化（volatile通常是非竞争的）。虽然无竞争同步的开销不为零，但它对应用程序整体性能的影响微乎其微，而另一种方法不仅会破坏安全性，而且会使维护人员经历痛苦的排错过程。

现代的 JVM 能通过优化来去掉一些不会发生竞争的锁，从而减少不必要的同步开销。如果一个锁对象只能由当前线程访问，那么 JVM 就可以通过优化来去掉这个锁获取操作，因为另一个线程无法与当前线程在这个锁上发生同步。一些更完备的 JVM 能通过逸出分析（Escape Analysis）来找出不会发布到堆的本地对象引用（因此这个引用是线程本地的）。

```java
public String getStoogeNames(){
  List<String> stooges = new Vector<String>();
  stooges.add("1");
  stooges.add("2");
  stooges.add("3");
  return stooges.toString();
}
```

对 List 的唯一引用就是局部变量 stooges，并且所有封闭在栈中的变量都会自动成为线程本地变量。在 getStoogeNames 执行过程中，至少会将 Vector 上的锁获取/释放 4 次，每次调用 add 或 toString 时都会执行 1 次。然而，一个智能的运行时编译器通常会分析这些调用，从而使 stooges 及其内部状态不会逸出，因此可以去掉这 4 次对锁获取的操作。即使不进行逸出分析，编译器也可以执行锁粒度粗化（Lock Coarsening）操作，即将邻近的同步代码块用同一个锁合并起来。在 getStoogeNames 中，如果 JVM 进行锁粒度粗化，可能会把 3 个 add 与 1 个 toString 调用合并为单个锁获取/释放操作，并采用启发式方法来评估同步代码块中采用同步操作以及指令之间的相对开销。这不仅减少了同步的开销，同时还能使优化器处理更大的代码块，从而可能实现进一步的优化。

某个线程中的同步可能会影响其他线程的性能。同步会增加共享内存总线上的通信量，总线的带宽是有限的，并且所有的处理器都将共享这条总线。如果有多个线程竞争同步带宽，那么所有使用了同步的线程都会受到影响。

## 3. 阻塞

非竞争的同步可以完全在 JVM 中进行处理，而竞争的同步可能需要操作系统的介入，从而增加开销。当在锁上发生竞争时，竞争失败的线程肯定会阻塞。JVM 在实现阻塞行为时，可以采用**自旋等待**（Spin-Watiing，指通过循环不断尝试获取锁直到成功）或者通过操作系统挂起被**阻塞**的线程。这两种方式的效率高低，取决于上下文切换的开销以及在成功获取锁之前需要等待的时间。如果等待时间较短，则适合采用自旋等待方式，而如果等待时间较长，则适合采用线程挂起方式。有些 JVM 将根据对历史等待时间的分析数据在这两者之间进行选择，但大多数 JVM 在等待锁时都只是将线程挂起。

当线程无法获取某个锁或者由于在某个条件等待或在 I/O 操作上阻塞时，需要被挂起，在这个过程中将包含两次额外的上下文切换，以及所有必要的操作系统操作和缓存操作：被阻塞的线程在其执行时间片还未用完之前就被交换出去，而在随后当要获取的锁或者其他资源可用时，又再次被切换回来。（由于锁竞争而导致阻塞时，线程在持有锁时将存在一定的开销：当它释放锁时，必须要告诉操作系统恢复运行阻塞的线程）



# 减少锁的竞争

串行操作会降低可伸缩性，并且上下文切换也会降低性能。在锁上发生竞争时将同时导致这两种问题，因此减少锁的竞争能够提高性能和可伸缩性。

在对某个独占锁保护的资源进行访问时，将采用串行方式——每次只有一个线程能访问。虽然避免数据被破坏，但获得这种安全性时需要付出代价，如果在锁上持续发生竞争，将限制代码的可伸缩性。

在并发程序中，对可伸缩性的最主要威胁是独占方式的资源锁。

有两个因素将影响在锁上发生竞争的可能性：锁的请求频率，以及每次持有锁的时间。如果二者的乘积很小，那么大多数获取锁的操作都不会发生竞争，因此在该锁上的竞争不会对可伸缩性造成严重的影响。然而，如果在锁上的请求量很高，那么需要获取该锁的线程将被阻塞并等待。在极端情况下，即使有大量工作等待完成，处理器也会被闲置。

有 3 种方式可以降低锁的竞争程度：减少锁的持有时间、降低锁的请求频率、使用带有协调机制的独占锁.

## 1. 减小锁的范围

降低发生竞争可能性的一种有效方式，**是尽量缩短锁的持有时间**，所以可以将一些与锁无关的代码移出同步代码块，尤其是那些开销较大的操作，以及可能被阻塞的操作，例如 I/O 操作。

坏的示例：

```java
@ThreadSafe
public class AttributeStore {
    @GuardedBy("this") private final Map<String, String>
            attributes = new HashMap<String, String>();

    public synchronized boolean userLocationMatches(String name,
                                                    String regexp) {
        String key = "users." + name + ".location";
        String location = attributes.get(key);
        if (location == null)
            return false;
        else
            return Pattern.matches(regexp, location);
    }
}
```

好的示例：

```java
@ThreadSafe
public class BetterAttributeStore {
    @GuardedBy("this") private final Map<String, String>
            attributes = new HashMap<String, String>();

    public boolean userLocationMatches(String name, String regexp) {
        String key = "users." + name + ".location";
        String location;
        synchronized (this) {
            location = attributes.get(key);
        }
        if (location == null)
            return false;
        else
            return Pattern.matches(regexp, location);
    }
}
```

同步代码块不能过小——一些需要采用原子方式执行的操作（例如对某个不变性条件中的多个变量进行更新）必须包含在一个同步块中。此外，同步需要一定的开销，当把一个同步代码块分解为多个同步代码块时（在确保正确性情况下），反而会对性能提升产生负面影响。（如果 JVM 执行锁粒度粗化，那么可能会将分解的同步块又重新合并）在分解同步代码块时，理想的平衡点将与平台相关，但在实际中，仅当可以将一些 "大量" 的计算或阻塞操作从同步代码块中移出时，才考虑同步代码块的大小。



## 2. 减少锁的粒度

通过锁分解和锁分段来减少锁的粒度，采用多个互相独立的锁来保护独立的状态变量，从而改变这些变量在之前由单个锁来保护的情况，实现更高的可伸缩性，然而使用的锁越多，发生死锁的风险就越高。

如果在整个应用程序中只有一个锁，而不是为每个对象分配一个独立的锁，那么所有的同步代码块的执行就会变成串行化执行，而不考虑各个同步代码块中的锁。由于很多线程将竞争同一个全局锁，因此两个线程同时请求这个锁的概率将剧增，从而导致更严重的竞争。所以如果将这些锁请求分布到更多的锁上，能有效地降低竞争程度。由于等待锁而被阻塞的线程将更少，因此可伸缩性将提高。

如果一个锁需要保护多个互相独立的状态变量，可以将这个锁分解为多个锁，并且每个锁只保护一个变量，从而提高可伸缩性，并最终降低每个锁被请求的频率。

 在锁上存在适中而不是激烈的竞争时，通过将一个锁分解为两个锁，能最大限度地提升性能。如果对竞争并不激烈的锁进行分解，那么在性能和吞吐量等方面带来的提升将非常有限，但也会提高性能随着竞争而下降的拐点值。对竞争适中的锁进行分解时，实际上是把这些锁转变为非竞争的锁，从而有效地提高性能和可伸缩性。



## 3. 锁分段

在某些情况下，将锁分解进一步拓展为对一组独立对象上的锁进行分解，例如，在 ConcurrentHashMap 的实现中使用了一个包含 16 个锁的数组，每个锁保护所有散列通的 1/16，其中第 N 个散列通由第 (N mod 16) 个锁来保护。假设散列函数具有合理的分布性，并且关键字能够实现均匀分布，那么这大约能把对于锁的请求减少到原来的 1/16，因此其能支持多达 16 个并发的写入器。（要使得拥有大量处理器的系统在高访问量的情况下实现更高的并发性，还可以进一步增加锁的数量，但仅当你能证明并发写入线程的竞争足够激烈并需要突破这个限制时，才能将锁分段的数量超过默认的 16 个）

锁分段的一个劣势在于：与采用单个锁来实现独占访问相比，要获得多个锁来实现独占访问将更加困难并且开销更高。通常，在执行一个操作时最多只需获得一个锁，但在某些情况下需要加锁整个容器，例如 ConcurrentHashMap 需要扩展映射范围，以及重新计算键值的散列值要分布到更大的桶集合中时，就需要获取分段锁集合中所有的锁。（要获取内置锁的一个集合，能采用的唯一方式是递归）



## 4. 避免热点域

锁分解和锁分段能提高可伸缩性，因为它们都能使不同的线程在不同数据（或同一个数据的不同部分）上操作，而不会相互干扰。当每个操作都能请求多个变量时，锁的粒度将很难降低，这是性能与可伸缩性之间相互制衡的另一个方面，一些常见的优化措施，例如将一些反复计算的结果缓存起来，都会引入一些热点域，而这些往往会限制可伸缩性。

当实现 HashMap 时，要考虑如何在 size 方法中计算 Map 中的元素数量。最简单的方法是，在每次调用时都统计一次元素的数量。一种常见的优化措施是，在插入和移除元素时更新一个计数器，虽然这在 put 和 remove 等方法中略微增加一些开销，以确保计数器时最新的值，但这将把 size 方法的开销从 O(n) 降低到 O(1)。

在单线程或者采用完全同步的实现中，使用一个独立的计数能很好地提高类似 size 和 isEmpty 等方法的执行速度，但却导致更难以提升实现的可伸缩性，因为每个修改 map 的操作都需要更新这个共享的计数器。即使使用锁分段来实现散列链，那么在对计数器的访问进行同步时，也会重新导致在使用独占锁时存在的可伸缩性问题。一个看似性能优化的措施——缓存 size 操作的结果，已经变成一个可伸缩性问题。在这种情况下，计数器也被称为热点域，因为每个导致元素数量发生变化的操作都需要访问它。

为了避免这个问题，ConcurrentHashMap 中的 size 将对每个分段进行枚举并将每个分段中的元素数量相加，而不是维护一个全局计数。为了避免枚举每个元素，其为每个分段都维护了一个独立的计数，并通过每个分段的锁来维护这个值。（如果 size 方法的调用频率与修改 Map 操作的执行频率大致相当，可以采用这种方式来优化所有已分段的数据结构，即每当调用 size 时，将返回值缓存到一个 volatile 变量中，并且每当容器被修改时，使这个缓存中的值无效（将其设为 -1）。如果发现缓存的值非负，就表示这个值时正确的，可以直接返回，否则需要重新计算。）

## 5. 一些替代独占锁的方法

放弃使用独占锁，从而有助于使用一种友好并发的方式来管理共享状态，例如，使用并发容器、读-写锁、不可变对象以及原子变量。

ReadWriteLock 实现一种在多个读操作以及单个写操作情况下的加锁规则：如果多个读操作都不会修改共享资源，那么这些读操作可以同时访问该共享资源，但在执行写操作时必须以独占方式来获取锁。对于读操作占多数的数据结构，ReadWriteLock 能提供比独占锁更高的并发性，而对于只读的数据结构，其中包含的不变性可以完全不需要加锁操作。

原子变量提供了一种方式来降低更新热点域时的开销，例如静态计数器、序列发生器、或者对链表数据结构中头节点的引用。原子变量类提供了在整数或者对象引用上的细粒度原子操作（因此可伸缩性更高），并使用了现代处理器中提供的底层并发原语。如果在类中只包含少量的热点域，并且这些域不会与其他变量参与到不变性条件中，那么用原子变量来替代它们能提高可伸缩性。（通过减少算法中的热点域，可提高可伸缩性——虽然原子变量能降低热点域的更新开销，但并不能完全消除。）



## 6. 不使用对象池

在 JVM 的早期版本中，对象分配和垃圾回收等操作的执行速度非常慢，但在后续的版本中，这些操作的性能得到了极大提高。事实上，现在 Java 的分配操作已经比 C 语言的 malloc 调用更快：在 HotSpot 1.4.x 和 5.0 中，"new Object" 的代码大约只包含 10 条机器指令。

为了解决 "缓慢的" 对象生命周期问题，许多开发人员都选择使用对象池，在对象池中，对象能被循环使用，而不是由垃圾收集器回收并在需要重新分配。在单线程程序中，尽管对象池能降低垃圾收集操作的开销，但对于高开销对象以外的其他对象来说，仍然存在性能缺失（除了损失 CPU 指令周期外，在对象池中还存在一些其他问题，其中最大的问题就是如何正确地设定对象池的大小（如果对象池太小，将没有作用，而如果太大，则会对垃圾回收器带来压力，因为过大的对象池将占用其他程序需要的内存资源）。如果在重新使用某个对象时没有将其恢复到正确的状态，那么可能会产生一些 "微妙的" 错误。此外，还可能出现一个线程在将对象归还给线程池后仍然使用该对象的问题，从而产生一种 "从旧对象到新对象" 的引用，导致基于代的垃圾回收器需要执行更多的工作）。

在并发应用程序中，对象池的表现更加糟糕。当线程分配新的对象时，基本上不需要在线程之间进行协调，因为对象分配器通常会使用线程本地的内存块，所以不需要在堆数据结构上进行同步。然而，如果这些线程从对象池中请求一个对象，那么久需要通过某种同步来协调对对象池数据结构的访问，从而可能使某个线程被阻塞。如果某个线程由于锁竞争而被阻塞，那么这种阻塞的开销将是内存分配操作开销的数百倍，因此即使对象池带来的竞争很小，也可能形成一个可伸缩性瓶颈。（即使是一个非竞争的同步，所导致的开销也会比分配一个对象的开销大）虽然这看似是一种性能优化，但实际上却会导致可伸缩性问题。在特定环境中，例如 J2ME 或 RTSJ，需要对象池来提高内存管理或响应性管理的效率等特定用途，才应该使用对象池。

---

# 减少上下文切换的开销

当任务在运行和阻塞这两个状态之间转换时，就相当于一次上下文切换。在服务器应用程序中，发生阻塞的原因之一就是在处理请求时产生各种日志消息。为了说明如何通过减少上下文切换的次数来提高吞吐量，将对两种日志方法的调度行为进行分析。

在大多数日志框架中都是简单地对 println 进行包装，当需要记录某个消息时，只需将其写入日志文件中。另一种记录日志方法：记录日志的工作由一个专门的后台线程完成，而不是由发出请求的线程完成。二者在性能上可能存在一些差异，这取决于日志操作的工作量，即有多少线程正在记录日志，以及其他一些因素，例如上下文切换的开销等。（如果日志模块将 I/O 操作从发出请求的线程转移到另一个线程，那么通常可以提高性能，但也会引入更多的设计复杂性，例如中断（当一个在日志操作中阻塞的线程被中断，将出现什么情况）、服务担保（日志模块能否保证队列中的日志消息都能在服务结束之前记录到日志文件）、饱和策略（当日志消息的产生速度比日志模块的处理速度更快时，将出现什么情况），以及服务生命周期（如何关闭日志模块，以及如何将服务状态通知给生产者）。）

日志操作的服务时间包括 I/O 流类相关的计算，如果 I/O 操作被阻塞，那么还会包括线程被阻塞的时间。操作系统将这个被阻塞的线程从调度队列中移走并直到 I/O 操作结束，这将比实际阻塞的时间更长。当 I/O 操作结束时，可能有其他线程正在执行它们的调度时间片，并且在调度队列中有些线程位于被阻塞线程之前，从而进一步增加服务时间。如果有多个线程在同时记录日志，那么还可能在输出流的锁上发生竞争，这种情况的结果与阻塞 I/O 的情况一样——线程被阻塞并等待锁，然后被线程调度器交换出去。在这种日志操作中包含了 I/O 操作和加锁操作，从而导致上下文切换次数的增多，以及服务时间的增加。

请求服务的时间不应该过长，主要有以下原因。首先，服务时间将影响服务质量：服务时间越长，意味着有程序在获得结果时需要等待更长的时间。但更重要的是，服务时间越长，就意味着存在越多的锁竞争。锁被持有时间应该尽可能短，因为锁的持有时间越长，那么在这个锁上发生竞争的可能性就越大。如果在大多数的锁获取操作上不存在竞争，那么并发系统就能执行得更好，因为在锁获取操作上发生竞争时将导致更多的上下文切换。在代码中造成的上下文切换次数越多，吞吐量就越低。

通过将 I/O 操作从处理请求的线程中分离出来，可以缩短处理请求的平均服务时间。调用 log 方法的线程将不会再因为等待输出流的锁或 I/O 完成而被阻塞，它们只需将消息放入队列，然后就返回到各自的任务中。另一方面，虽然在消息队列上可能会发生竞争，但 put 操作相对于记录日志的 I/O 操作是一种更为轻量级的操作，因此在实际使用中发生阻塞的概率更小（只要队列没有填满）。由于发出日志请求的线程现在被阻塞的概率降低，因此该线程在处理请求时被交换出去的概率也会降低。只要把一条包含 I/O 操作和锁竞争的复杂且不确定的代码路径变成一条简单的代码路径。通过把所有记录日志的 I/O 转移到一个线程，还消除了输出流上的竞争，将提升整体的吞吐量，因为在调度中消耗的资源更少，上下文切换次数更少，并且锁的管理也就更简单。

通过把 I/O  操作从处理请求的线程转移到一个专门的线程，类似于两种不同救火方案之间的差异：第一种方案是所有人排成一队，通过传递水桶来救火；第二种方案是每个人都拿着一个水桶去救火。在第二种方案中，每个人都可能在水源和着火点上存在更大的竞争（结果导致了只能讲更少的水传递到着火点），此外救火效率也会更低，因为每个人都在不停地切换模式（装水、跑步、倒水、跑步）。在第一种解决方案中，水不断地从水源传递到燃烧的建筑物，人们付出更少的体力却传递了更多的水，并且每个人从头至尾只需做一项工作。正如中断会干扰人们的工作并降低效率，阻塞和上下文切换同样会干扰线程的正常执行。
