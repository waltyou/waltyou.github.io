---
layout: post
title: 《Effective Java》学习日志（四）31：使用限定通配符来提高API灵活性
date: 2018-11-14 17:50:04
author: admin
comments: true
categories: [Java]
tags: [Java,Effective Java]
---


<!-- more -->

---

学习资料主要参考： 《Effective Java Third Edition》，作者：Joshua Bloch

---




* 目录
{:toc}

---

# 出现的背景

如Item 28所述，参数化类型是不变的。
换句话说，对于任何两个不同类型的Type1和Type2，List<Type1> 既不是 List<Type2>子类型也不是其父类型。
尽管List<String>不是List<Object>的子类型是违反直觉的，但它确实是有道理的。 
可以将任何对象放入List<Object>中，但是只能将字符串放入List<String>中。 
由于List<String>不能做List<Object>所能做的所有事情，所以它不是一个子类型（Item 10 中的里氏替代原则）。

相对于提供的不可变的类型，有时你需要比此更多的灵活性。 


# 例子

考虑Item 29中的Stack类。下面是它的公共API：

    public class Stack<E> {
        public Stack();
        public void push(E e);
        public E pop();
        public boolean isEmpty();
    }

## pushAll 方法 
    
假设我们想要添加一个方法来获取一系列元素，并将它们全部推送到栈上。 以下是第一种尝试：

    // pushAll method without wildcard type - deficient!
    public void pushAll(Iterable<E> src) {
        for (E e : src)
            push(e);
    }
    
这种方法可以干净地编译，但不完全令人满意。 
如果可遍历的src元素类型与栈的元素类型完全匹配，那么它工作正常。 

但是，假设有一个Stack<Number>，并调用push(intVal)，其中intVal的类型是Integer。 这是因为Integer是Number的子类型。 

从逻辑上看，这似乎也应该起作用：

    Stack<Number> numberStack = new Stack<>();
    Iterable<Integer> integers = ... ;
    
    numberStack.pushAll(integers);
    
但是，如果你尝试了，会得到这个错误消息，因为参数化类型是不变的：

    StackTest.java:7: error: incompatible types: Iterable<Integer>
    cannot be converted to Iterable<Number>
    numberStack.pushAll(integers);
    ^
    
幸运的是，有对应的解决方法。 
Java 提供了一种特殊的参数化类型来调用一个限定通配符类型来处理这种情况。 
pushAll的输入参数的类型不应该是“ Iterable of E ”，而应该是“Iterable of E 的子类”，并且有一个通配符类型，这意味着：Iterable<？ extends E>。 
（关键字extends的使用有点误导：回忆Item 29中，子类型被定义为每个类型都是它自己的子类型，即使它本身没有继承。）

让我们修改pushAll来使用这个类型：

    // Wildcard type for a parameter that serves as an E producer
    public void pushAll(Iterable<? extends E> src) {
        for (E e : src)
            push(e);
    }
    
有了这个改变，Stack类不仅可以干净地编译，而且客户端代码也不会用原始的pushAll声明编译。 因为Stack和它的客户端干净地编译，你知道一切都是类型安全的。

## popAll 方法 

现在假设你想写一个 popAll 方法，与pushAll方法相对应。 
popAll方法从栈中弹出每个元素并将元素添加到给定的集合中。 

以下是第一次尝试编写popAll方法的过程：

    // popAll method without wildcard type - deficient!
    public void popAll(Collection<E> dst) {
        while (!isEmpty())
            dst.add(pop());
    
    }
    
同样，如果目标集合的元素类型与栈的元素类型完全匹配，则干净编译并且工作正常。 

但是，这又不完全令人满意。 

假设你有一个Stack<Number>和Object类型的变量。 如果从栈中弹出一个元素并将其存储在该变量中，它将编译并运行而不会出错。 所以你也不能这样做吗？

    Stack<Number> numberStack = new Stack<Number>();
    Collection<Object> objects = ... ;
    numberStack.popAll(objects);
    
如果尝试将此客户端代码与之前显示的popAll版本进行编译，则会得到与我们的第一版pushAll非常类似的错误：Collection<Object>不是Collection<Number>的子类型。 

通配符类型再一次提供了一条出路。 popAll的输入参数的类型不应该是“E的集合”，而应该是“E的某个父类型的集合”（其中父类型被定义为E是它自己的父类型[JLS，4.10]）。 
再次，有一个通配符类型，正是这个意思：Collection<？ super E>。 让我们修改popAll来使用它：

    // Wildcard type for parameter that serves as an E consumer
    public void popAll(Collection<? super E> dst) {
        while (!isEmpty())
            dst.add(pop());
    }
    
通过这个改动，Stack类和客户端代码都可以干净地编译。

## PECS 

这个结论很清楚。 为了获得最大的灵活性，对代表生产者或消费者的输入参数使用通配符类型。 
如果一个输入参数既是一个生产者又是一个消费者，那么通配符类型对你没有好处：你需要一个精确的类型匹配，这就是没有任何通配符的情况。

这里有一个助记符来帮助你记住使用哪种通配符类型：
PECS代表： producer-extends，consumer-super。

换句话说，如果一个参数化类型代表一个T生产者，使用<? extends T>；如果它代表T消费者，则使用<? super T>。 
在我们的Stack示例中，pushAll方法的src参数生成栈使用的E实例，因此src的合适类型为Iterable<? extends E>；popAll方法的dst参数消费Stack中的E实例，因此dst的合适类型是Collection<？ super E>。 

PECS助记符抓住了使用通配符类型的基本原则。 Naftalin和Wadler称之为获取和放置原则（ Get and Put Principle ）[Naftalin07,2.4]。

记住这个助记符之后，让我们来看看本章中以前项目的一些方法和构造方法声明。 Item 28中的Chooser类构造方法有这样的声明：

    public Chooser(Collection<T> choices)
    
这个构造方法只使用集合选择来生产类型T的值（并将它们存储起来以备后用），所以它的声明应该使用一个extends T的通配符类型。下面是得到的构造方法声明：

    // Wildcard type for parameter that serves as an T producer
    public Chooser(Collection<? extends T> choices)

这种改变在实践中会有什么不同吗？ 是的，会有不同。 

假你有一个List<Integer>，并且想把它传递给Chooser<Number>的构造方法。 这不会与原始声明一起编译，但是它只会将限定通配符类型添加到声明中。

现在看看Item 30中的union方法。下是声明：

    public static <E> Set<E> union(Set<E> s1, Set<E> s2)

两个参数s1和s2都是E的生产者，所以PECS助记符告诉我们该声明应该如下：

    public static <E> Set<E> union(Set<? extends E> s1,  Set<? extends E> s2)

请注意，返回类型仍然是Set<E>。 不要使用限定通配符类型作为返回类型。除了会为用户提供额外的灵活性，还强制他们在客户端代码中使用通配符类型。 
通过修改后的声明，此代码将清晰地编译：

    Set<Integer>  integers =  Set.of(1, 3, 5);
    Set<Double>   doubles  =  Set.of(2.0, 4.0, 6.0);
    Set<Number>   numbers  =  union(integers, doubles);
    
如果使用得当，类的用户几乎不会看到通配符类型。 他们使方法接受他们应该接受的参数，拒绝他们应该拒绝的参数。 
如果一个类的用户必须考虑通配符类型，那么它的API可能有问题。

# 显式类型参数

在Java 8之前，类型推断规则不够聪明，无法处理先前的代码片段，这要求编译器使用上下文指定的返回类型（或目标类型）来推断E的类型。
union方法调用的目标类型如前所示是Set<Number>。 如果尝试在早期版本的Java中编译片段（以及适合的Set.of工厂替代版本），将会看到如此长的错综复杂的错误消息：

    Union.java:14: error: incompatible types
    Set<Number> numbers = union(integers, doubles);
    ^
    required: Set<Number>
    found: Set<INT#1>
    where INT#1,INT#2 are intersection types:
    INT#1 extends Number,Comparable<? extends INT#2>
    INT#2 extends Number,Comparable<?>
    
幸运的是有办法来处理这种错误。 如果编译器不能推断出正确的类型，你可以随时告诉它使用什么类型的显式类型参数[JLS，15.12]。 
甚至在Java 8中引入目标类型之前，这不是你必须经常做的事情，这很好，因为显式类型参数不是很漂亮。 
通过添加显式类型参数，如下所示，代码片段在Java 8之前的版本中进行了干净编译：

    // Explicit type parameter - required prior to Java 8
    Set<Number> numbers = Union.<Number>union(integers, doubles);
    
## 例子

接下来让我们把注意力转向Item 30中的max方法。这里是原始声明：

    public static <T extends Comparable<T>> T max(List<T> list)
    
以下是使用通配符类型的修改后的声明：

    public static <T extends Comparable<? super T>> T max(List<? extends T> list)

为了从原来到修改后的声明，我们两次应用了PECS。

首先直接的应用是参数列表。 
它生成T实例，所以将类型从List<T>更改为List<? extends T>。 棘手的应用是类型参数T。这是我们第一次看到通配符应用于类型参数。 
最初，T被指定为继承Comparable<T>，但Comparable的T消费T实例（并生成指示顺序关系的整数）。 
因此，参数化类型Comparable<T>被替换为限定通配符类型Comparable<? super T>。 

Comparable实例总是消费者，所以通常应该使用Comparable<? super T>优于Comparable<T>。 Comparator也是如此。因此，通常应该使用Comparator<? super T>优于Comparator<T>。

修改后的max声明可能是本书中最复杂的方法声明。 增加的复杂性是否真的起作用了吗？ 同样，它的确如此。 这是一个列表的简单例子，它被原始声明排除，但在被修改后的版本里是允许的：

    List<ScheduledFuture<?>> scheduledFutures = ... ;

无法将原始方法声明应用于此列表的原因是ScheduledFuture不实现Comparable<ScheduledFuture>。 
相反，它是Delayed的子接口，它继承了Comparable<Delayed>。 

换句话说，一个ScheduledFuture实例不仅仅和其他的ScheduledFuture实例相比较： 它可以与任何Delayed实例比较，并且足以导致原始的声明拒绝它。 
更普遍地说，通配符要求来支持没有直接实现Comparable（或Comparator）的类型，但继承了一个类型。


# 类型参数和通配符之间的双重性

还有一个关于通配符相关的话题。 

类型参数和通配符之间具有双重性，许多方法可以用一个或另一个声明。 例如，下面是两个可能的声明，用于交换列表中两个索引项目的静态方法。 

第一个使用无限制类型参数（Item 30），第二个使用无限制通配符：

    // Two possible declarations for the swap method
    public static <E> void swap(List<E> list, int i, int j);
    public static void swap(List<?> list, int i, int j);
    
这两个声明中的哪一个更可取，为什么？ 

在公共API中，第二个更好，因为它更简单。 你传入一个列表（任何列表），该方法交换索引的元素。 没有类型参数需要担心。 
通常，如果类型参数在方法声明中只出现一次，请将其替换为通配符。 

如果它是一个无限制的类型参数，请将其替换为无限制的通配符; 如果它是一个限定类型参数，则用限定通配符替换它。

第二个swap方法声明有一个问题。 这个简单的实现不会编译：

    public static void swap(List<?> list, int i, int j) {
        list.set(i, list.set(j, list.get(i)));
    }
    
试图编译它会产生这个不太有用的错误信息：

    Swap.java:5: error: incompatible types: Object cannot be
    converted to CAP#1
    list.set(i, list.set(j, list.get(i)));
    ^
    where CAP#1 is a fresh type-variable:
    CAP#1 extends Object from capture of ?
    
看起来我们不能把一个元素放回到我们刚刚拿出来的列表中。 问题是列表的类型是List<？>，并且不能将除null外的任何值放入List<？>中。 

幸运的是，有一种方法可以在不使用不安全的转换或原始类型的情况下实现此方法。 
这个想法是写一个私有辅助方法来捕捉通配符类型。 辅助方法必须是泛型方法才能捕获类型。 以下是它的定义：

    public static void swap(List<?> list, int i, int j) {
        swapHelper(list, i, j);
    }

    // Private helper method for wildcard capture
    private static <E> void swapHelper(List<E> list, int i, int j) {
        list.set(i, list.set(j, list.get(i)));
    }
    
swapHelper方法知道该列表是一个List<E>。 
因此，它知道从这个列表中获得的任何值都是E类型，并且可以安全地将任何类型的E值放入列表中。 

这个稍微复杂的swap的实现可以干净地编译。 
它允许我们导出基于通配符的漂亮声明，同时利用内部更复杂的泛型方法。 swap方法的客户端不需要面对更复杂的swapHelper声明，但他们从中受益。 
辅助方法具有我们认为对公共方法来说过于复杂的签名。

# 总结

总之，在你的API中使用通配符类型，虽然棘手，但使得API更加灵活。 

如果编写一个将被广泛使用的类库，正确使用通配符类型应该被认为是强制性的。 

记住基本规则： producer-extends, consumer-super（PECS）。 

还要记住，所有Comparable和Comparator都是消费者。
