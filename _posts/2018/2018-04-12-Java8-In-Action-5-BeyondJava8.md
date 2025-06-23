---
layout: post
title: 《Java 8 in Action》学习日志（五）：超越Java 8
date: 2018-4-12 10:30:04
author: admin
comments: true
categories: [Java]
tags: [Java,Java8]
---

书中最后一部分，介绍了对函数式编程的一些思考，以及函数式编程的使用技巧。也把java 与 scala进行比较。最后回顾了Java 8的新特性，以及对Java以后的展望。
<!-- more -->

学习资料主要参考： 《Java 8 In Action》、《Java 8实战》，以及其源码：[Java8 In Action](https://github.com/java8/Java8InAction)

---



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

## 4. 模式匹配

Java语言中暂时并未提供这一特性。

### 1）访问者设计模式

Java语言中还有另一种方式可以解包数据类型，那就是使用访问者（Visitor）设计模式。


使用这种方法你需要创建一个单独的类，称为访问者类，它接受某种数据类型的实例作为输入。它可以访问该实例的所有成员。


```java
class BinOp extends Expr{ 
    ... 
    public Expr accept(SimplifyExprVisitor v){ 
        return v.visit(this); 
    } 
}
```

SimplifyExprVisitor现在就可以访问BinOp对象并解包其中的内容了：

```java
public class SimplifyExprVisitor { 
    ... 
    public Expr visit(BinOp e){ 
        if("+".equals(e.opname) && e.right instanceof Number && …){ 
            return e.left; 
        }
        return e; 
    } 
}
```

### 2）用模式匹配力挽狂澜

```java
interface TriFunction<S, T, U, R>{ 
    R apply(S s, T t, U u); 
}

static <T> T patternMatchExpr( 
            Expr e, 
            TriFunction<String, Expr, Expr, T> binopcase, 
            Function<Integer, T> numcase, 
            Supplier<T> defaultcase) { 
    return 
        (e instanceof BinOp) ? 
            binopcase.apply(((BinOp)e).opname, ((BinOp)e).left, 
                                             ((BinOp)e).right) : 
        (e instanceof Number) ? 
            numcase.apply(((Number)e).val) : 
                            defaultcase.get(); 
}

public static Expr simplify(Expr e) { 
    TriFunction<String, Expr, Expr, Expr> binopcase = 
        (opname, left, right) -> { 
            if ("+".equals(opname)) { 
                if (left instanceof Number && ((Number) left).val == 0) { 
                    return right; 
                }
                if (right instanceof Number && ((Number) right).val == 0) { 
                    return left; 
                } 
            } 
            if ("*".equals(opname)) { 
                if (left instanceof Number && ((Number) left).val == 1) { 
                    return right; 
                } 
                if (right instanceof Number && ((Number) right).val == 1) { 
                    return left; 
                } 
            } 
            return new BinOp(opname, left, right); 
    }; 
    
    Function<Integer, Expr> numcase = val -> new Number(val); 
    Supplier<Expr> defaultcase = () -> new Number(0); 
    return patternMatchExpr(e, binopcase, numcase, defaultcase); 
}
//调用
Expr e = new BinOp("+", new Number(5), new Number(0)); 
Expr match = simplify(e); 
System.out.println(match);
```
## 5. 杂项

### 1）缓存或记忆表

当我们需要重复调用某个方法来获取某个数据结构的特征值时，而且这个方法本身的代价很高，我们可以考虑将结果，也就是特征值，缓存起来，使用记忆表。

最简单就是加个Map。

然而严格地说，这种方式并非纯粹的函数式解决方案，因为它会修改由多个调用者共享的数据结构。

既然是共享的数据结构，那么就存在多线程竞争和多核计算损失性能的问题。

### 2）“返回同样的对象”意味着什么

函数式编程通常不使用==（引用相等），而是使用equal对数据结构值进行比较。

### 3）结合器
例子:

接受一个参数，并使用函数f连续地对它进行操作（比如n次），类似循环的效果。

```java
static <A,B,C> Function<A,C> compose(Function<B,C> g, Function<A,B> f) { 
    return x -> g.apply(f.apply(x)); 
}

static <A> Function<A,A> repeat(int n, Function<A,A> f) { 
    return n==0 ? x -> x 
        : compose(f, repeat(n-1, f)); 
}

// 输出结果为80 = 2*(2*(2* 10))
System.out.println(repeat(3, (Integer x) -> 2*x).apply(10)); 
```

---

# Java 8和Scala语言的特性比较

## 1. Scala简介

### 1）例子：你好，啤酒
#### 命令式Scala
```scala
object Beer { 
    def main(args: Array[String]){ 
        var n : Int = 2 
        while( n <= 6 ){ 
            println(s"Hello ${n} bottles of beer") 
            n += 1 
        } 
    } 
}
```
#### 函数式Scala
java
```java
public class Foo { 
    public static void main(String[] args) { 
        IntStream.rangeClosed(2, 6) 
            .forEach(n -> System.out.println("Hello " + n + 
                                            " bottles of beer")); 
    } 
}
```
Scala
```scala
object Beer { 
    def main(args: Array[String]){ 
        2 to 6 foreach { n => println(s"Hello ${n} bottles of beer") } 
    } 
}
```
### 2）基础数据结构：List、Set、Map、Tuple、Stream以及Option
####  1、创建集合
```scala
val authorsToAge = Map("Raoul" -> 23, "Mario" -> 40, "Alan" -> 53)
val authors = List("Raoul", "Mario", "Alan") 
val numbers = Set(1, 1, 2, 3, 5, 8) 
```
#### 2、不可变与可变的比较
之前创建的集合在默认情况下都是只读的。这意味着它们从创建开始就不能修改。

那么，你怎样才能更新Scala语言中不可变的集合呢？
```scala
val numbers = Set(2, 5, 3); 
val newNumbers = numbers + 8 
println(newNumbers) 
println(numbers)
```
#### 3、使用集合
```scala
val fileLines = Source.fromFile("data.txt").getLines.toList() 
// 实现1
val linesLongUpper 
    = fileLines.filter(l => l.length() > 10) 
        .map(l => l.toUpperCase())
// 实现2
// 下划线是一种占位符，它按照位置匹配对应的参数。
val linesLongUpper 
    = fileLines filter (_.length() > 10) map(_.toUpperCase()) 
// 使用并行流
val linesLongUpper 
    = fileLines.par filter (_.length() > 10) map(_.toUpperCase()) 
```
#### 4、元组
```scala
val raoul = ("Raoul", "+ 44 887007007") 
val alan = ("Alan", "+44 883133700")
val book = (2014, "Java 8 in Action", "Manning") 
val numbers = (42, 1337, 0, 3, 14)
//访问
println(book._1) 
println(numbers._4) 
```
#### 5、Stream
Scala中的Stream可以记录它曾经计算出的值，所以之前的元素可以随时进行访问。除此之外，Stream还进行了索引，所以Stream中的元素可以像List那样通过索引访问。
#### 6、Option
```scala
def getCarInsuranceName(person: Option[Person], minAge: Int) = 
    person.filter(_.getAge() >= minAge) 
        .flatMap(_.getCar) 
        .flatMap(_.getInsurance) 
        .map(_.getName).getOrElse("Unknown")
```
## 2. 函数

### 1）Scala中的一等函数

函数在Scala语言中是一等值。

使用谓词（返回一个布尔型结果的函数）定义这两个筛选条件
```scala
def isJavaMentioned(tweet: String) : Boolean = tweet.contains("Java") 
def isShortTweet(tweet: String) : Boolean = tweet.length() < 20 
```
使用它们

```scala
val tweets = List( 
    "I love the new features in Java 8", 
    "How's it going?", 
    "An SQL query walks into a bar, sees two tables and says 'Can I join you?'" 
) 
tweets.filter(isJavaMentioned).foreach(println) 
tweets.filter(isShortTweet).foreach(println) 
```

内嵌方法filter的函数签名：

```scala
def filter[T](p: (T) => Boolean): List[T]
```
这其实是一种新的语法，Java中暂时还不支持。它描述的是一个函数类型。这里它表示的是这样一个函数，它接受类型为T的对象，返回一个布尔类型的值。

### 2）匿名函数和闭包

#### 匿名函数
```scala
val isLongTweet : String => Boolean 
    = (tweet : String) => tweet.length() > 60

// 使用
isLongTweet.apply("A very short tweet")
```

为了使用Lambda表达式，Java提供了几种内置的函数式接口，比如Predicate、Function、Consumer。Scala提供了trait（你可以暂时将trait想象成接口，我们会在接下来的一节介绍它们）来实现同样的功能：从Function0（一个函数不接受任何参数，并返回一个结果）到Function22（一个函数接受22个参数），它们都定义了apply方法。
Scala还提供了另一个非常酷炫的特性，你可以使用语法糖调用apply方法，效果就像一次函数调用：

```scala
isLongTweet("A very short tweet")
```

#### 闭包

什么是闭包呢？闭包是一个函数实例，它可以不受限制地访问该函数的非本地变量。

Java 8中的Lambda表达式自身带有一定的限制：它们不能修改定义Lambda表达式的函数中的本地变量值。这些变量必须隐式地声明为final。这些背景知识有助于我们理解“Lambda避免了对变量值的修改，而不是对变量的访问”。

与此相反，Scala中的匿名函数可以取得自身的变量，但并非变量当前指向的变量值。

```scala
def main(args: Array[String]) { 
    var count = 0 
    val inc = () => count+=1 
    inc() 
    println(count) // print 1
    inc() 
    println(count) // print 2
}
```

### 3）科里化

#### java实现

定义了一个简单的函数，对两个正整数做乘法运算：
```java
static int multiply(int x, int y) { 
    return x * y; 
} 
int r = multiply(2, 10); 
```
人工地对multiple方法进行切分，让其返回另一个函数：
```java
static Function<Integer, Integer> multiplyCurry(int x) { 
    return (Integer y) -> x * y; 
}
```
使用这个函数，下列代码会对每个元素依次乘以2：
```java
Stream.of(1, 3, 5, 7) 
    .map(multiplyCurry(2)) 
    .forEach(System.out::println);
```

#### scala实现

```scala
def multiply(x : Int, y: Int) = x * y 
val r = multiply(2, 10);
```
科里化
```scala
def multiplyCurry(x :Int)(y : Int) = x * y
val r = multiplyCurry(2)(10)
```
这个函数是部分应用的，因为它并未提供所有的参数。

这意味着你可以将对multiplyCurry的第一次调用保存到一个变量中，进行复用：

```scala
val multiplyByTwo : Int => Int = multiplyCurry(2) 
val r = multiplyByTwo(10) 
```

## 3. 类和trait

### 1）更加简洁的Scala类

由于Scala也是一门完全的面向对象语言，你可以创建类，并将其实例化生成对象。最基础的形态上，声明和实例化类的语法与Java非常类似。

```scala
class Hello { 
    def sayThankYou(){ 
        println("Thanks for reading our book") 
    } 
} 
val h = new Hello() 
h.sayThankYou()
```
#### getter方法和setter方法

一旦定义的Java类具有了字段，我们就不得不需要声明一长串的getter方法、setter方法，以及恰当的构造器。非常麻烦！

Scala语言中构造器、getter方法以及setter方法都能隐式地生成，从而大大降低你代码中的冗余：
```scala
class Student(var name: String, var id: Int) 
val s = new Student("Raoul", 1) 
println(s.name) 
s.id = 1337 
println(s.id)
```

### 2）Scala的trait与Java 8的接口对比

Scala还提供了另一个非常有助于抽象对象的特性，名称叫trait。

它是Scala为实现Java中的接口而设计的替代品。

trait中既可以**定义抽象方法**，也可以定义**带有默认实现的方法**。trait同时还支持Java中接口那样的**多继承**。

现在，Java 8通过默认方法又引入了对**行为的多继承**，不过它依旧不支持对**状态的多继承**，而这恰恰是trait支持的。

来创建一个实例：
```scala
trait Sized{ 
    var size : Int = 0 
    def isEmpty() = size == 0 
} 
```
使用一个类在声明时构造它：
```scala
class Empty extends Sized 
println(new Empty().isEmpty())
```

有一件事非常有趣，trait和Java的接口类似，也是在对象实例化时被创建（不过这依旧是一个编译时的操作）。比如，你可以创建一个Box类，动态地决定到底选择哪一个实例支持由trait Sized定义的操作：
```scala
class Box 
val b1 = new Box() with Sized 
println(b1.isEmpty()) // print true
val b2 = new Box() 
b2.isEmpty() // 编译错误，因为Box类的声明并未继承Sized 
```

---

# 回顾与展望
## 1. 回归Java 8的特性

### 1）行为参数化（Lambda以及方法引用）
- 传递一个Lambda表达式，即一段精简的代码片段，比如

```java
apple -> apple.getWeight() > 150
```

- 传递一个方法引用，该方法引用指向了一个现有的方法，比如这样的代码：

```java
Apple::isHeavy
```
### 2）流

为什么要引入了一套全新的Stream API？

对于大型数据集，集合处理会随着处理逻辑的复杂，会对数据进行多次读取，而stream只是读一次，会更高效。

而且Stream它的parallel方法能帮助将一个Stream标记为适合进行并行处理。

### 3）CompletableFuture

一个非常有用，不过不那么精确的格言这么说：“Completable-Future对于Future的意义就像Stream之于Collection。”让我们比较一下这二者：

- 通过Stream你可以对一系列的操作进行流水线，通过map、filter或者其他类似的方法提供行为参数化，它可有效避免使用迭代器时总是出现模板代码。
- 类似地，CompletableFuture提供了像thenCompose、thenCombine、allOf这样的
操作，对Future涉及的通用设计模式提供了函数式编程的细粒度控制，有助于避免使用命令式编程的模板代码。

### 4）Optional

Java 8的库提供了Optional<T>类，这个类允许你在代码中指定哪一个变量的值既可能是类型T的值，也可能是由静态方法Optional.empty表示的缺失值。

Optional类提供了map、filter和ifPresent方法。这些方法能以函数式的结构串接计算，由于库自身提供了缺失值的检测机制，不再需要用户代码的干预。

### 5）默认方法

接口中新引入的默认方法对类库的设计者而言简直是如鱼得水。

它提供的能力能帮助类库的设计者们定义新的操作，增强接口的能力，类库的用户们（即那些实现该接口的程序员们）不需要花费额外的精力重新实现该方法。

因此，默认方法与库的用户也有关系，它们屏蔽了将来的变化对用户的影响。

## 2. Java的未来

### 1）集合

集合（通过Collection接口）提供了一种更优秀也更一致的解决方案。不过它们的初始化被忽略了。

现在：
```java
Map<String, Integer> map = new HashMap<>(); 
map.put("raoul", 23); 
map.put("mario", 40); 
map.put("alan", 53);
```
期望：
```java
Map<String, Integer> map = #{"Raoul" -> 23, "Mario" -> 40, "Alan" -> 53};
```

### 2）类型系统的改进

#### 1. 声明位置变量
Java加入了对通配符的支持，来更灵活地支持泛型的子类型（subtyping）, 或者我们可以更通俗地称之为“用户定义变量”（use-site variance）。

这也是下面这段代码合法的原因：
```java
List<? extends Number> numbers = new ArrayList<Integer>(); 
```
不过下面的这段赋值（省略了? extends）会产生一个编译错误：
```java
List<Number> numbers = new ArrayList<Integer>();
```

#### 2. 更多的类型推断

最初在Java中，无论何时我们使用一个变量或方法，都需要同时给出它的类型。

随着时间的推移，这种限制被逐渐放开了。首先，你可以在一个表达式中忽略泛型参数的类型，通过上下文决定其类型。比如：
```java
// before
Map<String, List<String>> myMap = new HashMap<String, List<String>>();
// after
Map<String, List<String>> myMap = new HashMap<>();

// before
Function<Integer, Boolean> p = (Integer x) -> booleanExpression;
// after
Function<Integer, Boolean> p = x -> booleanExpression;
```

Scala和C#中都允许使用关键词var替换本地变量的初始化声明，编译器会依据右边的变量填充恰当的类型。这种思想被称为**本地变量类型推断**。

### 3）模式匹配

函数式语言通常都会提供某种形式的模式匹配——作为switch的一种改良形式。

### 4）更加丰富的泛型形式

本节会讨论Java泛型的两个局限性，并探讨可能的解决方案。

#### 1.具化泛型

Java 5中初次引入泛型时，需要它们尽量保持与现存JVM的后向兼容性。为了达到这一目的，ArrayList<String>和ArrayList<Integer>的运行时表示是相同的。这 被称作**泛型多态（generic polymorphism）的消除模式（erasure model）**。

所以Java不支持ArrayList<int>这种类型的泛型。

因为这样一来ArrayList容器就无法了解它所容纳的到底是一个对象类型的值，然后JVM就无法判断ArrayList中的元素到底是一个Integer的引用（可以被垃圾收集器标记为“in use”并进行跟踪），还是int类型的简单数据（几乎可以说是无法跟踪的）。

C#语言中，ArrayList<String>、ArrayList<Integer>以及ArrayList<int>的运行时
表示在原则上就是不同的。即使它们的值是相同的，也伴随着足够的运行时类型信息，这些信息可以帮助垃圾收集器判断一个字段值到底是引用，还是简单数据。这被称为**泛型多态的具化模式，或具化泛型**。“具化”这个词意味着“将某些默认隐式的东西变为显式的”。

#### 2.泛型中特别为函数类型增加的语法灵活性

BiFunction<T, U, R>，这里的T表示第一个参数的类型，U表示第二个参数的类型，而R是计算的结果。

同理，你不能用Function<T, R>引用表示某个不接受任何参数，返回值为R类型的函数；只能通过Supplier<R>达到这一目的。

在很多的函数式编程语言中，你可以用(Integer, Double) 
=> String这样的类型实现Java 8中BiFunction<Integer, Double, String>调用得到同样的效果。

#### 3.原型特化和泛型

比如，有人可能会问为什么Java 8中我们需要编写Predicate<Apple>，而不是直接采用Function<Apple, Boolean>的方式？

事实上，Predicate<Apple>类型的对象在执行test方法调用时，其返回值依旧是简单类型boolean。

所以使用
Predicate<Apple>更加高效，因为它无需将boolean装箱为Boolean。

另一个例子和void之间的区别有关，void只能修饰不带任何值的方法，而Void对象实际包含了一个值，它有且仅有一个null值。

对于Function的特殊情况，比如Supplier<T>，你可以用前面建议的新操作符将其改写为() => T。

### 5）对不变性的更深层支持

Java 8只支持三种类型的值，分别为：
- 简单类型值
- 指向对象的引用
- 指向函数的引用

如果我们想在Java中真正实现函数式编程，那么语言层面的支持就必不可少了，比如“不可变值”。正如我们在第13章中所了解的那样，关键字final并未在真正意义上是要达到这一目标，它仅仅避免了对它所修饰字段的更新。

```java
final int[] arr = {1, 2, 3}; 
final List<T> list = new ArrayList<>();
```

前者禁止了直接的赋值操作arr = ...，不过它并未阻止以arr[1]=2这样的方式对数组进行修改；

而后者禁止了对列表的赋值操作，但并未禁止以其他方法修改列表中的元素！

关键字final对于**简单数据类型**的值操作效果很好，不过对于对象引用，它通常只是一种错误的安全感。

由于函数式编程对不能修改现存数据结构有非常严格的要求，所以它提供了一个更强大的关键字，比如**transitively_final**，该关键字用于修饰引用类型的字段，确保无论是直接对该字段本身的修改，还是对通过该字段能直接或间接访问到的对象的修改都不会发生。

### 6）值类型

#### 1. 为什么编译器不能对Integer和int一视同仁

用于建模复数的Complex包含了两个部分，分别是实数（real）和虚数（imaginary），一种很直观的定义如下：
```java
class Complex { 
    public final double re; 
    public final double im; 
    public Complex(double re, double im) { 
        this.re = re; 
        this.im = im; 
    } 
    public static Complex add(Complex a, Complex b) { 
        return new Complex(a.re+b.re, a.im+b.im); 
    } 
}
```
不过类型Complex的值为引用类型，对Complex的每个操作都需要进行对象分配——增加了add中两次加法操作的开销。我们需要的是类似Complex的简单数据类型，我们也许可以称其为complex。

以下面的这个难题为例：
```java
double d1 = 3.14; 
double d2 = d1; 
Double o1 = d1; 
Double o2 = d2; 
Double ox = o1; 
System.out.println(d1 == d2 ? "yes" : "no"); // yes
System.out.println(o1 == o2 ? "yes" : "no"); // no
System.out.println(o1 == ox ? "yes" : "no"); // yes
```
对于简单变量，特征比较采用的是逐位比较（bitwise comparison），对于对象
类型它采用的是引用比较（reference equality）。

#### 2.值对象——无论简单类型还是对象类型都不能包打天下

Java的初心：
1. 任何事物，如果不是简单数据类型，就是对象类型，所有的对象类型都继承自Object；
2. 所有的引用都是指向对象的引用。

Java中有两种类型的值：
- 一类是对象类型，它们包含着可变的字段（除非使用了final关键字进行修饰），对这种类型值的特征，可以使用==进行比较；
- 还有一类是值类型，这种类型的变量是不能改变的，也不带任何的引用特征（reference identity），简单类型就属于这种更宽泛意义上的值类型。

对于值类型，默认情况下，硬件对int进行比较时会以一个字节接着一个字节逐次的方式进行，==会以同样的方式一个元素接着一个元素地对两个变量进行比较。 

此外，值类型可以减少对存储的要求，因为它们并不包含引用特征。

#### 3.装箱、泛型、值类型——互相交织的问题

我们希望能够在Java中引入值类型，因为函数式编程处理的不可变对象并不含有特征。

我们希望简单数据类型可以作为值类型的特例，但又不要有当前Java所携带的泛型的消除模式，因为这意味着值类型不做装箱就不能使用泛型。

由于对象的消除模式，简单类型（比如int）的对象（装箱）版本（比如Integer）对集合和Java泛型依旧非常重要，不过它们继承自Object（并因此引用相等），这被当成了一种缺点。

解决这些问题中的任何一个就意味着解决了所有的问题。