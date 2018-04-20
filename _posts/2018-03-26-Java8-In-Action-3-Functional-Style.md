---
layout: post
title: 《Java 8 in Action》学习日志（三）：函数式数据处理
date: 2018-3-26 22:15:04
author: admin
comments: true
categories: [Java]
tags: [Java,Java8]
---


第三部分主要介绍了Java8的函数式编程：Stream，它是超越for-each的高级迭代器，它天生可以利用多核cpu，比起之前写法更简洁更易读。

<!-- more -->

学习资料主要参考： 《Java 8 In Action》、《Java 8实战》，以及其源码：[Java8 In Action](https://github.com/java8/Java8InAction)

---
# 目录
{:.no_toc}

* 目录
{:toc}
---

# 介绍stream

<br>

## 三个问号
### 1. Why

以声明性方式处理数据集合, 遍历数据集的高级迭代器

特点：

1. 声明性：更简洁,更易读
2. 可复合：更灵活
3. 可并行（parallelStream）：性能更好

### 2. What

定义：从（支持**数据处理操作**的**源**）生成的（**元素序列**）

- 元素序列：流提供了一个接口,可以访问特定元素类型的一组有序值。
- 源：集合、数组或输入/输出资源
- 数据处理操作：filter 、 map 、 reduce 、 find 、 match 、 sort等，可顺序，可并行。

两个重要特点：

- 流水线：多个操作可以链接起来
- 内部迭代：流的迭代操作是在背后进行的，优点：透明地并行处理;优化处理顺序

注意：链中的方法调用都在排队等待,直到调用 collect 。

#### collection vs stream

- 粗略地说区别在于什么时候进行计算。
- stream： 按需生成，需求驱动；只能遍历一次; 内部迭代
- collection：急切创建；外部迭代

### 3. How

两大类操作

1. *中间操作*：会返回另一个流，map，filter等
2. *终端操作*：从流的流水线生成结果，collect，foreach， count等

使用三要素

- 一个数据源(如集合)来执行一个查询;
- 一个中间操作链,形成一条流的流水线;
- 一个终端操作,执行流水线,并能生成结果。

```
dishes.stream()
    .filter(d -> d.getCalories() < 400)
    .sorted(comparing(Dish::getCalories))
    .map(Dish::getName)
    .collect(toList());
```

---
# 使用stream

Java 8 提供了很多现有的方法来面对不同的需求，以下是一些常用函数。

1. 筛选和切片 Filtering and slicing
    
    filter()，distinct()，limit(n), skip(n)

2. 映射 Mapping

    1. map: 对流中每一个元素应用函数
    2. flatmap: 把一个流中的每个值都换成另一个流,然后把所有的流连接起来成为一个流。

3. 查找和匹配 Finding and matching

    1. anyMatch: 流中是否有一个元素能匹配给定的谓词, 方法返回一个 boolean
    2. allMatch: 流中的元素是否都能匹配给定的谓词, 方法返回一个 boolean
    3. noneMatch: 流中没有任何元素与给定的谓词匹配
    4. findAny: 返回当前流中的任意元素(Optional<T>)
    5. findFirst: 找到第一个元素

4. 归约 Reducing

    将流中所有元素反复结合起来。

    1. 元素求和
    ```
    int sum = numbers.stream().reduce(0, Integer::sum);
    Optional<Integer> sum = numbers.stream().reduce(Integer::sum);
    ```
    2. 最大值，最小值
    ```
    Optional<Integer> max = numbers.stream().reduce(Integer::max);
    Optional<Integer> min = numbers.stream().reduce(Integer::min);
    ```

5. 数值流 Numeric Streams
    
    为了避免装箱带来的复杂性

    1. 映射到数值流: mapToInt、mapToDouble 和 mapToLong
    2. 转换回对象流
    ```
    IntStream intStream = menu.stream().mapToInt(Dish::getCalories);
    Stream<Integer> stream = intStream.boxed();
    ```
    3. 默认值 OptionalInt 、 OptionalDouble 和 OptionalLong
    4. 数值范围 IntStream.rangeClosed(1, 100)
    
6. 构建流 Building streams

    1. 由值创建流： 
    ```
    Stream<String> stream = Stream.of("Java 8 ", "Lambdas ", "In ", "Action");
    stream.map(String::toUpperCase).forEach(System.out::println);
    //空流
    Stream<String> emptyStream = Stream.empty();
    ```
    2. 由数组创建流
    ```
    int[] numbers = {2, 3, 5, 7, 11, 13};
    int sum = Arrays.stream(numbers).sum();
    ```
    3. 由文件生成流
    ```
    long uniqueWords = Files
				.lines(Paths.get("data.txt"),
						Charset.defaultCharset())
				.flatMap(line -> Arrays.stream(line.split(" "))).distinct()
				.count();
    ```
    4. 由函数生成流:创建无限流
    ```
    // iterate
    Stream.iterate(0, n -> n + 2)
        .limit(10)
        .forEach(System.out::println);
    // generate
    Stream.generate(Math::random)
        .limit(5)
        .forEach(System.out::println);
    ```

---

# 用stream收集数据

## 常用的收集器

- Collectors.groupingBy
- Collectors.counting()
- Collectors.maxBy
- Collectors.minBy
- Collectors.summingInt，Collectors.summingLong，Collectors.summingDouble
- Collectors.averagingInt，Collectors.averagingLong，Collectors.averagingDouble
- Collectors.summarizingInt
- Collectors.joining

## 详细介绍

### 1. 广义的归约汇总 Collectors.reducing
需要三个参数：
1. 归约操作的起始值
2. 获取或操作对象的属性数值(转换函数)
3. BinaryOperator，如加法

例子：
```
int totalCalories = menu.stream().collect(reducing(
    0,                      // 归约操作的起始值
    Dish::getCalories,      // 获取或操作对象的属性数值(转换函数)
    (i, j) -> i + j));      // BinaryOperator，如加法
```

思考：Stream 接口的 collect和 reduce 方法有何不同？
- 语义问题: reduce 方法旨在把两个值结合起来生成一个新值,它是一个不可变的归约。与此相反, collect 方法的设计就是要改变容器,从而累积要输出的结果。
- 实际问题: 以错误的语义使用 reduce 方法不能并行工作

### 2. 分组 Collectors.groupingBy

#### 一级分组
```
Map<Dish.Type, List<Dish>> dishesByType =
menu.stream().collect(groupingBy(Dish::getType));
// 自定义分组
public enum CaloricLevel { DIET, NORMAL, FAT }
Map<CaloricLevel, List<Dish>> dishesByCaloricLevel = menu.stream()
                .collect(groupingBy(dish -> {
                    if (dish.getCalories() <= 400)
                        return CaloricLevel.DIET;
                    else if (dish.getCalories() <= 700)
                        return CaloricLevel.NORMAL;
                    else
                        return CaloricLevel.FAT;
                }));

```
#### 多级分组
```
menu.stream().collect(
            groupingBy(Dish::getType,
                    groupingBy((Dish dish) -> {
                        if (dish.getCalories() <= 400) return CaloricLevel.DIET;
                        else if (dish.getCalories() <= 700) return CaloricLevel.NORMAL;
                        else return CaloricLevel.FAT;
                    } )
            );

```

#### 与 groupingBy 联合使用的其他收集器

有时候在groupBy的时候，我们还想做一下其他操作，比如设定返回类型，或者只取对象中的某个属性。
```
// summingInt
Map<Dish.Type, Integer> totalCaloriesByType =
    menu.stream().collect(groupingBy(Dish::getType,
        summingInt(Dish::getCalories)));

// mapping
menu.stream().collect(
    groupingBy(Dish::getType, mapping(dish -> {
        if (dish.getCalories() <= 400)
            return CaloricLevel.DIET;
        else if (dish.getCalories() <= 700)
            return CaloricLevel.NORMAL;
        else
            return CaloricLevel.FAT;
    }, toSet())));

//按子组收集数据
Map<Dish.Type, Long> typesCount = menu.stream().collect(
    groupingBy(Dish::getType, counting()));

//Collectors.collectingAndThen： 把收集器返回的结果转换为另一种类型
Map<Dish.Type, Dish> mostCaloricByType =
    menu.stream()
        .collect(groupingBy(Dish::getType,
            collectingAndThen(
                maxBy(comparingInt(Dish::getCalories)),
            Optional::get)));
```

### 4. 分区
    
好处：保留了分区函数返回 true 或 false 的两套流元素列表。

与groupby的区别：需要一个谓词（返回一个布尔值的函数）

```
Map<Boolean, List<Dish>> partitionedMenu =
    menu.stream().collect(partitioningBy(Dish::isVegetarian));
//二级分区
menu.stream().collect(partitioningBy(Dish::isVegetarian,
    partitioningBy (d -> d.getCalories() > 500)));
//联合其他收集器
menu.stream().collect(partitioningBy(Dish::isVegetarian,
    counting()));
```

### 5. Collector 接口

#### 基本定义：
```
public interface Collector<T, A, R> {
    Supplier<A> supplier();
    BiConsumer<A, T> accumulator();
    Function<A, R> finisher();
    BinaryOperator<A> combiner();
    Set<Characteristics> characteristics();
}
```
<T, A, R> 意义：

- T 是流中要收集的项目的泛型
- A 是累加器的类型,累加器是在收集过程中用于累积部分结果的对象。
- R 是收集操作得到的对象(通常但并不一定是集合)的类型。

#### 方法分析：

1. 建立新的结果容器: supplier 方法
2. 将元素添加到结果容器: accumulator 方法
3. 对结果容器应用最终转换: finisher 方法
4. 合并两个结果容器: combiner 方法（并行归约）
5. characteristics 方法：返回一个不可变的 Characteristics 集合
    - UNORDERED：归约结果不受流中项目的遍历和累积顺序的影响
    - CONCURRENT：accumulator函数可以从多个线程同时调用,且该收集器可以并行归约流。如果收集器没有标为UNORDERED,那它仅在用于无序数据源时才可以并行归约
    - IDENTITY_FINISH：表明完成器方法返回的函数是一个恒等函数




### 6. 自定义收集器

必要时，可以根据自己需求实现收集器， 来避免一些不必要的操作（如装箱拆箱），这样子可以获取更好的性能。

例子如下：
```
import java.util.*;
import java.util.function.*;
import java.util.stream.Collector;
import static java.util.stream.Collector.Characteristics.*;

public class ToListCollector<T> implements Collector<T, List<T>, List<T>> {

    @Override
    public Supplier<List<T>> supplier() {
        return () -> new ArrayList<T>();
    }

    @Override
    public BiConsumer<List<T>, T> accumulator() {
        return (list, item) -> list.add(item);
    }

    @Override
    public Function<List<T>, List<T>> finisher() {
        return i -> i;
    }

    @Override
    public BinaryOperator<List<T>> combiner() {
        return (list1, list2) -> {
            list1.addAll(list2);
            return list1;
        };
    }

    @Override
    public Set<Characteristics> characteristics() {
        return Collections.unmodifiableSet(EnumSet.of(IDENTITY_FINISH, CONCURRENT));
    }
}
```
---

# 并行数据处理与性能

## 1. 并行流处理数据

### 简单了解

java 8 中提供了现成的并行处理流，即parallelStream。

#### 并行流与顺序流的转换

对顺序流调用parallel方法，对并行流调用sequential方法。在合适的时候顺序流与并行流相互转换，可以提高效率。
```
stream.parallel()
    .filter(...)
    .sequential()
    .map(...)
    .parallel()
    .reduce();
```

**注意点**：
- 保证在内核中并行执行工作的时间比在内核之间传输数据的时间长。
- 避免改变了某些共享状态

### 配置并行流使用的线程池

并行流内部使用了默认的 ForkJoinPool， 它默认的线程数量就是你的处理器数量, 这个值是由Runtime.getRuntime().availableProcessors() 得到的。

可以通过系统属性java.util.concurrent.ForkJoinPool.common.parallelism 来改变线程池大小：
```java
System.setProperty("java.util.concurrent.ForkJoinPool.common.parallelism","12");
```

### 如何高效使用：

- 测量
- 留意装箱
- 依赖于元素顺序的操作，本身在并行流上的性能就比顺序流差
- 流的操作流水线的总计算成本
- 数据少
- 数据结构是否易于分解
- 流自身的特点，以及流水线中的中间操作修改流的方式，都可能会改变分解过程的性能
- 终端操作中合并步骤的代价是大是小

## 2. 分支/合并框架(The fork/join framework)
    
### 目的

以递归方式将可以并行的任务拆分成更小的任务，然后将每个子任务的结果合并起来生成整体结果。（先拆，并行处理，合并结果）

### 定义

RecursiveTask是ExecutorService接口的一个实现，它把子任务分配给线程池（称为ForkJoinPool）中的工作线程。

### 使用

实现compute()方法，提交至ForkJoinPool.invoke

实际例子：
```java
public class ForkJoinSumCalculator extends java.util.concurrent.RecursiveTask<Long> {

    // 拆分任务的标准大小
    public static final long THRESHOLD = 10_000;

    private final long[] numbers;
    private final int start;
    private final int end;

    public ForkJoinSumCalculator(long[] numbers) {
        this(numbers, 0, numbers.length);
    }

    private ForkJoinSumCalculator(long[] numbers, int start, int end) {
        this.numbers = numbers;
        this.start = start;
        this.end = end;
    }

    // 实现compute方法
    @Override
    protected Long compute() {
        int length = end - start; // 获取当前剩余任务的大小
        if (length <= THRESHOLD) {
            return computeSequentially();
        }
        // 创建另外一个子任务 leftTask
        ForkJoinSumCalculator leftTask = new ForkJoinSumCalculator(numbers, start, start + length/2);
        // 异步执行 leftTask
        leftTask.fork();
        // 创建剩余一半任务的子任务 rightTask
        ForkJoinSumCalculator rightTask = new ForkJoinSumCalculator(numbers, start + length/2, end);
        // 递归调用获取结果
        Long rightResult = rightTask.compute();
        // 获取 leftTask 结果
        Long leftResult = leftTask.join();
        return leftResult + rightResult;
    }

    private long computeSequentially() {
        long sum = 0;
        for (int i = start; i < end; i++) {
            sum += numbers[i];
        }
        return sum;
    }

    // 如何调用fork/join框架
    public static long forkJoinSum(long n) {
        long[] numbers = LongStream.rangeClosed(1, n).toArray();
        ForkJoinTask<Long> task = new ForkJoinSumCalculator(numbers);
        return new ForkJoinPool().invoke(task);
    }
}
```

### 好的做法

- join方法会阻塞，所以先确保两个子任务全部启动，再调用join
- RecursiveTask内部不应该调用ForkJoinPool.invoke，应该直接调用compute、fork，只有顺序代码才应该用 invoke 来启动并行计算
- 一边子任务fork，一边子任务compute，避免在线程池中多分配一个任务造成的开销
- 调试使用分支/合并框架的并行计算可能有点棘手, 调用compute的线程并不是概念上的调用方(即调用fork的那个).
- 不应理所当然地认为在多核处理器上使用分支/合并框架就比顺序计算快。

**工作窃取（work stealing）**

目的：为解决因为每个子任务所花的时间可能天差地别而造成的效率低下。
过程：线程把任务保存到一个双向链式队列，当一个线程的队列空了，它就随机从其他线程的队列尾部“偷”一个任务执行

### 思考：
问：都是拆分任务，并行执行，为什么不使用线程池，如ThreadPoolExecutor呢？

答：

Thread pool 默认期望它们所有执行的任务都是不相关的，可以尽可能的并行执行。

而fork join框架解决的问题，是一个全局问题，所有子任务拆分运行后的结果，是要合并起来的。

另外，fork-join pool另一个特点work stealing，如果用ThreadPoolExecutor实现是比较麻烦的。

## 3. Spliterator分割流

### 简单了解

描述：一种自动机制来拆分流。新的接口“可分迭代器”（splitable iterator）

```
public interface Spliterator<T> {
    boolean tryAdvance(Consumer<? super T> action);
    Spliterator<T> trySplit();
    long estimateSize();
    int characteristics();
}
```

### 拆分过程

递归过程。框架不断对Spliterator调用trySplit直到它返回null,表明它处理的数据结构不能再分割。

**Spliterator 的特性**

特性 | 含义
---|---
ORDERED | 元素有既定的顺序(例如 List ),因此 Spliterator 在遍历和划分时也会遵循这一顺序
DISTINCT | 对于任意一对遍历过的元素 x 和 y , x.equals(y) 返回 false
SORTED | 遍历的元素按照一个预定义的顺序排序
SIZED | 该 Spliterator 由一个已知大小的源建立(例如 Set ),因此 estimatedSize() 返回的是准确值
NONNULL | 保证遍历的元素不会为 null
IMMUTABL | Spliterator 的数据源不能修改。这意味着在遍历时不能添加、删除或修改任何元素E
CONCURRENT | 该 Spliterator 的数据源可以被其他线程同时修改而无需同步
SUBSIZED | 该 Spliterator 和所有从它拆分出来的 Spliterator 都是 SIZED

### 各个函数及作用

函数名 | 作用
---|---
tryAdvance | 执行一个操作给传入的元素，并且返回一个boolean，来表示是否有剩余元素需要处理
trySplit | 最重要的函数，如果数据可以继续分割，返回一个Spliterator，否则返回null
estimateSize | 返回一个对剩余元素数量的估值
characteristics | 设置Spliterator的某些特性，参考上表。