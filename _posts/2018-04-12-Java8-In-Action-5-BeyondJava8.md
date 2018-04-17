---
layout: post
title: 《Java 8 in Action》学习日志（五）：超越Java 8
date: 2018-4-12 10:30:04
author: admin
comments: true
categories: [Java]
tags: [Java,Java8]
---


<!-- more -->

学习资料主要参考： 《Java 8 In Action》、《Java 8实战》，以及其源码：[Java8 In Action](https://github.com/java8/Java8InAction)

---
# 目录
{:.no_toc}

* 目录
{:toc}
---


# 函数式编程的思考

## 1. 实现和维护系统

实现和维护系统时，大多数程序员最关心是什么？

是代码遭遇一些无法预期的值就有可能发生崩溃，换句话说是我们无法预知的变量修改问题。

而这些问题都源于**共享的数据结构**被你所维护的代码中的多个方法读取和更新。

### 1）共享的可变数据

正是由于使用了可变的共享数据结构，我们很难追踪你程序的各个组成部分所发生的变化。

如果一个方法既不修改它内嵌类的状态，也不修改其他对象的状态，使用return返回所有的计算结果，那么我们称其为**纯粹的**或者**无副作用**的。

#### 哪些因素会造成副作用呢？
- 除了构造器内的初始化操作，对类中数据结构的任何修改，包括字段的赋值操作（一个典型的例子是setter方法）
- 抛出一个异常
- 进行输入/输出操作，比如向一个文件写数据

#### “无副作用”的好处

1. 如果构成系统的各个组件都能遵守这一原则，该系统就能在完全无锁的情况下，使用多核的并发机制，因为任何一个方法都不会对其他的方法造成干扰
2. 这还是一个让你了解你的程序中哪些部分是相互独立的非常棒的机会

### 2）函数式编程的基石:声明式编程 

一般通过编程实现一个系统，有两种思考方式：
1. 专注于如何实现
2. 更加关注要做什么

Stream API的思考方式正是后一种。它把实现的细节留给了函数库。
```java
//找出最大值
Optional<Transaction> mostExpensive = 
    transactions.stream() 
        .max(comparing(Transaction::getValue));
```
我们把这种思想称之为内部迭代。

采用这种“要做什么”风格的编程通常被称*为声明式编程*。

### 3）为什么要采用函数式编程

1. 声明式编程
2. 无副作用计算

## 2. 什么是函数式编程

简而言之：它是一种使用**函数**进行编程的方式

那什么又是“函数”呢？

在函数式编程的上下文中，一个“函数”对应于一个数学函数：它接受零个或多个参数，生成一个或多个结果，并且不会有任何副作用。你可以把它看成一个黑盒，它接收输入并产生一些输出。

### 1）函数式Java编程

被称为“函数式”的函数或方法应具备以下特点：
1. 都只能修改本地变量
2. 它引用的对象都应该是不可修改的对象
3. 函数或者方法不应该抛出任何异常（使用Optional<T>类型）

### 2）引用透明性

“没有可感知的副作用”（不改变对调用者可见的变量、不进行I/O、不抛出异常）的这些限制都隐含着**引用透明性**。

如果一个函数只要传递同样的参数值，总是返回同样的结果，那这个函数就是引用透明的。

### 3）面向对象的编程和函数式编程的对比

Java 8认为函数式编程其实只是面向对象的一个极端。

实际操作中，Java程序员经常混用这两种风格。

### 4）函数式编程实战

#### 需求：

给定一个列表List<value>，比如{1,4,9}，构造一个List<List<Integer>>，它的成员都是类表{1, 4, 9}的子集

#### 初步实现
递归实现
```java
static List<List<Integer>> subsets(List<Integer> list) { 
    if (list.isEmpty()) { 
        List<List<Integer>> ans = new ArrayList<>(); 
        ans.add(Collections.emptyList()); 
        return ans; 
    } 
    // 将list分为两部分，第一个元素与subList
    Integer first = list.get(0); 
    List<Integer> rest = list.subList(1, list.size()); 
    // 获取 subList的所有子集
    List<List<Integer>> subans = subsets(rest); 
    // 将第一个元素加入subList的所有子集中，生成新的list
    List<List<Integer>> subans2 = insertAll(first, subans); 
    // 合并两个list
    return concat(subans, subans2);
}
```
#### 如何定义insertAll方法

这个地方要小心，不能在产生subans2时，修改到subans

```java
static List<List<Integer>> insertAll(Integer first, 
                                    List<List<Integer>> lists) { 
    List<List<Integer>> result = new ArrayList<>(); 
    for (List<Integer> list : lists) { 
        // 创建一个新的list，而不是直接修改
        List<Integer> copyList = new ArrayList<>(); 
        copyList.add(first); 
        copyList.addAll(list); 
        result.add(copyList); 
    } 
    return result; 
}
```
#### 定义concat方法

简单的实现，但是希望你不要这样使用，因为它修改了参数
```java
static List<List<Integer>> concat(List<List<Integer>> a, 
                                List<List<Integer>> b) { 
    a.addAll(b); 
    return a; 
} 
```
纯粹的函数式
```java
static List<List<Integer>> concat(List<List<Integer>> a, 
                                List<List<Integer>> b) { 
    List<List<Integer>> r = new ArrayList<>(a); 
    r.addAll(b); 
    return r; 
}
```

## 3. 递归和迭代

来实现一个阶乘计算

### 1）迭代式

```java
static int factorialIterative(int n) {
    int r = 1;
    for (int i = 1; i <= n; i++) {
        r *= i;
    }
    return r;
}
```
### 2）递归式

```java
static long factorialRecursive(long n) {
    return n == 1 ? 1 : n * factorialRecursive(n-1);
}
```

### 3）基于 Stream

```java
static long factorialStreams(long n){
return LongStream.rangeClosed(1, n)
    .reduce(1, (long a, long b) -> a * b);
}
```
### 4）基于“尾-递”的阶乘

```java
static long factorialTailRecursive(long n) {
    return factorialHelper(1, n);
}

static long factorialHelper(long acc, long n) {
    return n == 1 ? acc : factorialHelper(acc * n, n-1);
}
```

第4个递归和第2个递归相比，采用了**尾-调优化(tail-call optimization)**。

尾-调优化的基本的思想是你可以编写阶乘的一个迭代定义,不过迭代调用发生在函数的最后(所以我们说调用发生在尾部)。它避免了过多的在不同的栈帧上保存每次递归计算的中间值，编译器能够自行决定复用某个栈帧进行计算，从一定程度上避免了StackOverflowError 异常。

---

# 函数式编程中的高级技术

## 1. 无处不在的函数

对于程序员来讲，“函数式编程” 有两个特点：
1. 没有任何副作用
2. 函数可以像任何其他值一样随意使用，即一等函数

### 1）高阶函数
能满足下面任一要求就可以被称为**高阶函数（higher-order function）**：
- 接受至少一个函数作为参数
- 返回的结果是一个函数

如：Comparator.comparing
```java
Comparator<Apple> c = comparing(Apple::getWeight); 
```

### 2）科里化

#### 实例

先来看个例子，我们需要进行将一套单位转换到另一套单位的问题，

比如下面这种 x * f + b的需求：
```
static double converter(double x, double f, double b) { 
    return x * f + b; 
}
```
定义一个“工厂”方法，它输入f和b，返回一个带参数的方法：
```java
static DoubleUnaryOperator curriedConverter(double f, double b){ 
    return (double x) -> x * f + b; 
}
```
现在可以按照你的需求使用工厂方法产生你需要的任何converter：
```java
DoubleUnaryOperator convertCtoF = curriedConverter(9.0/5, 32); 
DoubleUnaryOperator convertUSDtoGBP = curriedConverter(0.6, 0); 
DoubleUnaryOperator convertKmtoMi = curriedConverter(0.6214, 0);
```

使用：
```java
double gbp = convertUSDtoGBP.applyAsDouble(1000);
```

上面的解决方案，不仅更加灵活，也复用了现有的转换逻辑。


#### 科里化的理论定义

是一种将具备**2个参数**（比如，x和y）的函数f转化为使用**一个参数**的函数g，并且这个函数的**返回值也是一个函数**，它会作为新函数的一个参数。后者的返回值和初始函数的返回值相同，即f(x,y) = (g(x))(y)。


一个函数使用所有参数仅有部分被传递时，通常我们说这个函数是**部分应用的（partially applied）**。

## 2. 持久化数据结构

### 1）破坏式更新和函数式更新的比较

实现一个可变类TrainJourney（利用一个简单的单向链接列表实现）表示从A地到B地的火车旅行。
```java
class TrainJourney { 
    public int price; 
    public TrainJourney onward; 
    public TrainJourney(int p, TrainJourney t) { 
        price = p; 
        onward = t; 
    } 
}
```
如果想串联起两条路线呢？一个是X到Y，一个是Y到Z。

传统做法：
```java
static TrainJourney link(TrainJourney a, TrainJourney b){ 
    if (a==null) return b; 
    TrainJourney t = a; 
    while(t.onward != null){ 
        t = t.onward; 
    } 
    t.onward = b; 
    return a; 
} 
```
这个方法是不停地把a的站走完，然后把a的最后一站和b的第一站连接起来。

然而这个方法是有副作用的，因为它更改了第一段旅程a。这违背了透明原则。

解决方法也很常见，就是创建它的一个副本而不要直接修改现存的数据结构。

但是有没有更好的方法呢？

来看看函数式编程方案代码：
```java
static TrainJourney append(TrainJourney a, TrainJourney b){ 
    return a==null ? b : new TrainJourney(a.price, append(a.onward, b)); 
}
```
很明显，这段代码是函数式的（它没有做任何修改，即使是本地的修改），它没有改动任何现存的数据结构。不过，也请特别注意，这段代码有一个特别的地方，它并未创建整个新TrainJourney对象的副本。

但仍需注意，第二个TrainJourney b是共享的。

### 2）另一个使用Tree的例子

```java
class Tree { 
    private String key; 
    private int val; 
    private Tree left, right; 
    public Tree(String k, int v, Tree l, Tree r) { 
        key = k; val = v; left = l; right = r; 
    } 
} 
class TreeProcessor { 
    public static int lookup(String k, int defaultval, Tree t) { 
        if (t == null) return defaultval; 
        if (k.equals(t.key)) return t.val; 
        return lookup(k, defaultval, 
        k.compareTo(t.key) < 0 ? t.left : t.right); 
    } 
    // 处理Tree的其他方法
} 
```
你希望通过二叉查找树找到String值对应的整型数。

该如何更新与某个键对应的值（简化起见，我们假设键已经存在于这个树中了）：

```java
public static Tree update(String k, int newval, Tree t) { 
    if (t == null) 
        t = new Tree(k, newval, null, null); 
    else if (k.equals(t.key)) 
        t.val = newval; 
    else if (k.compareTo(t.key) < 0) 
        t.left = update(k, newval, t.left); 
    else 
        t.right = update(k, newval, t.right); 
    return t; 
}
```
这个update方法会对现有的树进行修改。
### 3）采用函数式的方法

```java
public static Tree fupdate(String k, int newval, Tree t) { 
    return (t == null) ? 
        new Tree(k, newval, null, null) : 
            k.equals(t.key) ? 
                new Tree(k, newval, t.left, t.right) : 
            k.compareTo(t.key) < 0 ? 
                new Tree(t.key, t.val, fupdate(k,newval, t.left), t.right) : 
                new Tree(t.key, t.val, t.left, fupdate(k,newval, t.right)); 
}
```

fupdate是纯函数式的。它会创建一个新的树，并将其作为结果返回，通过参数的方式实现共享。

这种函数式数据结构通常被称为*持久化的*：数据结构的值始终保持一致，不受其他部分变化的影响。

## 3. Stream的延迟计算

### 1）自定义的Stream

回忆一下之前将的“是否为质数”这个需求。
```java
public static Stream<Integer> primes(int n) { 
    return Stream.iterate(2, i -> i + 1) 
        .filter(MyMathUtils::isPrime) 
        .limit(n); 
} 
public static boolean isPrime(int candidate) { 
    int candidateRoot = (int) Math.sqrt((double) candidate); 
    return IntStream.rangeClosed(2, candidateRoot) 
        .noneMatch(i -> candidate % i == 0); 
} 
```
但是有些啰嗦，遍历只需遍历所有质数即可。

#### 第一步： 构造由数字组成的Stream
```java
static Intstream numbers(){ 
    return IntStream.iterate(2, n -> n + 1); 
} 
```
#### 第二步： 取得首元素
```java
static int head(IntStream numbers){ 
    return numbers.findFirst().getAsInt(); 
}
```
#### 第三步： 对尾部元素进行筛选
定义一个方法取得Stream的尾部元素：
```java
static IntStream tail(IntStream numbers){ 
    return numbers.skip(1); 
} 
```
进行筛选
```java
IntStream numbers = numbers(); 
int head = head(numbers); 
IntStream filtered = tail(numbers).filter(n -> n % head != 0);
```
#### 第四步：递归地创建由质数组成的Stream
```java
static IntStream primes(IntStream numbers) { 
    int head = head(numbers); 
    return IntStream.concat( 
            IntStream.of(head), 
            primes(tail(numbers).filter(n -> n % head != 0)) 
        ); 
} 
```
#### 坏消息

不幸的是，如果执行步骤四中的代码，你会遭遇如下这个错误：“java.lang.IllegalStateException: stream has already been operated upon or closed.”实际上，你正试图使用两个终端操作：findFirst和skip将Stream切分成头尾两部分。

#### 延迟计算

除此之外，该操作还附带着一个更为严重的问题：静态方法IntStream.concat接受两个Stream实例作参数。但是，由于第二个参数是primes方法的直接递归调用，最终会导致出现无限递归的状况。

### 2）创建你自己的延迟列表

**延迟操作**：当你向一个Stream发起一系列的操作请求时，这些请求只是被一一保存起来。只有当你向Stream发起一个终端操作时，才会实际地进行计算。

#### 1. 一个基本的链接列表
```java
interface MyList<T> { 
    T head(); 
    MyList<T> tail();
    default boolean isEmpty() { 
        return true; 
    } 
} 
class MyLinkedList<T> implements MyList<T> { 
    private final T head; 
    private final MyList<T> tail; 
    public MyLinkedList(T head, MyList<T> tail) { 
        this.head = head; 
        this.tail = tail; 
    } 
    public T head() { 
        return head; 
    } 
    public MyList<T> tail() { 
        return tail; 
    } 
    public boolean isEmpty() { 
        return false; 
    } 
} 

class Empty<T> implements MyList<T> { 
    public T head() { 
        throw new UnsupportedOperationException(); 
    } 
    public MyList<T> tail() { 
        throw new UnsupportedOperationException(); 
    } 
}
```
构造一个示例的MyLinkedList值
```java
MyList<Integer> l = 
        new MyLinkedList<>(5, new MyLinkedList<>(10, new Empty<>())); 
```
#### 2. 一个基础的延迟列表

```java
import java.util.function.Supplier; 
class LazyList<T> implements MyList<T>{ 
    final T head; 
    final Supplier<MyList<T>> tail; 
    public LazyList(T head, Supplier<MyList<T>> tail) { 
        this.head = head;
        this.tail = tail; 
    } 
    public T head() { 
        return head; 
    } 
    //注意，与前面的head不同，
    //这里tail使用了一个Supplier方法提供了延迟性 
    public MyList<T> tail() { 
        return tail.get(); 
    } 
    public boolean isEmpty() { 
        return false; 
    } 
}
```
调用Supplier的get方法会触发延迟列表（LazyList）的节点创建，就像工厂会创建新的对象一样。

传递一个Supplier作为LazyList的构造器的tail参数，创建由数字构成的无限延迟列表了，该方法会创建一系列数字中的下一个元素：
```java
public static LazyList<Integer> from(int n) { 
    return new LazyList<Integer>(n, () -> from(n+1)); 
}
```
下面的代码执行会打印输出“2 3 4”。这些数字真真实实都是实时计算得出的。

```java
LazyList<Integer> numbers = from(2); 
int two = numbers.head(); 
int three = numbers.tail().head(); 
int four = numbers.tail().tail().head(); 
System.out.println(two + " " + three + " " + four);
```
#### 3. 回到生成质数

```java
public static MyList<Integer> primes(MyList<Integer> numbers) { 
    return new LazyList<>( 
        numbers.head(), 
        () -> primes( 
            numbers.tail() 
                    .filter(n -> n % numbers.head() != 0) 
                ) 
    ); 
}
```
#### 4. 实现一个延迟筛选器filter
上述MyList并未定义filter方法，来实现一个：

```java
public MyList<T> filter(Predicate<T> p) { 
    return isEmpty() ? 
            this : 
            p.test(head()) ? 
                new LazyList<>(head(), () -> tail().filter(p)) : 
                tail().filter(p); 
}
```
使用
```java
LazyList<Integer> numbers = from(2); 
int two = primes(numbers).head(); 
int three = primes(numbers).tail().head(); 
int five = primes(numbers).tail().tail().head(); 
System.out.println(two + " " + three + " " + five)；
```
试试其他想法.

比如打印输出所有的质数（这个程序会永久地运行下去）：
```java
static <T> void printAll(MyList<T> list){ 
    while (!list.isEmpty()){ 
        System.out.println(list.head()); 
        list = list.tail(); 
    } 
} 
printAll(primes(from(2)));
```
更加简洁地方式完成这个需求（递归操作）：
```java
static <T> void printAll(MyList<T> list){ 
    if (list.isEmpty()) 
        return; 
    System.out.println(list.head()); 
    printAll(list.tail()); 
}
```
但是，这个程序不会永久地运行下去；它最终会由于栈溢出而失效，因为Java不支持尾部调用消除（tail call elimination）。

#### 5. 何时使用

当你需要创建一个在开始阶段需要大量计算的数据结构时，可以考虑使用延迟操作，它会将这些操作在运行时在进行。



# 未完待续。。。。。。