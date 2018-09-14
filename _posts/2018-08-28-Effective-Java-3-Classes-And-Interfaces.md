---
layout: post
title: 《Effective Java》学习日志（三）： 类与接口
date: 2018-08-28 15:11:04
author: admin
comments: true
categories: [Java]
tags: [Java,Effective Java]
---

类和接口是Java编程语言的核心。 它们是 Java 的基本抽象单位。 

<!-- more -->
---

学习资料主要参考： 《Effective Java Third Edition》，作者：Joshua Bloch

---
## 目录
{:.no_toc}

* 目录
{:toc}

---

# Item 15：最大限度地减少类和成员的可访问性

一个能区分组件是好是坏的重要因素，就是看这个组件隐藏其内部数据以及其他组件的实现细节的程度。
精心设计的组件隐藏其所有实现细节，将其API与其实现完全分离。 
组件然后只通过他们的API进行通信，并且忘记了彼此的内部工作。 
这种被称为信息隐藏或封装（information hiding or encapsulation）的概念是软件设计的基本原则。

信息隐藏很重要，原因很多，其中大部分原因在于它将构成系统的组件分离，允许它们单独开发，测试，优化，使用，理解和修改。
这可以加速系统开发，因为组件可以并行开发。
它减轻了维护的负担，因为可以更快地理解组件并进行调试或更换，而不必担心会损害其他组件。
虽然信息隐藏本身并不会导致良好的性能，但它可以实现有效的性能调整：
一旦系统完成并且分析确定哪些组件导致性能问题（Item 67），那些组件可以在不影响其他组件正确性的情况下进行优化。
信息隐藏增加了软件的重用，因为松耦合的组件，在除了开发之外的环境中，经常也是有用的。
最后，信息隐藏降低了构建大型系统的风险，因为即使整个系统没有成功，单个组件也可以成功。

Java有许多辅助信息隐藏的工具。 
访问控制机制指定类，接口和成员的可访问性。 
实体的可访问性由其声明的位置确定，并且声明中存在访问修饰符（private , protected , public）的位置（如果有）。 
正确使用这些修饰符对于信息隐藏至关重要。

经验法则很简单：**让每个班级或成员尽可能无法访问**。
换句话说，在保证软件的正常运行时，保持最低的访问级别。

对于顶层(非嵌套的)类和接口，只有两个可能的访问级别: 包级私有（package-private）和公共的（public）。
如果你使用public修饰符声明顶级类或接口，那么它是公开的；否则，它是包级私有的。
如果一个顶层类或接口可以被做为包级私有，那么它就应该这样被设置。
通过将其设置为包级私有，可以将其作为实现的一部分，而不是导出的API，你可以修改它、替换它，或者在后续版本中消除它，而不必担心损害现有的客户端。
如果你把它公开，你就有义务永远地支持它，以保持兼容性。

如果一个包级私有顶级类或接口只被一个类使用，那么可以考虑这个类作为使用它的唯一类的私有静态嵌套类(Item 24)。
这将它的可访问性从包级的所有类减少到使用它的一个类。
但是，减少不必要的公共类的可访问性要比包级私有的顶级类更重要：公共类是包的API的一部分，而包级私有的顶级类已经是这个包实现的一部分了。

对于成员(属性、方法、嵌套类和嵌套接口)，有四种可能的访问级别，在这里，按照可访问性从小到大列出：
- private： 该成员只能在声明它的顶级类内访问。
- package-private： 成员可以从被声明的包中的任何类中访问。从技术上讲，如果没有指定访问修饰符(接口成员除外，它默认是公共的)，这是默认访问级别。
- protected： 成员可以从被声明的类的子类中访问(受一些限制，JLS，6.6.2)，以及它声明的包中的任何类。
- public： 该成员可以从任何地方被访问。 

在仔细设计你的类的公共API之后，你的反应应该是让所有其他成员设计为私有的。 
只有当同一个包中的其他类真的需要访问成员时，需要删除私有修饰符，从而使成员包成为包级私有的。 
如果你发现自己经常这样做，你应该重新检查你的系统的设计，看看另一个分解可能产生更好的解耦的类。 
也就是说，私有成员和包级私有成员都是类实现的一部分，通常不会影响其导出的API。 
但是，如果类实现Serializable接口（Item 86和87），则这些属性可以“泄漏（leak）”到导出的API中。

对于公共类的成员，当访问级别从包私有到受保护级时，可访问性会大大增加。 
受保护（protected）的成员是类导出的API的一部分，并且必须永远支持。 
此外，导出类的受保护成员表示对实现细节的公开承诺（Item 19）。 
对受保护成员的需求应该相对较少。

有一个关键的规则限制了你减少方法访问性的能力。 
如果一个方法重写一个超类方法，那么它在子类中的访问级别就不能低于父类中的访问级别。 
这对于确保子类的实例在父类的实例可用的地方是可用的（Liskov替换原则，见 Item 15）是必要的。 
如果违反此规则，编译器将在尝试编译子类时生成错误消息。 
这个规则的一个特例是，如果一个类实现了一个接口，那么接口中的所有类方法都必须在该类中声明为public。

为了便于测试你的代码，你可能会想要让一个类，接口或者成员更容易被访问。 
这没问题。 
为了测试将公共类的私有成员指定为包级私有是可以接受的，但是提高到更高的访问级别却是不可接受的。 
换句话说，将类，接口或成员作为包级导出的API的一部分来促进测试是不可接受的。 
幸运的是，这不是必须的，因为测试可以作为被测试包的一部分运行，从而获得对包私有元素的访问。

公共类的实例属性很少公开(Item 16)。
如果一个实例属性是非 final 的，或者是对可变对象的引用，那么通过将其公开，你就放弃了限制可以存储在属性中的值的能力。
这意味着你放弃了执行涉及该属性的不变量的能力。
另外，当属性被修改时，就放弃了采取任何操作的能力，因此公共可变属性的类通常不是线程安全的。
即使属性是 final 的，并且引用了一个不可变的对象，通过使它公开，你就放弃切换到不存在属性的新的内部数据表示的灵活性。

同样的建议适用于静态属性，但有一个例外。 
假设常量是类的抽象的一个组成部分，你可以通过public static final属性暴露常量。 
按照惯例，这些属性的名字由大写字母组成，字母用下划线分隔（Item 68）。 
很重要的一点是，这些属性包含基本类型的值或对不可变对象的引用（Item 17）。 
包含对可变对象的引用的属性具有非final属性的所有缺点。 
虽然引用不能被修改，但引用的对象可以被修改，并会带来灾难性的结果。

请注意，非零长度的数组总是可变的，所以类具有公共静态final数组属性，或返回这样一个属性的访问器是错误的。 
如果一个类有这样的属性或访问方法，客户端将能够修改数组的内容。 这是安全漏洞的常见来源：

```java
// Potential security hole!
public static final Thing[] VALUES = { ... };
```

要小心这样的事实，一些IDE生成的访问方法返回对私有数组属性的引用，导致了这个问题。有两种方法可以解决这个问题。 
你可以使公共数组私有并添加一个公共的不可变列表：
```java
private static final Thing[] PRIVATE_VALUES = { ... };

public static final List<Thing> VALUES = Collections.unmodifiableList(Arrays.asList(PRIVATE_VALUES));
```

或者，可以将数组设置为private，并添加一个返回私有数组拷贝的公共方法：

```java
private static final Thing[] PRIVATE_VALUES = { ... };

public static final Thing[] values() {
    return PRIVATE_VALUES.clone();
}
```

要在这些方法之间进行选择，请考虑客户端可能如何处理返回的结果。 哪种返回类型会更方便？ 哪个会更好的表现？

在Java 9中，作为模块系统（module system）的一部分引入了两个额外的隐式访问级别。
模块包含一组包，就像一个包包含一组类一样。模块可以通过模块声明中的导出（export）声明显式地导出某些包(这是module-info.java的源文件中包含的约定)。
模块中的未导出包的公共和受保护成员在模块之外是不可访问的；在模块中，可访问性不受导出（export）声明的影响。
使用模块系统允许你在模块之间共享类，而不让它们对整个系统可见。
在未导出的包中，公共和受保护的公共类的成员会产生两个隐式访问级别，这是普通公共和受保护级别的内部类似的情况。
这种共享的需求是相对少见的，并且可以通过重新安排包中的类来消除。

与四个主要访问级别不同，这两个基于模块的级别主要是建议（advisory）。 
如果将模块的JAR文件放在应用程序的类路径而不是其模块路径中，那么模块中的包将恢复为非模块化行为：
包的公共类的所有公共类和受保护成员都具有其普通的可访问性，不管包是否由模块导出。 
新引入的访问级别严格执行的地方是JDK本身：Java类库中未导出的包在模块之外真正无法访问。

对于典型的Java程序员来说，不仅程序模块所提供的访问保护存在局限性，而且在本质上是很大程度上建议性的；
为了利用它，你必须把你的包组合成模块，在模块声明中明确所有的依赖关系，重新安排你的源码树层级，并采取特殊的行动来适应你的模块内任何对非模块化包的访问。 
现在说模块是否会在JDK之外得到广泛的使用还为时尚早。 
与此同时，除非你有迫切的需要，否则似乎最好避免它们。

总而言之，应该尽可能地减少程序元素的可访问性（在合理范围内）。 
在仔细设计一个最小化的公共API之后，你应该防止任何散乱的类，接口或成员成为API的一部分。 
除了作为常量的公共静态final属性之外，公共类不应该有公共属性。 
确保public static final属性引用的对象是不可变的。

---

# Item 16: 使用存取器方法（setter or getter）代替public属性

有时候你可能会写一些很简单的类来把一些实例属性聚集在一起，类似于下面的类：

```java
//这种退化的类不应该被设置成public
class Point {
	public double x;
	public double y;
}
```

这种类的属性可以被直接访问，所以会破坏封装性（Item 15）。
因为当你想调整这个类就必须同时调整相应的API，也不能强制使这些变量变得不可变，而且当变量被改变时你也很难做一些特殊的额外操作。
这样做是有悖于面向对象开发的，在面向对象开发中一般会把它改成私有的变量，然后设置他们的存取器方法（getter和setter）：

```java
class Point {
	private double x;
	private double y;
	public Point(double x, double y) {
		this.x = x;
		this.y = y;
	}
	public double getX() { return x; }
	public double getY() { return y; }
	public void setX(double x) { this.x = x; }
	public void setY(double y) { this.y = y; }
}
```

当一个类是公共类时应该强制遵守这样的规则：如果一个类可以在它的包外部被访问到，就应该提供访问器方法来保持类可被修改的灵活性。
如果你的类公开了自己数据域，那么将来当你想修改的时候就要考虑已经基于你的这些数据域开发的客户端可能已经遍布全球，你也应该有责任不影响它们的使用，
所以此时你想修改这些数据域将会面临困难。

然而，如果一个类是 package-private 或者是一个私有的内部嵌套类，那么公开它的数据域本身就没有什么错误—假设是这些数据域已经足以描述类的抽象性。
这种方式无论是在定义的时候还是在被其他人调用的时候都比访问器看起来更加简洁。
你可能会想这些类的内部数据不也会被调用者绑定并修改么？但此时也只有包内部的类才有权限修改。
而且如果是内部类，修改的范围会更小一步被限制在包含内部类的类中。
Java库中的很多类也违反了这条规则，直接公开了一些属性。
最常见的例子就是在java.awt包中的Point 和 Dimension类。
这样的类我们应该摒弃而不是模仿。
在 Item 67 中会介绍到，因为Dimension类的内部实现对外暴露，造成一系列性能问题一直延续到今天都没能解决。

虽然公共的类直接暴露属性给外部不可取，但是如果暴露的是不可变对象，危险就会降低很多。
因为这时候你不能修改类的内部数据，也不能当对象被读取的时候做一些违规操作，你能做的是更加加强其不可变性。
例如下面例子中的每一个实例变量代表一种时间：

```java
//公共类暴露不可变对象
public final class Time {
	private static final int HOURS_PER_DAY = 24;
	private static final int MINUTES_PER_HOUR = 60;
    
	public final int hour;
	public final int minute;
    
	public Time(int hour, int minute) {
		if (hour < 0 || hour >= HOURS_PER_DAY)
			throw new IllegalArgumentException("Hour: " + hour);
		if (minute < 0 || minute >= MINUTES_PER_HOUR)
			throw new IllegalArgumentException("Min: " + minute);
		this.hour = hour;
		this.minute = minute;
	}
	... // Remainder omitted
}
```

总结起来就是，public的类不能暴露可变对象给外部。
即使是不可变对象也同样具有一定危害。
然而，有时在一些设计中 package-private 或嵌套内部类中还是需要暴露可变或不可变对象的。

---

# Item 17: 最小化可变性

不可变类简单来说是它的实例不能被修改的类。 
包含在每个实例中的所有信息在对象的生命周期中是固定的，因此不会观察到任何变化。 
Java平台类库包含许多不可变的类，包括String类，基本类型包装类以及BigInteger类和BigDecimal类。 
有很多很好的理由：不可变类比可变类更容易设计，实现和使用。 
他们不太容易出错，更安全。

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
这种模式被称为函数式方法，因为方法返回将操作数应用于函数的结果，而不修改它们。 
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
一个简单的方法是为常用的值提供公共的静态 final常量。 例如，Complex类可能提供这些常量：

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
第一种办法，先猜测一下会经常用到哪些多步的操作，然后讲它们作为基本类型提供。
如果一个多步操作是作为一个基本类型提供的，那么不可变类就不必在每一步创建一个独立的对象。
在内部，不可变的类可以是任意灵活的。 
例如，BigInteger有一个包级私有的可变的“伙伴类（companion class）”，它用来加速多步操作，比如模幂运算（ modular exponentiation）。
出于前面所述的所有原因，使用可变伙伴类比使用BigInteger要困难得多。 
幸运的是，你不必使用它：BigInteger类的实现者为你做了很多努力。

如果你可以准确预测客户端要在你的不可变类上执行哪些复杂的操作，那么包级私有可变伙伴类的方式可以正常工作。
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

---

# Item 18: 组合优于继承

继承是实现代码重用的有效方式，但并不总是最好的工具。
使用不当，会导致脆弱的软件。 
在包中使用继承是安全的，其中子类和父类的实现都在同一个程序员的控制之下。
对应专门为了继承而设计的，并且有文档说明的类来说（条目 19），使用继承也是安全的。 
然而，从普通的具体类跨越包级边界继承，是危险的。 
提醒一下，本书使用“继承”一词来表示实现继承（当一个类继承另一个类时）。 
在这个项目中讨论的问题不适用于接口继承（当类实现接口或当接口继承另一个接口时）。

与方法调用不同，继承打破了封装。 
换句话说，一个子类依赖于其父类的实现细节来保证其正确的功能。 
父类的实现可能会从发布版本不断变化，如果是这样，子类可能会被破坏，即使它的代码没有任何改变。 
因此，一个子类必须与其超类一起更新而变化，除非父类的作者为了继承的目的而专门设计它，并对应有文档的说明。

为了具体说明，假设有一个使用HashSet的程序。 
为了调整程序的性能，需要查询HashSet，从创建它之后已经添加了多少个元素（不要和当前的元素数量混淆，当元素被删除时数量也会下降）。 
为了提供这个功能，编写了一个HashSet变体，它保留了尝试元素插入的数量，并导出了这个插入数量的一个访问方法。 
HashSet类包含两个添加元素的方法，分别是add和addAll，所以我们重写这两个方法：


```java
// Broken - Inappropriate use of inheritance!
public class InstrumentedHashSet<E> extends HashSet<E> {
    // The number of attempted element insertions
    private int addCount = 0;

    public InstrumentedHashSet() {
    }

    public InstrumentedHashSet(int initCap, float loadFactor) {
        super(initCap, loadFactor);
    }
    @Override public boolean add(E e) {
        addCount++;
        return super.add(e);
    }
    @Override public boolean addAll(Collection<? extends E> c) {
        addCount += c.size();
        return super.addAll(c);
    }
    public int getAddCount() {
        return addCount;
    }
}

```

这个类看起来很合理，但是不能正常工作。 
假设创建一个实例并使用addAll方法添加三个元素。 
顺便提一句，请注意，下面代码使用在Java 9中添加的静态工厂方法List.of来创建一个列表；
如果使用的是早期版本，请改为使用Arrays.asList：

```java
InstrumentedHashSet<String> s = new InstrumentedHashSet<>();
s.addAll(List.of("Snap", "Crackle", "Pop"));
```

我们期望getAddCount方法返回的结果是3，但实际上返回了6。
哪里出来问题？在HashSet内部，addAll方法是基于它的add方法来实现的，即使HashSet文档中没有指名其实现细节，倒也是合理的。
InstrumentedHashSet中的addAll方法首先给addCount属性设置为3，然后使用super.addAll方法调用了HashSet的addAll实现。
然后反过来又调用在InstrumentedHashSet类中重写的add方法，每个元素调用一次。
这三次调用又分别给addCount加1，所以，一共增加了6：通过addAll方法每个增加的元素都被计算了两次。

我们可以通过消除addAll方法的重写来“修复”子类。 
尽管生成的类可以正常工作，但是它依赖于它的正确方法，因为HashSet的addAll方法是在其add方法之上实现的。 
这个“自我使用（self-use）”是一个实现细节，并不保证在Java平台的所有实现中都可以适用，并且可以随发布版本而变化。 
因此，产生的InstrumentedHashSet类是脆弱的。

稍微好一点的做法是，重写addAll方法遍历指定集合，为每个元素调用add方法一次。 
不管HashSet的addAll方法是否在其add方法上实现，都会保证正确的结果，因为HashSet的addAll实现将不再被调用。
然而，这种技术并不能解决所有的问题。 
这相当于重新实现了父类方法，这样的方法可能不能确定到底是否时自用（self-use）的，实现起来也是困难的，耗时的，容易出错的，并且可能会降低性能。
此外，这种方式并不能总是奏效，因为子类无法访问一些私有属性，所以有些方法就无法实现。

导致子类脆弱的一个相关原因是，它们的父类在后续的发布版本中可以添加新的方法。
假设一个程序的安全性依赖于这样一个事实：所有被插入到集中的元素都满足一个先决条件。
可以通过对集合进行子类化，然后并重写所有添加元素的方法，以确保在添加每个元素之前满足这个先决条件，来确保这一问题。
如果在后续的版本中，父类没有新增添加元素的方法，那么这样做没有问题。
但是，一旦父类增加了这样的新方法，则很有肯能由于调用了未被重写的新方法，将非法的元素添加到子类的实例中。
这不是个纯粹的理论问题。
在把Hashtable和Vector类加入到Collections框架中的时候，就修复了几个类似性质的安全漏洞。

这两个问题都源于重写方法。 
如果仅仅添加新的方法并且不要重写现有的方法，可能会认为继承一个类是安全的。 
虽然这种扩展更为安全，但这并非没有风险。 
如果父类在后续版本中添加了一个新的方法，并且你不幸给了子类一个具有相同签名和不同返回类型的方法，那么你的子类编译失败[JLS，8.4.8.3]。 
如果已经为子类提供了一个与新的父类方法具有相同签名和返回类型的方法，那么你现在正在重写它，因此将遇到前面所述的问题。 
此外，你的方法是否会履行新的父类方法的约定，这是值得怀疑的，因为在你编写子类方法时，这个约定还没有写出来。

幸运的是，有一种方法可以避免上述所有的问题。
不要继承一个现有的类，而应该给你的新类增加一个私有属性，该属性是 现有类的实例引用，这种设计被称为组合（composition），因为现有的类成为新类的组成部分。
新类中的每个实例方法调用现有类的包含实例上的相应方法并返回结果。
这被称为**转发（forwarding）**，而新类中的方法被称为转发方法。
由此产生的类将坚如磐石，不依赖于现有类的实现细节。
即使将新的方法添加到现有的类中，也不会对新类产生影响。
为了具体说用，下面代码使用组合和转发方法替代InstrumentedHashSet类。
请注意，实现分为两部分，类本身和一个可重用的转发类，其中包含所有的转发方法，没有别的方法：

```java
// Reusable forwarding class
import java.util.Collection;
import java.util.Iterator;
import java.util.Set;

public class ForwardingSet<E> implements Set<E> {

    private final Set<E> s;

    public ForwardingSet(Set<E> s) {
        this.s = s;
    }

    public void clear() {
        s.clear();
    }

    public boolean contains(Object o) {
        return s.contains(o);
    }

    public boolean isEmpty() {
        return s.isEmpty();
    }

    public int size() {
        return s.size();
    }

    public Iterator<E> iterator() {
        return s.iterator();
    }

    public boolean add(E e) {
        return s.add(e);
    }

    public boolean remove(Object o) {
        return s.remove(o);
    }

    public boolean containsAll(Collection<?> c) {
        return s.containsAll(c);
    }

    public boolean addAll(Collection<? extends E> c) {
        return s.addAll(c);
    }

    public boolean removeAll(Collection<?> c) {
        return s.removeAll(c);
    }

    public boolean retainAll(Collection<?> c) {
        return s.retainAll(c);
    }

    public Object[] toArray() {
        return s.toArray();
    }

    public <T> T[] toArray(T[] a) {
        return s.toArray(a);
    }

    @Override
    public boolean equals(Object o) {
        return s.equals(o);
    }

    @Override
    public int hashCode() {
        return s.hashCode();
    }

    @Override
    public String toString() {
        return s.toString();
    }
}
// Wrapper class - uses composition in place of inheritance
import java.util.Collection;
import java.util.Set;

public class InstrumentedSet<E> extends ForwardingSet<E> {

    private int addCount = 0;

    public InstrumentedSet(Set<E> s) {
        super(s);
    }
    
    @Override public boolean add(E e) {
        addCount++;
        return super.add(e);
    }

    @Override public boolean addAll(Collection<? extends E> c) {
        addCount += c.size();
        return super.addAll(c);
    }

    public int getAddCount() {
        return addCount;
    }
}
```

InstrumentedSet类的设计是通过存在的Set接口来实现的，该接口包含HashSet类的功能特性。
除了功能强大，这个设计是非常灵活的。
InstrumentedSet类实现了Set接口，并有一个构造方法，其参数也是Set类型的。
本质上，这个类把Set转换为另一个类型Set， 同时添加了计数的功能。
与基于继承的方法不同，该方法仅适用于单个具体类，并且父类中每个需要支持构造方法，提供单独的构造方法，所以可以使用包装类来包装任何Set实现，并且可以与任何预先存在的构造方法结合使用：

```java
Set<Instant> times = new InstrumentedSet<>(new TreeSet<>(cmp));
Set<E> s = new InstrumentedSet<>(new HashSet<>(INIT_CAPACITY));
```

InstrumentedSet类甚至可以用于临时替换没有计数功能下使用的集合实例：

```java
static void walk(Set<Dog> dogs) {
    InstrumentedSet<Dog> iDogs = new InstrumentedSet<>(dogs);
    ... // Within this method use iDogs instead of dogs
}
```

InstrumentedSet类被称为包装类，因为每个InstrumentedSet实例都包含（“包装”）另一个Set实例。 
这也被称为装饰器模式[Gamma95]，因为InstrumentedSet类通过添加计数功能来“装饰”一个集合。 
有时组合和转发的结合被不精确地地称为委托（delegation）。 
从技术上讲，除非包装对象把自身传递给被包装对象，否则不是委托[Lieberman86;Gamma95]。

包装类的缺点很少。 
一个警告是包装类不适合在回调框架（callback frameworks）中使用，其中对象将自我引用传递给其他对象以用于后续调用（“回调”）。 
因为一个被包装的对象不知道它外面的包装对象，所以它传递一个指向自身的引用（this），回调时并不记得外面的包装对象。 
这被称为SELF问题[Lieberman86]。 
有些人担心转发方法调用的性能影响，以及包装对象对内存占用。 
两者在实践中都没有太大的影响。 
编写转发方法有些繁琐，但是只需为每个接口编写一次可重用的转发类，并且提供转发类。 
例如，Guava为所有的Collection接口提供转发类[Guava]。

只有在子类真的是父类的子类型的情况下，继承才是合适的。 
换句话说，只有在两个类之间存在“is-a”关系的情况下，B类才能继承A类。 
如果你试图让B类继承A类时，问自己这个问题：每个B都是A吗？ 如果你不能如实回答这个问题，那么B就不应该继承A。
如果答案是否定的，那么B通常包含一个A的私有实例，并且暴露一个不同的API：A不是B的重要部分 ，只是其实现细节。

在Java平台类库中有一些明显的违反这个原则的情况。 
例如，stacks实例并不是vector实例，所以Stack类不应该继承Vector类。 
同样，一个属性列表不是一个哈希表，所以Properties不应该继承Hashtable类。 
在这两种情况下，组合方式更可取。

如果在合适组合的地方使用继承，则会不必要地公开实现细节。
由此产生的API将与原始实现联系在一起，永远限制类的性能。
更严重的是，通过暴露其内部，客户端可以直接访问它们。
至少，它可能导致混淆语义。

例如，属性p指向Properties实例，那么 p.getProperty(key)和p.get(key)就有可能返回不同的结果：
前者考虑了默认的属性表，而后者是继承Hashtable的，它则没有考虑默认属性列表。
最严重的是，客户端可以通过直接修改超父类来破坏子类的不变性。
在Properties类，设计者希望只有字符串被允许作为键和值，但直接访问底层的Hashtable允许违反这个不变性。
一旦违反，就不能再使用属性API的其他部分（load和store方法）。
在发现这个问题的时候，纠正这个问题为时已晚，因为客户端依赖于使用非字符串键和值了。

在决定使用继承来代替组合之前，你应该问自己最后一组问题。
对于试图继承的类，它的API有没有缺陷呢？ 
如果有，你是否愿意将这些缺陷传播到你的类的API中？
继承传播父类的API中的任何缺陷，而组合可以让你设计一个隐藏这些缺陷的新API。

总之，继承是强大的，但它是有问题的，因为它违反封装。 
只有在子类和父类之间存在真正的子类型关系时才适用。 
即使如此，如果子类与父类不在同一个包中，并且父类不是为继承而设计的，继承可能会导致脆弱性。 
为了避免这种脆弱性，使用合成和转发代替继承，特别是如果存在一个合适的接口来实现包装类。 
包装类不仅比子类更健壮，而且更强大。

---

# 未完待续。。。。。。
