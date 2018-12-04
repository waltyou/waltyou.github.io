---
layout: post
title: 《Effective Java》学习日志（六）48：谨慎使用并行流
date: 2018-12-03 18:50:04
author: admin
comments: true
categories: [Java]
tags: [Java,Effective Java]
---

并行流并不像看起来的那么好用，它只在几个场景下可以提高效率。

<!-- more -->

---

学习资料主要参考： 《Effective Java Third Edition》，作者：Joshua Bloch

---

## 目录
{:.no_toc}

* 目录
{:toc}

---

在主流语言中，Java一直处于提供简化并发编程任务的工具的最前沿。 
当Java于1996年发布时，它内置了对线程的支持，包括同步和wait / notify机制。 
Java 5引入了java.util.concurrent类库，带有并发集合和执行器框架。 
Java 7引入了fork-join包，这是一个用于并行分解的高性能框架。 
Java 8引入了流，可以通过对parallel方法的单个调用来并行化。 
用Java编写并发程序变得越来越容易，但编写正确快速的并发程序还像以前一样困难。 
安全和活跃度违规（liveness violation）是并发编程中的事实，并行流管道也不例外。

考虑条目 45中的程序：

    // Stream-based program to generate the first 20 Mersenne primes
    public static void main(String[] args) {
        primes().map(p -> TWO.pow(p.intValueExact()).subtract(ONE))
            .filter(mersenne -> mersenne.isProbablePrime(50))
            .limit(20)
            .forEach(System.out::println);
    }
    
    static Stream<BigInteger> primes() {
        return Stream.iterate(TWO, BigInteger::nextProbablePrime);
    }
    
在我的机器上，这个程序立即开始打印素数，运行到完成需要12.5秒。
假设我天真地尝试通过向流管道中添加一个到parallel()的调用来加快速度。
你认为它的表现会怎样?它会快几个百分点吗?慢几个百分点?
遗憾的是，它不会打印任何东西，但是CPU使用率会飙升到90%，并且会无限期地停留在那里(liveness failure:活性失败)。
这个程序可能最终会终止，但我不愿意去等待；半小时后我强行阻止了它。

这里发生了什么？简而言之，流类库不知道如何并行化此管道并且启发式失败（heuristics fail.）。
**即使在最好的情况下，如果源来自Stream.iterate方法，或者使用中间操作limit方法，并行化管道也不太可能提高其性能。**
这个管道必须应对这两个问题。
更糟糕的是，默认的并行策略处理不可预测性的limit方法，假设在处理一些额外的元素和丢弃任何不必要的结果时没有害处。
在这种情况下，找到每个梅森素数的时间大约是找到上一个素数的两倍。
因此，计算单个额外元素的成本大致等于计算所有先前元素组合的成本，并且这种无害的管道使自动并行化算法瘫痪。
这个故事的寓意很简单：不要无差别地并行化流管道（stream pipelines）。
性能后果可能是灾难性的。

**通常，并行性带来的性能收益在ArrayList、HashMap、HashSet和ConcurrentHashMap实例、数组、int类型范围和long类型的范围的流上最好。**
这些数据结构的共同之处在于，它们都可以精确而廉价地分割成任意大小的子程序，这使得在并行线程之间划分工作变得很容易。
用于执行此任务的流泪库使用的抽象是spliterator，它由spliterator方法在Stream和Iterable上返回。

所有这些数据结构的共同点的另一个重要因素是它们在顺序处理时提供了从良好到极好的引用位置（ locality of reference）：
顺序元素引用在存储器中存储在一块。 
这些引用所引用的对象在存储器中可能彼此不接近，这降低了引用局部性。 
对于并行化批量操作而言，引用位置非常重要：没有它，线程大部分时间都处于空闲状态，等待数据从内存传输到处理器的缓存中。 
具有最佳引用位置的数据结构是基本类型的数组，因为数据本身连续存储在存储器中。

流管道终端操作的性质也会影响并行执行的有效性。 
如果与管道的整体工作相比，在终端操作中完成了大量的工作，并且这种操作本质上是连续的，那么并行化管道的有效性将是有限的。 
并行性的最佳终操作是缩减（reductions），即使用流的reduce方法组合管道中出现的所有元素，或者预先打包的reduce(如min、max、count和sum)。
短路操作anyMatch、allMatch和noneMatch也可以支持并行性。
由Stream的collect方法执行的操作，称为可变缩减（mutable reductions），不适合并行性，因为组合集合的开销非常大。

如果编写自己的Stream，Iterable或Collection实现，并且希望获得良好的并行性能，则必须重写spliterator方法并广泛测试生成的流的并行性能。 
编写高质量的spliterator很困难，超出了本书的范围。

**并行化一个流不仅会导致糟糕的性能，包括活性失败（liveness failures）;它会导致不正确的结果和不可预知的行为(安全故障)。**
使用映射器（mappers），过滤器（filters）和其他程序员提供的不符合其规范的功能对象的管道并行化可能会导致安全故障。 
Stream规范对这些功能对象提出了严格的要求。 
例如，传递给Stream的reduce方法操作的累加器（accumulator）和组合器（combiner）函数必须是关联的，非干扰的和无状态的。 
如果违反了这些要求（其中一些在第46项中讨论过），但按顺序运行你的管道，则可能会产生正确的结果; 如果将它并行化，它可能会失败，也许是灾难性的。

沿着这些思路，值得注意的是，即使并行的梅森素数程序已经运行完成，它也不会以正确的(升序的)顺序打印素数。
为了保持顺序版本显示的顺序，必须将forEach终端操作替换为forEachOrdered操作，它保证以遇出现顺序（encounter order）遍历并行流。

即使假设正在使用一个高效的可拆分的源流、一个可并行化的或廉价的终端操作以及非干扰的函数对象，也无法从并行化中获得良好的加速效果，除非管道做了足够的实际工作来抵消与并行性相关的成本。
作为一个非常粗略的估计，流中的元素数量乘以每个元素执行的代码行数应该至少是100,000 [Lea14]。

重要的是要记住并行化流是严格的性能优化。 
与任何优化一样，必须在更改之前和之后测试性能，以确保它值得做（第67项）。 
理想情况下，应该在实际的系统设置中执行测试。 通常，程序中的所有并行流管道都在公共fork-join池中运行。 
单个行为不当的管道可能会损害系统中不相关部分的其他行为。

如果在并行化流管道时，这种可能性对你不利，那是因为它们确实存在。
这并不意味着应该避免并行化流。
**在适当的情况下，只需向流管道添加一个parallel方法调用，就可以实现处理器内核数量的近似线性加速。**
某些领域，如机器学习和数据处理，特别适合这些加速。

作为并行性有效的流管道的简单示例，请考虑此函数来计算π(n)，素数小于或等于n：

    // Prime-counting stream pipeline - benefits from parallelization
    static long pi(long n) {
        return LongStream.rangeClosed(2, n)
            .mapToObj(BigInteger::valueOf)
            .filter(i -> i.isProbablePrime(50))
            .count();
    }
    
在我的机器上，使用此功能计算π（108）需要31秒。 
只需添加parallel()方法调用即可将时间缩短为9.2秒：

    // Prime-counting stream pipeline - parallel version
    static long pi(long n) {
        return LongStream.rangeClosed(2, n)
            .parallel()
            .mapToObj(BigInteger::valueOf)
            .filter(i -> i.isProbablePrime(50))
            .count();
    }
    
换句话说，在我的四核计算机上，并行计算速度提高了3.7倍。
值得注意的是，这不是你在实践中如何计算π(n)为n的值。
还有更有效的算法，特别是Lehmer’s formula。

如果要并行化随机数流，请从SplittableRandom实例开始，而不是ThreadLocalRandom（或基本上过时的Random）。 
SplittableRandom专为此用途而设计，具有线性加速的潜力。
ThreadLocalRandom设计用于单个线程，并将自身适应作为并行流源，但不会像SplittableRandom一样快。
Random实例在每个操作上进行同步，因此会导致过度的并行杀死争用（ parallelism-killing contention）。

总之，甚至不要尝试并行化流管道，除非你有充分的理由相信它将保持计算的正确性并提高其速度。
不恰当地并行化流的代价可能是程序失败或性能灾难。
如果您认为并行性是合理的，那么请确保您的代码在并行运行时保持正确，并在实际情况下进行仔细的性能度量。
如果您的代码是正确的，并且这些实验证实了您对性能提高的怀疑，那么并且只有这样才能在生产代码中并行化流。