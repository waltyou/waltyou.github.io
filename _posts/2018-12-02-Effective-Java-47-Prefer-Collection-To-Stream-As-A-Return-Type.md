---
layout: post
title: 《Effective Java》学习日志（六）47：比起Stream，偏爱Collection来作为方法的返回类型
date: 2018-12-02 13:10:04
author: admin
comments: true
categories: [Java]
tags: [Java,Effective Java]
---



<!-- more -->

---

学习资料主要参考： 《Effective Java Third Edition》，作者：Joshua Bloch

---

## 目录
{:.no_toc}

* 目录
{:toc}

---

许多方法返回元素序列（sequence）。
在Java 8之前，通常方法的返回类型是Collection，Set和List这些接口；还包括Iterable和数组类型。
通常，很容易决定返回哪一种类型。
规范（norm）是集合接口。
如果该方法仅用于启用for-each循环，或者返回的序列不能实现某些Collection方法(通常是contains(Object))，则使用迭代（Iterable）接口。
如果返回的元素是基本类型或有严格的性能要求，则使用数组。
在Java 8中，将流（Stream）添加到平台中，这使得为序列返回方法选择适当的返回类型的任务变得非常复杂。

你可能听说过，流现在是返回元素序列的明显的选择，但是正如Item 45所讨论的，流不会使迭代过时：编写好的代码需要明智地结合流和迭代。
如果一个API只返回一个流，并且一些用户想用for-each循环遍历返回的序列，那么这些用户肯定会感到不安。
这尤其令人沮丧，因为Stream接口在Iterable接口中包含唯一的抽象方法，Stream的方法规范与Iterable兼容。
阻止程序员使用for-each循环在流上迭代的唯一原因是Stream无法继承Iterable。

遗憾的是，这个问题没有好的解决方法。 
乍一看，似乎可以将方法引用传递给Stream的iterator方法。 
结果代码可能有点嘈杂和不透明，但并非不合理：

    // Won't compile, due to limitations on Java's type inference
    for (ProcessHandle ph : ProcessHandle.allProcesses()::iterator) {
        // Process the process
    }
    
不幸的是，如果你试图编译这段代码，会得到一个错误信息:

    Test.java:6: error: method reference not expected here
    for (ProcessHandle ph : ProcessHandle.allProcesses()::iterator) {
                            ^
                            
为了使代码编译，必须将方法引用强制转换为适当参数化的Iterable类型：

    // Hideous workaround to iterate over a stream
    for  (ProcessHandle ph : (Iterable<ProcessHandle>)
                             ProcessHandle.allProcesses()::iterator)
    
此代码有效，但在实践中使用它太嘈杂和不透明。 
更好的解决方法是使用适配器方法。 
JDK没有提供这样的方法，但是使用上面的代码片段中使用的相同技术，很容易编写一个方法。 
请注意，在适配器方法中不需要强制转换，因为Java的类型推断在此上下文中能够正常工作：

    // Adapter from  Stream<E> to Iterable<E>
    public static <E> Iterable<E> iterableOf(Stream<E> stream) {
        return stream::iterator;
    }
    
使用此适配器，可以使用for-each语句迭代任何流：

    for (ProcessHandle p : iterableOf(ProcessHandle.allProcesses())) {
        // Process the process
    }
    
注意，Item 34中的Anagrams程序的流版本使用Files.lines方法读取字典，而迭代版本使用了scanner。
Files.lines方法优于scanner，scanner在读取文件时无声地吞噬所有异常。
理想情况下，我们也会在迭代版本中使用Files.lines。
如果API只提供对序列的流访问，而程序员希望使用for-each语句遍历序列，那么他们就要做出这种妥协。

相反，如果一个程序员想要使用流管道来处理一个序列，那么一个只提供Iterable的API会让他感到不安。
JDK同样没有提供适配器，但是编写这个适配器非常简单:

    // Adapter from Iterable<E> to Stream<E>
    public static <E> Stream<E> streamOf(Iterable<E> iterable) {
        return StreamSupport.stream(iterable.spliterator(), false);
    }
    
如果你正在编写一个返回对象序列的方法，并且它只会在流管道中使用，那么当然可以自由地返回流。
类似地，返回仅用于迭代的序列的方法应该返回一个Iterable。
但是如果你写一个公共API，它返回一个序列，你应该为用户提供哪些想写流管道，哪些想写for-each语句，除非你有充分的理由相信大多数用户想要使用相同的机制。

Collection接口是Iterable的子类型，并且具有stream方法，因此它提供迭代和流访问。 
因此，Collection或适当的子类型通常是公共序列返回方法的最佳返回类型。 
数组还使用Arrays.asList和Stream.of方法提供简单的迭代和流访问。 
如果返回的序列小到足以容易地放入内存中，那么最好返回一个标准集合实现，例如ArrayList或HashSet。 
但是不要在内存中存储大的序列，只是为了将它作为集合返回。

如果返回的序列很大但可以简洁地表示，请考虑实现一个专用集合。 
例如，假设返回给定集合的幂集（power set：就是原集合中所有的子集（包括全集和空集）构成的集族），该集包含其所有子集。 
{a，b，c}的幂集为{{}，{a}，{b}，{c}，{a，b}，{a，c}，{b，c}，{a，b ， C}}。 如果一个集合具有n个元素，则幂集具有2n个。 
因此，你甚至不应考虑将幂集存储在标准集合实现中。 
但是，在AbstractList的帮助下，很容易为此实现自定义集合。

诀窍是使用幂集中每个元素的索引作为位向量（bit vector），其中索引中的第n位指示源集合中是否存在第n个元素。 
本质上，从0到2n-1的二进制数和n个元素集和的幂集之间存在自然映射。 

这是代码：

    // Returns the power set of an input set as custom collection
    public class PowerSet {
       public static final <E> Collection<Set<E>> of(Set<E> s) {
          List<E> src = new ArrayList<>(s);
          if (src.size() > 30)
             throw new IllegalArgumentException("Set too big " + s);
          return new AbstractList<Set<E>>() {
             @Override public int size() {
                return 1 << src.size(); // 2 to the power srcSize
             }
             @Override public boolean contains(Object o) {
                return o instanceof Set && src.containsAll((Set)o);
             }
             @Override public Set<E> get(int index) {
                Set<E> result = new HashSet<>();
                for (int i = 0; index != 0; i++, index >>= 1)
                   if ((index & 1) == 1)
                      result.add(src.get(i));
                return result;
             }
          };
       }
    }
    
请注意，如果输入集合超过30个元素，则PowerSet.of方法会引发异常。 
这突出了使用Collection作为返回类型而不是Stream或Iterable的缺点：Collection有int返回类型的size的方法，该方法将返回序列的长度限制为Integer.MAX_VALUE或231-1。
Collection规范允许size方法返回231 - 1，如果集合更大，甚至无限，但这不是一个完全令人满意的解决方案。

为了在AbstractCollection上编写Collection实现，除了Iterable所需的方法之外，只需要实现两种方法：contains和size。 
通常，编写这些方法的有效实现很容易。 
如果不可行，可能是因为在迭代发生之前未预先确定序列的内容，返回Stream还是Iterable的，无论哪种感觉更自然。 
如果选择，可以使用两种不同的方法分别返回。

有时，你会仅根据实现的易用性选择返回类型。
例如，假设希望编写一个方法，该方法返回输入列表的所有(连续的)子列表。
生成这些子列表并将它们放到标准集合中只需要三行代码，但是保存这个集合所需的内存是源列表大小的二次方。
虽然这没有指数幂集那么糟糕，但显然是不可接受的。
实现自定义集合(就像我们对幂集所做的那样)会很乏味，因为JDK缺少一个框架Iterator实现来帮助我们。

然而，实现输入列表的所有子列表的流是直截了当的，尽管它确实需要一点的洞察力（insight）。 
让我们调用一个子列表，该子列表包含列表的第一个元素和列表的前缀。 
例如，（a，b，c）的前缀是（a），（a，b）和（a，b，c）。 
类似地，让我们调用包含后缀的最后一个元素的子列表，因此（a，b，c）的后缀是（a，b，c），（b，c）和（c）。 
洞察力是列表的子列表只是前缀的后缀（或相同的后缀的前缀）和空列表。 
这一观察直接展现了一个清晰，合理简洁的实现：

    // Returns a stream of all the sublists of its input list
    public class SubLists {
       public static <E> Stream<List<E>> of(List<E> list) {
          return Stream.concat(Stream.of(Collections.emptyList()),
             prefixes(list).flatMap(SubLists::suffixes));
       }
       private static <E> Stream<List<E>> prefixes(List<E> list) {
          return IntStream.rangeClosed(1, list.size())
             .mapToObj(end -> list.subList(0, end));
       }
       private static <E> Stream<List<E>> suffixes(List<E> list) {
          return IntStream.range(0, list.size())
             .mapToObj(start -> list.subList(start, list.size()));
       }
    }
    
请注意，Stream.concat方法用于将空列表添加到返回的流中。 
还有，flatMap方法（Item 45）用于生成由所有前缀的所有后缀组成的单个流。 
最后，通过映射IntStream.range和IntStream.rangeClosed返回的连续int值流来生成前缀和后缀。
这个习惯用法，粗略地说，流等价于整数索引上的标准for循环。
因此，我们的子列表实现似于明显的嵌套for循环:

    for (int start = 0; start < src.size(); start++)
        for (int end = start + 1; end <= src.size(); end++)
            System.out.println(src.subList(start, end));
            
可以将这个for循环直接转换为流。
结果比我们以前的实现更简洁，但可能可读性稍差。
它类似于Item 45中的笛卡尔积的使用流的代码:

    // Returns a stream of all the sublists of its input list
    public static <E> Stream<List<E>> of(List<E> list) {
       return IntStream.range(0, list.size())
          .mapToObj(start ->
             IntStream.rangeClosed(start + 1, list.size())
                .mapToObj(end -> list.subList(start, end)))
          .flatMap(x -> x);
    }
    
与之前的for循环一样，此代码不会包换空列表。 
为了解决这个问题，可以使用concat方法，就像我们在之前版本中所做的那样，或者在rangeClosed调用中用(int) Math.signum(start)替换1。

这两种子列表的流实现都可以，但都需要一些用户使用流-迭代适配器( Stream-to-Iterable adapte)，或者在更自然的地方使用流。
流-迭代适配器不仅打乱了客户端代码，而且在我的机器上使循环速度降低了2.3倍。
一个专门构建的Collection实现(此处未显示)要冗长，但运行速度大约是我的机器上基于流的实现的1.4倍。

总之，在编写返回元素序列的方法时，请记住，某些用户可能希望将它们作为流处理，而其他用户可能希望迭代方式来处理它们。 
尽量适应两个群体。 
如果返回集合是可行的，请执行此操作。 
如果已经拥有集合中的元素，或者序列中的元素数量足够小，可以创建一个新的元素，那么返回一个标准集合，比如ArrayList。 
否则，请考虑实现自定义集合，就像我们为幂集程序里所做的那样。 
如果返回集合是不可行的，则返回流或可迭代的，无论哪个看起来更自然。 
如果在将来的Java版本中，Stream接口声明被修改为继承Iterable，那么应该随意返回流，因为它们将允许流和迭代处理。