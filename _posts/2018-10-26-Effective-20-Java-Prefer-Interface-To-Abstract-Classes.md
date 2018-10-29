---
layout: post
title: 《Effective Java》学习日志(三) -- 20: 接口优于抽象类
date: 2018-08-28 15:11:04
author: admin
comments: true
categories: [Java]
tags: [Java,Effective Java]
---

Java 有两种机制来定义多实现的类：接口与抽象类。但接口优于抽象类，来看看为什么。

<!-- more -->
---

学习资料主要参考： 《Effective Java Third Edition》，作者：Joshua Bloch

---
## 目录
{:.no_toc}

* 目录
{:toc}

---

# 不同之处

在 Java 8 之后， 接口中引入了 default 方法，如此之后，接口和抽象类现在都允许有实现的方法在其中。

但是它俩之间最大的不同就是：一个类只能继承一个抽象类，这是由于 Java 单继承的机制。这个机制严重的限制了使用抽象类的灵活性。

但是接口不一样，它允许多继承。

# 任何存在的类都可以通过实现接口来轻易的改装

你只需添加所需的方法（如果尚不存在的话），并向类声明中添加一个implements子句。 
例如，当Comparable, Iterable， 和Autocloseable接口添加到Java平台时，很多现有类需要实现它们来加以改进。 

一般来说，现有的类不能改进以继承一个新的抽象类。 
如果你想让两个类继承相同的抽象类，你必须把它放在类型层级结构中的上面位置，它是两个类的祖先。 
不幸的是，这会对类型层级结构造成很大的附带损害，迫使新的抽象类的所有后代对它进行子类化，无论这些后代类是否合适。

# 接口是定义混合类型（mixin）的理想选择

一般来说，mixin是一个类，除了它的“主类型”之外，还可以声明它提供了一些可选的行为。 

例如，Comparable是一个类型接口，它允许一个类声明它的实例相对于其他可相互比较的对象是有序的。 
这样的接口被称为类型，因为它允许可选功能被“混合”到类型的主要功能。 

抽象类不能用于定义混合类，这是因为它们不能被加载到现有的类中：一个类不能有多个父类，并且在类层次结构中没有合理的位置来插入一个类型。

# 接口允许构建非层级类型的框架

类型层级对于组织某些事物来说是很好的，但是其他的事物并不是整齐地落入严格的层级结构中。 

例如，假设我们有一个代表歌手的接口，另一个代表作曲家的接口：

```java
public interface Singer {
    AudioClip sing(Song s);
}

public interface Songwriter {
    Song compose(int chartPosition);
}
```

在现实生活中，一些歌手也是作曲家。 因为我们使用接口而不是抽象类来定义这些类型，所以单个类实现歌手和作曲家两个接口是完全允许的。 
事实上，我们可以定义一个继承歌手和作曲家的第三个接口，并添加适合于这个组合的新方法：
```java
public interface SingerSongwriter extends Singer, Songwriter {
     AudioClip strum();
 
     void actSensitive();
 }
```

你并不总是需要这种灵活性，但是当你这样做的时候，接口是一个救星。 

另一种方法是对于每个受支持的属性组合，包含一个单独的类的臃肿类层级结构。 
如果类型系统中有n个属性，则可能需要支持2^n种可能的组合。 
这就是所谓的组合爆炸（combinatorial explosion）。 
臃肿的类层级结构可能会导致具有许多方法的臃肿类，这些方法仅在参数类型上有所不同，因为类层级结构中没有类型来捕获通用行为。

# 接口通过包装类模式确保安全的，强大的功能增强成为可能

如果使用抽象类来定义类型，那么就让程序员想要添加功能，只能继承。 生成的类比包装类更弱，更脆弱。

当其他接口方法有明显的接口方法实现时，可以考虑向程序员提供默认形式的方法实现帮助。 
有关此技术的示例，请参阅第104页的removeIf方法。
如果提供默认方法，请确保使用**@implSpec** Javadoc标记（条目19）将它们文档说明为继承。

使用默认方法可以提供的帮助多多少少是有些限制的。 
尽管许多接口指定了Object类中方法（如equals和hashCode）的行为，但不允许为它们提供默认方法。 
此外，接口不允许包含实例属性或非公共静态成员（私有静态方法除外）。 
最后，不能将默认方法添加到不受控制的接口中。

但是，你可以通过提供一个抽象的骨架实现类（abstract skeletal implementation class）来与接口一起使用，将接口和抽象类的优点结合起来。 
接口定义了类型，可能提供了一些默认的方法，而骨架实现类在原始接口方法的顶层实现了剩余的非原始接口方法。 
继承骨架实现需要大部分的工作来实现一个接口。 这就是模板方法设计模式[Gamma95]。

按照惯例，骨架实现类被称为AbstractInterface，其中Interface是它们实现的接口的名称。
例如，集合框架（ Collections Framework）提供了一个框架实现以配合每个主要集合接口：AbstractCollection，AbstractSet，AbstractList和AbstractMap。 
可以说，将它们称为SkeletalCollection，SkeletalSet，SkeletalList和SkeletalMap是有道理的，但是现在已经确立了抽象约定。 
如果设计得当，骨架实现（无论是单独的抽象类还是仅由接口上的默认方法组成）可以使程序员非常容易地提供他们自己的接口实现。 

例如，下面是一个静态工厂方法，在AbstractList的顶层包含一个完整的功能齐全的List实现：

```java
// Concrete implementation built atop skeletal implementation
static List<Integer> intArrayAsList(int[] a) {
    Objects.requireNonNull(a);
    // The diamond operator is only legal here in Java 9 and later
    // If you're using an earlier release, specify <Integer>
    return new AbstractList<>() {
        @Override public Integer get(int i) {
            return a[i]; // Autoboxing (Item 6)
        }
        @Override public Integer set(int i, Integer val) {
            int oldVal = a[i];
            a[i] = val; // Auto-unboxing
            return oldVal; // Autoboxing
        }
        @Override public int size() {
            return a.length;
        }
    };
}
```
当你考虑一个List实现为你做的所有事情时，这个例子是一个骨架实现的强大的演示。 
顺便说一句，这个例子是一个适配器（Adapter ）[Gamma95]，它允许一个int数组被看作Integer实例列表。 
由于int值和整数实例（装箱和拆箱）之间的来回转换，其性能并不是非常好。 请注意，实现采用匿名类的形式（条目 24）。

骨架实现类的优点在于，它们提供抽象类的所有实现的帮助，而不会强加抽象类作为类型定义时的严格约束。
对于具有骨架实现类的接口的大多数实现者来说，继承这个类是显而易见的选择，但它不是必需的。
如果一个类不能继承骨架的实现，这个类可以直接实现接口。
该类仍然受益于接口本身的任何默认方法。

此外，骨架实现类仍然可以协助接口的实现。
实现接口的类可以将接口方法的调用转发给继承骨架实现的私有内部类的包含实例。
这种被称为模拟多重继承的技术与条目 18讨论的包装类模式密切相关。
它提供了多重继承的许多好处，同时避免了缺陷。

编写一个骨架的实现是一个相对简单的过程，虽然有些乏味。 
首先，研究接口，并确定哪些方法是基本的，其他方法可以根据它们来实现。 
这些基本方法是你的骨架实现类中的抽象方法。 
接下来，为所有可以直接在基本方法之上实现的方法提供接口中的默认方法，回想一下，你可能不会为诸如Object类中equals和hashCode等方法提供默认方法。 
如果基本方法和默认方法涵盖了接口，那么就完成了，并且不需要骨架实现类。 
否则，编写一个声明实现接口的类，并实现所有剩下的接口方法。 
为了适合于该任务，此类可能包含任何的非公共属性和方法。

作为一个简单的例子，考虑一下Map.Entry接口。 
显而易见的基本方法是getKey，getValue和（可选的）setValue。 
接口指定了equals和hashCode的行为，并且在基本方面方面有一个toString的明显的实现。 
由于不允许为Object类方法提供默认实现，因此所有实现均放置在骨架实现类中：

```java
// Skeletal implementation class
public abstract class AbstractMapEntry<K,V> implements Map.Entry<K,V> {
    // Entries in a modifiable map must override this method
    @Override public V setValue(V value) {
        throw new UnsupportedOperationException();
    }
    // Implements the general contract of Map.Entry.equals
    @Override public boolean equals(Object o) {
        if (o == this)
            return true;
        if (!(o instanceof Map.Entry))
            return false;
        Map.Entry<?,?> e = (Map.Entry) o;
        return Objects.equals(e.getKey(), getKey())
                    && Objects.equals(e.getValue(), getValue());
    }
    // Implements the general contract of Map.Entry.hashCode
    @Override public int hashCode() {
        return Objects.hashCode(getKey())
                    ^ Objects.hashCode(getValue());
    }
    @Override public String toString() {
        return getKey() + "=" + getValue();
    }
}
```
请注意，这个骨架实现不能在Map.Entry接口中实现，也不能作为子接口实现，因为默认方法不允许重写诸如equals，hashCode和toString等Object类方法。

由于骨架实现类是为了继承而设计的，所以你应该遵循条目 19中的所有设计和文档说明。
为了简洁起见，前面的例子中省略了文档注释，但是**好的文档在骨架实现中是绝对必要的**，无论它是否包含 一个接口或一个单独的抽象类的默认方法。

与骨架实现有稍许不同的是简单实现，以AbstractMap.SimpleEntry为例。 
一个简单的实现就像一个骨架实现，它实现了一个接口，并且是为了继承而设计的，但是它的不同之处在于它不是抽象的：它是最简单的工作实现。 
你可以按照情况使用它，也可以根据情况进行子类化。

总而言之，一个接口通常是定义允许多个实现的类型的最佳方式。 
如果你导出一个重要的接口，应该强烈考虑提供一个骨架的实现类。 
在可能的情况下，应该通过接口上的默认方法提供骨架实现，以便接口的所有实现者都可以使用它。 
也就是说，对接口的限制通常要求骨架实现类采用抽象类的形式。





