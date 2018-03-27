---
layout: post
title: Java 8 in Action
date: 2018-3-15 22:15:04
author: admin
comments: true
categories: [Java]
tags: [Java,Java8]
---

# 前言
{:.no_toc}
Java 8 早已发布过去四年，但是发现自己对其新特性还不清楚，所以决定学习一下，顺便做个日志记录一下自己学习过程。

学习资料主要参考： 《Java 8 In Action》、《Java 8实战》，以及其源码：[Java8 In Action](https://github.com/java8/Java8InAction)

<!-- more -->
---
# 目录
{:.no_toc}

* 目录
{:toc}
---

# 第一步： 全书脑图

[![](/images/posts/Java+8+In+Action.png){:width="400" height="300"}](/images/posts/Java+8+In+Action.png)

---

# 第二步：梳理脉络

通过脑图可以看出，全书分为四个部分：
1. 基础知识，重点是**为何关心java8，行为参数化**和**lambda**
2. 函数式编程，重点是全面系统的介绍**Stream**
3. Java8的其他改善点：**重构/测试/调试，默认方法（Default Function），Optional替代null，CompletableFuture 组合式异步编程，日期时间API**
4. Java8之上：对**函数式编程**的思考，函数编程的技巧，与Scala的比较

---
# 第三步：逐步落实

## 1. 基础知识
### 1）为何关心java8
1. 新概念和新功能，有助于写出高效简约的代码
2. 现有的Java编程实践并不能很好地利用多核处理器
3. 借鉴函数式语言的其他优点: 处理null和模式匹配

### 2）行为参数化
#### Why：
应对不断变化的需求，避免啰嗦，而且不打破DRY（Don’t Repeat Yourself）规则。
#### What：

简单讲：把方法（你的代码）作为参数传递给另一个方法。

复杂讲： 让方法接受多种行为（或战
略）作为参数，并在内部使用，来完成不同的行为。

#### How：

Example 1：

用一个Comparator排序Apple，使用Java 8中List默认的sort方法。
```
// java.util.Comparator 
public interface Comparator<T> { 
    public int compare(T o1, T o2); 
} 

// 匿名类写法
inventory.sort(new Comparator<Apple>() {
    public int compare(Apple a1, Apple a2){
        return a1.getWeight().compareTo(a2.getWeight()); 
    } 
});
// lambda写法
inventory.sort(
    (Apple a1, Apple a2) ->
    a1.getWeight().compareTo(a2.getWeight()));
```
Example 2：

用Runnable执行代码块。
```
// java.lang.Runnable 
public interface Runnable{ 
    public void run(); 
}

// 匿名类写法
Thread t = new Thread(new Runnable() { 
    public void run(){ 
        System.out.println("Hello world"); 
    } 
});
// lambda写法
Thread t = new Thread(() -> System.out.println("Hello world"));
```
Example 3：

GUI事件处理。
```
Button button = new Button("Send"); 
// 匿名类写法
button.setOnAction(new EventHandler<ActionEvent>() { 
    public void handle(ActionEvent event) { 
        label.setText("Sent!!"); 
    } 
});
// lambda写法
button.setOnAction((ActionEvent event) -> label.setText("Sent!!"));
```

### 3） 匿名函数 lambda
#### Why：
匿名类太啰嗦
#### What：
简单讲：匿名函数。

复杂讲：简洁地表示可传递的匿名函数的一种方式：它没有名称，但它有参数列表、函数主体、返回类型，可能还有一个可以抛出的异常列表。

- 匿名——不需要明确的名称
- 函数——不需要属于特定的类，但和方法一样，有参数列表、函数主体、返回类型，还可能有可以抛出的异常列表。
- 传递——作为参数传递或存储在变量中。
- 简洁——无需像匿名类那样写很多模板代码。

关键词：
参数列表 + 箭头 + 主体
```
(parameters) -> expression
```
或
```
(parameters) -> { statements; }
```
注意：当主体中出现控制流语句，如return等，要使此Lambda有效，需要使花括号。

#### How：
**函数式接口**： 只定义一个抽象方法的接口。

如：Runable和Comparator。 lambda其实可以看作是函数式接口的实例。

**函数描述符**：函数式接口的抽象方法的签名基本上就是Lambda表达式的签名。

```
() -> void
(Apple) -> int
(Apple, Apple) -> boolean
```
**@FunctionalInterface**：标注用于表示该接口会设计成一个函数式接口。


**Java 8 新的函数式接口**

位置：java.util.function

1. Predicate.test: (T) -> boolean
2. Consumer.accept： (T) -> void
3. Function.apply： (T) -> R
 
#### Detail:
1. 类型检查：上下文中Lambda表达式需要的类型称为目标类型
    1. 同样的Lambda，不同的函数式接口
    2. 特殊的void兼容规则：如果一个Lambda的主体是一个语句表达式 它就和一个返回void的函数描述符兼容。
2. 类型推断：编译器可以了解Lambda表达式的参数类型，这样就可
    以在Lambda语法中省去标注参数类型。
    ```
    int portNumber = 1337; 
    Runnable r = () -> System.out.println(portNumber); 
    ```
3. 使用局部变量：
    ```
    int portNumber = 1337; 
    Runnable r = () -> System.out.println(portNumber); 
    ```
    注意：
    Lambda可以没有限制地捕获（也就是在其主体中引用）实例变量和静态变量。但局部变量必须显式声明为final，或事实上是final。
    
    原因：1）局部变量保存在栈上，并且隐式表示它们仅限于其所在线程，如果允许捕获可改变的局部变量，就会引发造成线程不安全新的可能性；2）不鼓励你使用改变外部变量的典型命令式编程模式
4. 方法引用（method reference）

    目标引用放在分隔符 :: 前, 方法的名称放在后面。
    ```
    inventory.sort(comparing(Apple::getWeight));
    ```
    方法引用主要有三类:
    1. 指向静态方法的方法引用: Integer::parseInt
    2. 指向任意类型实例方法的方法引用: String::length
    3. 指向现有对象的实例方法的方法引用: expensiveTransaction::getValue

    ```
    //改写
    Function<String, Integer> stringToInteger = (String s) -> Integer.parseInt(s);
    Function<String, Integer> stringToInteger = Integer::parseInt;
    ```
    ```
    BiPredicate<List<String>, String> contains = (list, element) -> list.contains(element);
    BiPredicate<List<String>, String> contains = List::contains;
    ```
    构造函数引用： 
    ```
    Supplier<Apple> c1 = Apple::new;
    Apple a1 = c1.get();
    ```
    ```
    Function<Integer, Apple> c2 = Apple::new;
    Apple a2 = c2.apply(110);
    ```
5. 复合Lambda表达式 (因为引入了默认方法)
    1. 比较器复合
    ```
    Comparator<Apple> c = Comparator.comparing(Apple::getWeight);
    // 逆序
    inventory.sort(comparing(Apple::getWeight).reversed());
    // 比较器链
    inventory.sort(comparing(Apple::getWeight)
        .reversed()
        .thenComparing(Apple::getCountry))
    ```
    2. 谓词复合：negate、and和or
    ```
    //取非
    Predicate<Apple> notRedApple = redApple.negate();
    //and操作
    Predicate<Apple> redAndHeavyApple = 
        redApple.and(a -> a.getWeight() > 150);
    //and + or操作
    Predicate<Apple> redAndHeavyAppleOrGreen = 
        redApple.and(a -> a.getWeight() > 150)
        .or(a -> "green".equals(a.getColor()));
    ```
    *注意：从左向右确定优先级，如a.or(b).and(c)可以看做 (a || b) && c*
    3. 函数复合:Function提供了andThen(), compose()。
    ```
    Function<Integer, Integer> f = x -> x + 1; 
    Function<Integer, Integer> g = x -> x * 2; 
    // expect: (2 + 1) * 2 = 4
    // f(g(x))
    System.out.println(f.andThen(g).apply(1));
    // expect: 1 * 2 + 1 = 3
    // g(f(x))
    System.out.println(f.compose(g).apply(1));
    ```
    *复合Lambda表达式可以用来创建各种转型流水线。*
    
## 2. 函数式编程
### 1）介绍stream
#### Why:
以声明性方式处理数据集合, 遍历数据集的高级迭代器
特点：
1. 声明性：更简洁,更易读
2. 可复合：更灵活
3. 可并行（parallelStream）：性能更好

#### What:
定义：从（支持**数据处理操作**的**源**）生成的（**元素序列**）
- 元素序列：流提供了一个接口,可以访问特定元素类型的一组有序值。
- 源：集合、数组或输入/输出资源
- 数据处理操作：filter 、 map 、 reduce 、 find 、 match 、 sort等，可顺序，可并行。

两个重要特点：
- 流水线：多个操作可以链接起来
- 内部迭代：流的迭代操作是在背后进行的，优点：透明地并行处理;优化处理顺序

注意：链中的方法调用都在排队等待,直到调用 collect 。

#### collection vs stream
粗略地说区别在于什么时候进行计算。
stream： 按需生成，需求驱动；只能遍历一次; 内部迭代
collection：急切创建；外部迭代

#### How
两大类操作
1. 中间操作：会返回另一个流，map，filter等
2. 终端操作：从流的流水线生成结果，collect，foreach， count等

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
### 2）使用stream
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
### 3）用stream收集数据

1. 常用
- Collectors.groupingBy
- Collectors.counting()
- Collectors.maxBy
- Collectors.minBy
- Collectors.summingInt，Collectors.summingLong，Collectors.summingDouble
- Collectors.averagingInt，Collectors.averagingLong，Collectors.averagingDouble
- Collectors.summarizingInt
- Collectors.joining

2. 广义的归约汇总 Collectors.reducing

- 第一个参数是归约操作的起始值
- 第二个参数获取或操作对象的属性数值(转换函数)
- 第三个参数BinaryOperator，如加法
```
int totalCalories = menu.stream().collect(reducing(
    0, 
    Dish::getCalories, 
    (i, j) -> i + j));
```
思考：Stream 接口的 collect和 reduce 方法有何不同？
- 语义问题: reduce 方法旨在把两个值结合起来生成一个新值,它是一个不可变的归约。与此相反, collect 方法的设计就是要改变容器,从而累积要输出的结果。
- 实际问题: 以错误的语义使用 reduce 方法不能并行工作

3. 分组

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
    多级分组
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
    按子组收集数据
    
    ```
    Map<Dish.Type, Long> typesCount = menu.stream().collect(
        groupingBy(Dish::getType, counting()));
    
    ```    
    Collectors.collectingAndThen： 把收集器返回的结果转换为另一种类型
    ```
    Map<Dish.Type, Dish> mostCaloricByType =
        menu.stream()
            .collect(groupingBy(Dish::getType,
                collectingAndThen(
                    maxBy(comparingInt(Dish::getCalories)),
                Optional::get)));
    ```
    与 groupingBy 联合使用的其他收集器的例子
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
    
    ```
4. 分区
    
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
5. Collector 接口
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
    
    方法分析：
    1. 建立新的结果容器: supplier 方法
    2. 将元素添加到结果容器: accumulator 方法
    3. 对结果容器应用最终转换: finisher 方法
    4. 合并两个结果容器: combiner 方法（并行归约）
    5. characteristics 方法：返回一个不可变的 Characteristics 集合
        - UNORDERED：归约结果不受流中项目的遍历和累积顺序的影响
        - CONCURRENT：accumulator函数可以从多个线程同时调用,且该收集器可以并行归约流。如果收集器没有标为UNORDERED,那它仅在用于无序数据源时才可以并行归约
        - IDENTITY_FINISH：表明完成器方法返回的函数是一个恒等函数
        
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
6. 自定义收集器

    · 必要时，可以根据自己需求实现收集器
    · 可以获取更好的性能

### 4）并行数据处理与性能
1. 并行流处理数据

    What：parallelStream
    
    对顺序流调用parallel方法，对并行流调用sequential方法
    ```
    stream.parallel() 
        .filter(...) 
        .sequential() 
        .map(...) 
        .parallel() 
        .reduce(); 
    ```
    注意点：
    - 保证在内核中并行执行工作的时间比在内核之间传输数据的时间长。
    - 避免改变了某些共享状态
    
    如何高效使用：
    - 测量
    - 留意装箱
    - 依赖于元素顺序的操作，本身在并行流上的性能就比顺序流差
    - 流的操作流水线的总计算成本
    - 数据少
    - 数据结构是否易于分解
    - 流自身的特点，以及流水线中的中间操作修改流的方式，都可能会改变分解过程的性能
    - 终端操作中合并步骤的代价是大是小

2. 分支/合并框架(The fork/join framework)
    
    **目的**：以递归方式将可以并行的任务拆分成更小的任务，然后将每个子任务的结果合并起来生成整体结果。（先拆，并行处理，合并结果）
    
    **定义**：它是ExecutorService接口的一个实现，它把子任务分配给线程池（称为ForkJoinPool）中的工作线程。

    **使用**：实现compute()方法，提交至ForkJoinPool.invoke
    
    **好的做法**：
    - join方法会阻塞，所以先确保两个子任务全部启动，再调用join
    - RecursiveTask内部不应该调用ForkJoinPool.invoke，应该直接调用compute、fork，只有顺序代码才应该用 invoke 来启动并行计算
    - 一边子任务fork，一边子任务compute，避免在线程池中多分配一个任务造成的开销
    - 调试使用分支/合并框架的并行计算可能有点棘手, 调用compute的线程并不是概念上的调用方(即调用fork的那个).
    - 不应理所当然地认为在多核处理器上使用分支/合并框架就比顺序计算快。
    
    **工作窃取（work stealing）**
    
    目的：为解决因为每个子任务所花的时间可能天差地别而造成的效率低下。
    过程：线程把任务保存到一个双向链式队列，当一个线程的队列空了，它就随机从其他线程的队列尾部“偷”一个任务执行

3. Spliterator分割流

    描述：一种自动机制来拆分流。新的接口“可分迭代器”（splitable iterator）。
    ```
    public interface Spliterator<T> { 
        boolean tryAdvance(Consumer<? super T> action); 
        Spliterator<T> trySplit(); 
        long estimateSize(); 
        int characteristics(); 
    }
    ```
    1. 拆分过程
    
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

    2. 各个函数及作用

        函数名 | 作用
        ---|---
        tryAdvance | 执行一个操作给传入的元素，并且返回一个boolean，来表示是否有剩余元素需要处理
        trySplit | 最重要的函数，如果数据可以继续分割，返回一个Spliterator，否则返回null
        estimateSize | 返回一个对剩余元素数量的估值
        characteristics | 设置Spliterator的某些特性，参考上表。