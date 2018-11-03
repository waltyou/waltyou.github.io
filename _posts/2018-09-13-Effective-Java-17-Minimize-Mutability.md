---
layout: post
title: 《Effective Java》学习日志（三）17: 最小化可变性
date: 2018-09-13 09:41:04
author: admin
comments: true
categories: [Java]
tags: [Java,Effective Java]
---

不可变类简单来说是它的实例不能被修改的类。 
包含在每个实例中的所有信息在对象的生命周期中是固定的，因此不会观察到任何变化。 

<!-- more -->

---

学习资料主要参考： 《Effective Java Third Edition》，作者：Joshua Bloch

---

## 目录
{:.no_toc}

* 目录
{:toc}

---

# 不可变的类

Java平台类库包含许多不可变的类，包括String类，基本类型包装类以及BigInteger类和BigDecimal类。 
有很多很好的理由：不可变类比可变类更容易设计，实现和使用。 
他们不太容易出错，更安全。

# 不可变类的规则

要使一个类不可变，请遵循以下五条规则：

1. 不要提供修改对象状态的方法（也称为mutators）。
2. 确保这个类不能被继承。 
    
    这可以防止粗心的或恶意的子类，假设对象的状态已经改变，从而破坏类的不可变行为。
    防止子类化通常是通过final修饰类，但是我们稍后将讨论另一种方法。
3. 把所有属性设置为final。

    通过系统强制执行，清楚地表达了你的意图。 
    另外，如果一个新创建的实例的引用从一个线程传递到另一个线程而没有同步，就必须保证正确的行为，正如内存模型[JLS，17.5; Goetz06,16]所述。
4. 把所有的属性设置为private。 

    这可以防止客户端获得对属性引用的可变对象的访问权限并直接修改这些对象。 
    虽然技术上允许不可变类具有包含基本类型数值的公共final属性或对不可变对象的引用，但不建议这样做，因为它不允许在以后的版本中更改内部表示（项目15和16）。
5. 确保对任何可变组件的互斥访问。 

    如果你的类有任何引用可变对象的属性，请确保该类的客户端无法获得对这些对象的引用。 
    切勿将这样的属性初始化为客户端提供的对象引用，或从访问方法返回属性。 
    在构造方法，访问方法和 readObject 方法（Item 88）中进行防御性拷贝（Item 50）。

# 示例

以前Item中的许多示例类都是不可变的。 
其中这样的类是Item 11中的PhoneNumber类，它具有每个属性的访问方法（accessors），但没有相应的设值方法（mutators）。 
这是一个稍微复杂一点的例子：

```java
// Immutable complex number class

public final class Complex {

    private final double re;
    private final double im;

    public Complex(double re, double im) {
        this.re = re;
        this.im = im;
    }

    public double realPart() {
        return re;
    }

    public double imaginaryPart() {
        return im;
    }

    public Complex plus(Complex c) {
        return new Complex(re + c.re, im + c.im);
    }

    public Complex minus(Complex c) {
        return new Complex(re - c.re, im - c.im);
    }

    public Complex times(Complex c) {
        return new Complex(re * c.re - im * c.im,
                re * c.im + im * c.re);
    }

    public Complex dividedBy(Complex c) {
        double tmp = c.re * c.re + c.im * c.im;
        return new Complex((re * c.re + im * c.im) / tmp,
                (im * c.re - re * c.im) / tmp);
    }

    @Override
    public boolean equals(Object o) {
        if (o == this) {
            return true;
        }
        if (!(o instanceof Complex)) {
            return false;
        }
        Complex c = (Complex) o;
        // See page 47 to find out why we use compare instead of ==
        return Double.compare(c.re, re) == 0
                && Double.compare(c.im, im) == 0;
    }

    @Override
    public int hashCode() {
        return 31 * Double.hashCode(re) + Double.hashCode(im);
    }

    @Override
    public String toString() {
        return "(" + re + " + " + im + "i)";
    }
}
```

这个类代表了一个复数（包含实部和虚部的数字）。 

除了标准的Object方法之外，它还为实部和虚部提供访问方法，并提供四个基本的算术运算：加法，减法，乘法和除法。 
注意算术运算如何创建并返回一个新的Complex实例，而不是修改这个实例。 

# 函数式方法

这种模式被称为**函数式方法**，因为方法返回将操作数应用于函数的结果，而不修改它们。 

与其对应的过程（procedural）或命令（imperative）的方法相对比，在这种方法中，将一个过程作用在操作数上，导致其状态改变。 
请注意，方法名称是介词（如plus）而不是动词（如add）。 
这强调了方法不会改变对象的值的事实。 

BigInteger和BigDecimal类没有遵守这个命名约定，并导致许多使用错误。

如果你不熟悉函数式方法，可能会显得不自然，但它具有不变性，具有许多优点。 
不可变对象很简单。 
一个不可变的对象可以完全处于一种状态，也就是被创建时的状态。
 如果确保所有的构造方法都建立了类不变量，那么就保证这些不变量在任何时候都保持不变，使用此类的程序员无需再做额外的工作。 
 另一方面，可变对象可以具有任意复杂的状态空间。 
 如果文档没有提供由设置（mutator）方法执行的状态转换的精确描述，那么可靠地使用可变类可能是困难的或不可能的。

不可变对象本质上是线程安全的; 它们不需要同步。 
被多个线程同时访问它们时并不会被破坏。 
这是实现线程安全的最简单方法。 
由于没有线程可以观察到另一个线程对不可变对象的影响，所以不可变对象可以被自由地共享。 
因此，不可变类应鼓励客户端尽可能重用现有的实例。 
一个简单的方法是为常用的值提供公共的静态 final常量。
 
例如，Complex类可能提供这些常量：

```java
public static final Complex ZERO = new Complex(0, 0);
public static final Complex ONE  = new Complex(1, 0);
public static final Complex I    = new Complex(0, 1);
```

这种方法可以更进一步。 
一个不可变的类可以提供静态的工厂（Item 1）来缓存经常被请求的实例，以避免在现有的实例中创建新的实例。 
所有基本类型的包装类和BigInteger类都是这样做的。 
使用这样的静态工厂会使客户端共享实例而不是创建新实例，从而减少内存占用和垃圾回收成本。 
在设计新类时，选择静态工厂代替公共构造方法，可以在以后增加缓存的灵活性，而不需要修改客户端。

# 优点

不可变对象可以自由分享的结果是，你永远不需要做出防御性拷贝（ defensive copies）（Item 50）。 
事实上，永远不需要做任何拷贝，因为这些拷贝永远等于原始对象。 
因此，你不需要也不应该在一个不可变的类上提供一个clone方法或拷贝构造方法（copy constructor）（Item 13）。 
这一点在Java平台的早期阶段还不是很好理解，所以String类有一个拷贝构造方法，但是它应该尽量很少使用（Item 6）。

不仅可以共享不可变的对象，而且可以共享内部信息。 
例如，BigInteger类在内部使用符号数值表示法。 
符号用int值表示，数值用int数组表示。 
negate方法生成了一个数值相同但符号相反的新BigInteger实例。 
即使它是可变的，也不需要复制数组；新创建的BigInteger指向与原始相同的内部数组。

不可变对象为其他对象提供了很好的构件（building blocks），无论是可变的还是不可变的。 
如果知道一个复杂组件的内部对象不会发生改变，那么维护复杂对象的不变量就容易多了。
这一原则的特例是，不可变对象可以构成Map对象的键和Set的元素，一旦不可变对象作为Map的键或Set里的元素，即使破坏了Map和Set的不可变性，但不用担心它们的值会发生变化。

不可变对象提供了免费的原子失败机制（Item 76）。
它们的状态永远不会改变，所以不可能出现临时的不一致。

# 缺点

不可变类的主要缺点是对于每个不同的值都需要一个单独的对象。 
创建这些对象可能代价很高，特别是如果是大型的对象下。 

例如，假设你有一个百万位的 BigInteger，你想改变它的低位：

```java
BigInteger moby = ...;
moby = moby.flipBit(0);
```

flipBit方法创建一个新的BigInteger实例，也是一百万位长，与原始位置只有一位不同。 
该操作需要与BigInteger大小成比例的时间和空间。 
将其与java.util.BitSet对比。 
像BigInteger一样，BitSet表示一个任意长度的位序列，但与BigInteger不同，BitSet是可变的。 B
itSet类提供了一种方法，允许你在固定时间内更改百万位实例中单个位的状态：

```java
BitSet moby = ...;
moby.flip(0);
```

如果执行一个多步操作，在每一步生成一个新对象，除最终结果之外丢弃所有对象，则性能问题会被放大。
这里有两种方式来处理这个问题。

## 解决办法一

先猜测一下会经常用到哪些多步的操作，然后讲它们作为基本类型提供。
如果一个多步操作是作为一个基本类型提供的，那么不可变类就不必在每一步创建一个独立的对象。
在内部，不可变的类可以是任意灵活的。 

例如，BigInteger有一个包级私有的可变的“伙伴类（companion class）”，它用来加速多步操作，比如模幂运算（ modular exponentiation）。
出于前面所述的所有原因，使用可变伙伴类比使用BigInteger要困难得多。 
幸运的是，你不必使用它：BigInteger类的实现者为你做了很多努力。

如果你可以准确预测客户端要在你的不可变类上执行哪些复杂的操作，那么包级私有可变伙伴类的方式可以正常工作。

## 解决办法二

如果不是的话，那么最好的办法就是提供一个公开的可变伙伴类。 
这种方法在Java平台类库中的主要例子是String类，它的可变伙伴类是StringBuilder（及其过时的前身StringBuffer类）。

现在你已经知道如何创建一个不可改变类，并且了解不变性的优点和缺点，下面我们来讨论几个设计方案。 
回想一下，为了保证不变性，一个类不得允许子类化。 
这可以通过使类用 final 修饰，但是还有另外一个更灵活的选择。 
而不是使不可变类设置为 final，可以使其所有的构造方法私有或包级私有，并添加公共静态工厂，而不是公共构造方法（Item 1）。 
为了具体说明这种方法，下面以Complex为例，看看如何使用这种方法：

```java
// Immutable class with static factories instead of constructors
public class Complex {
    private final double re;
    private final double im;

    private Complex(double re, double im) {
        this.re = re;
        this.im = im;
    }

    public static Complex valueOf(double re, double im) {
        return new Complex(re, im);
    }

    ... // Remainder unchanged
}
```

这种方法往往是最好的选择。 

这是最灵活的，因为它允许使用多个包级私有实现类。 

对于驻留在包之外的客户端，不可变类实际上是final的，因为不可能继承来自另一个包的类，并且缺少公共或受保护的构造方法。 
除了允许多个实现类的灵活性以外，这种方法还可以通过改进静态工厂的对象缓存功能来调整后续版本中类的性能。

当BigInteger和BigDecimal被写入时，不可变类必须是有效的final，因此它们的所有方法都可能被重写。
不幸的是，在保持向后兼容性的同时，这一事实无法纠正。
如果你编写一个安全性取决于来自不受信任的客户端的BigInteger或BigDecimal参数的不变类时，则必须检查该参数是“真实的”BigInteger还是BigDecimal，而不应该是不受信任的子类的实例。
如果是后者，则必须在假设可能是可变的情况下保护性拷贝（defensively copy）（Item 50）：

```java
public static BigInteger safeInstance(BigInteger val) {

    return val.getClass() == BigInteger.class ?
            val : new BigInteger(val.toByteArray());
}
```

# 变通之处

在本Item开头关于不可变类的规则说明，没有方法可以修改对象，并且它的所有属性必须是final的。

事实上，这些规则比实际需要的要强硬一些，其实可以有所放松来提高性能。 
事实上，任何方法都不能在对象的状态中产生外部可见的变化。 
然而，一些不可变类具有一个或多个非final属性，在第一次需要时将开销昂贵的计算结果缓存在这些属性中。 
如果再次请求相同的值，则返回缓存的值，从而节省了重新计算的成本。 
这个技巧的作用恰恰是因为对象是不可变的，这保证了如果重复的话，计算会得到相同的结果。

例如，PhoneNumber类的hashCode方法（第53页的Item 11）在第一次调用改方法时计算哈希码，并在再次调用时对其进行缓存。 
这种延迟初始化（Item 83）的一个例子，String类也使用到了。

关于序列化应该加上一个警告。 
如果你选择使您的不可变类实现Serializable接口，并且它包含一个或多个引用可变对象的属性，则必须提供显式的readObject或readResolve方法，或者使用ObjectOutputStream.writeUnshared和ObjectInputStream.readUnshared方法，即默认的序列化形式也是可以接受的。 
否则攻击者可能会创建一个可变的类的实例。 这个主题会在Item 88中会详细介绍。

# 总结

总而言之，坚决不要为每个属性编写一个get方法后再编写一个对应的set方法。 
除非有充分的理由使类成为可变类，否则类应该是不可变的。 
不可变类提供了许多优点，唯一的缺点是在某些情况下可能会出现性能问题。 
你应该始终使用较小的值对象（如PhoneNumber和Complex），使其不可变。 
（Java平台类库中有几个类，如java.util.Date和java.awt.Point，本应该是不可变的，但实际上并不是）。
你应该认真考虑创建更大的值对象，例如String和BigInteger ，设成不可改变的。 
只有当你确认有必要实现令人满意的性能（Item 67）时，才应该为不可改变类提供一个公开的可变伙伴类。

对于一些类来说，不变性是不切实际的。如果一个类不能设计为不可变类，那么也要尽可能地限制它的可变性。
减少对象可以存在的状态数量，可以更容易地分析对象，以及降低出错的可能性。
因此，除非有足够的理由把属性设置为非 final 的情况下，否则应该每个属性都设置为 final 的。
把本Item的建议与Item15的建议结合起来，你自然的倾向就是：除非有充分的理由不这样做，否则应该把每个属性声明为私有final的。

构造方法应该创建完全初始化的对象，并建立所有的不变性。 
除非有令人信服的理由，否则不要提供独立于构造方法或静态工厂的公共初始化方法。 
同样，不要提供一个“reinitialize”方法，使对象可以被重用，就好像它是用不同的初始状态构建的。 
这样的方法通常以增加的复杂度为代价，仅仅提供很少的性能优势。

CountDownLatch类是这些原理的例证。 
它是可变的，但它的状态空间有意保持最小范围内。 
创建一个实例，使用它一次，并完成：一旦countdown锁的计数器已经达到零，不能再重用它。

在这个Item中，应该添加关于Complex类的最后一个注释。 
这个例子只是为了说明不变性。 
这不是一个工业强度复杂的复数实现。 
它对复数使用了乘法和除法的标准公式，这些公式会进行不正确的四舍五入，没有为复数的NaN和无穷大提供良好的语义[Kahan91，Smith62，Thomas94]。
