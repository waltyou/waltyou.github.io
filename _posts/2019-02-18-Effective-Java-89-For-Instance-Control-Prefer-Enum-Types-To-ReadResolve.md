---
layout: post
title: 《Effective Java》学习日志（十一）89:为了控制实例，在readResolve方法中使用枚举类型
date: 2019-02-18 15:25:00
author: admin
comments: true
categories: [Java]
tags: [Java,Effective Java]


---



<!-- more -->

------

学习资料主要参考： 《Effective Java Third Edition》，作者：Joshua Bloch

------

## 目录
{:.no_toc}

* 目录
{:toc}

------

第3项描述了Singleton模式，并给出了单例类的以下示例。 此类限制对其构造函数的访问，以确保只创建一个实例：

```java
public class Elvis {
    public static final Elvis INSTANCE = new Elvis();
    private Elvis() { ... }
    public void leaveTheBuilding() { ... }
}
```

如第3项所述，如果将实现Serializable的单词添加到其声明中，则此类将不再是单例。 类是否使用默认的序列化表单或自定义序列化表单（第87项）并不重要，该类是否提供显式的readObject方法（第88项）也无关紧要。 任何readObject方法，无论是显式方法还是默认方法，都会返回一个新创建的实例，该实例与在类初始化时创建的实例不同。

readResolve功能允许您将另一个实例替换为 readObject 创建的实例。 如果被反序列化的对象的类使用适当的声明定义了readResolve方法，则在反序列化后对新创建的对象调用此方法。 然后返回此方法返回的对象引用来代替新创建的对象。 在此功能的大多数用途中，不保留对新创建的对象的引用，因此它立即有资格进行垃圾回收。

如果Elvis类用于实现Serializable，则以下read-Resolve方法足以保证singleton属性：

```java
// readResolve for instance control - you can do better!
private Object readResolve() {
    // Return the one true Elvis and let the garbage collector
    // take care of the Elvis impersonator.
    return INSTANCE;
}
```

此方法忽略反序列化对象，返回在初始化类时创建的区分Elvis实例。 因此，Elvis实例的序列化形式不需要包含任何实际数据; 应将所有实例字段声明为瞬态。实际上，**如果您依赖readResolve进行实例控制，则必须将具有对象引用类型的所有实例字段声明为transient**。 否则，确定的攻击者有可能在运行readResolve方法之前使用类似于第88项中的MutablePeriod攻击的技术来保护对反序列化对象的引用。

攻击有点复杂，但潜在的想法很简单。 如果单例包含非瞬态对象引用字段，则在运行单例的readResolve方法之前，将对该字段的内容进行反序列化。 这允许精心设计的流在反序列化对象引用字段的内容时“窃取”对原始反序列化单例的引用。

以下是它如何更详细地工作。 首先，编写一个“stealer”类，它同时具有readResolve方法和一个实例字段，该字段引用窃取程序“隐藏”的序列化单例。在序列化流中，将单例的非瞬态字段替换为窃取程序的实例。 你现在有一个圆形：单例包含窃取者，偷窃者指的是这个单例。

因为单例包含窃取程序，所以当单例被反序列化时，窃取程序的readResolve方法首先运行。 因此，当窃取程序的readResolve方法运行时，其实例字段仍然引用部分反序列化（并且尚未解析）的单例。

窃取程序的readResolve方法将引用从其实例字段复制到静态字段，以便在readResolve方法运行后访问引用。 然后，该方法返回其隐藏的字段的正确类型的值。 如果它没有这样做，当序列化系统试图将窃取者引用存储到该字段时，VM将抛出ClassCastException。

为了使这个具体，请考虑以下破坏的单例：

```java
// Broken singleton - has nontransient object reference field!
public class Elvis implements Serializable {
    public static final Elvis INSTANCE = new Elvis();
    private Elvis() { }
    private String[] favoriteSongs =
    			{ "Hound Dog", "Heartbreak Hotel" };
    public void printFavorites() {
    	System.out.println(Arrays.toString(favoriteSongs));
    }
    private Object readResolve() {
    	return INSTANCE;
    }
}
```

这是一个“窃取者”类，按照上面的描述构建：

```java
public class ElvisStealer implements Serializable {
    static Elvis impersonator;
    private Elvis payload;
    
    private Object readResolve() {
        // Save a reference to the "unresolved" Elvis instance
        impersonator = payload;
        // Return object of correct type for favoriteSongs field
        return new String[] { "A Fool Such as I" };
    }
    
    private static final long serialVersionUID = 0;
}
```

最后，这是一个丑陋的程序，它将手工流反序列化，以生成有缺陷的单例的两个不同实例。 deserialize方法在此程序中省略，因为它与第354页上的方法相同：

```java
public class ElvisImpersonator {
    // Byte stream couldn't have come from a real Elvis instance!
    private static final byte[] serializedForm = {
    (byte)0xac, (byte)0xed, 0x00, 0x05, 0x73, 0x72, 0x00, 0x05,
    0x45, 0x6c, 0x76, 0x69, 0x73, (byte)0x84, (byte)0xe6,
    (byte)0x93, 0x33, (byte)0xc3, (byte)0xf4, (byte)0x8b,
    0x32, 0x02, 0x00, 0x01, 0x4c, 0x00, 0x0d, 0x66, 0x61, 0x76,
    0x6f, 0x72, 0x69, 0x74, 0x65, 0x53, 0x6f, 0x6e, 0x67, 0x73,
    0x74, 0x00, 0x12, 0x4c, 0x6a, 0x61, 0x76, 0x61, 0x2f, 0x6c,
    0x61, 0x6e, 0x67, 0x2f, 0x4f, 0x62, 0x6a, 0x65, 0x63, 0x74,
    0x3b, 0x78, 0x70, 0x73, 0x72, 0x00, 0x0c, 0x45, 0x6c, 0x76,
    0x69, 0x73, 0x53, 0x74, 0x65, 0x61, 0x6c, 0x65, 0x72, 0x00,
    0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x02, 0x00, 0x01,
    0x4c, 0x00, 0x07, 0x70, 0x61, 0x79, 0x6c, 0x6f, 0x61, 0x64,
    0x74, 0x00, 0x07, 0x4c, 0x45, 0x6c, 0x76, 0x69, 0x73, 0x3b,
    0x78, 0x70, 0x71, 0x00, 0x7e, 0x00, 0x02
    };
    public static void main(String[] args) {
        // Initializes ElvisStealer.impersonator and returns
        // the real Elvis (which is Elvis.INSTANCE)
        Elvis elvis = (Elvis) deserialize(serializedForm);
        Elvis impersonator = ElvisStealer.impersonator;
        elvis.printFavorites();
        impersonator.printFavorites();
    }
}
```

运行此程序会产生以下输出，最终证明可以创建两个不同的Elvis实例（音乐中具有不同的品味）：

> [Hound Dog, Heartbreak Hotel]
> [A Fool Such as I]

您可以通过声明favoriteSongs字段瞬态来解决问题，但最好通过使Elvis成为单元素枚举类型来修复它（第3项）。 正如ElvisStealer攻击所证明的那样，使用readResolve方法来防止攻击者访问“临时”反序列化实例是非常脆弱的，需要非常小心。

如果将可序列化的实例控制类编写为枚举，Java会保证除了声明的常量之外不能有任何实例，除非攻击者滥用AccessibleObject.setAccessible等特权方法。 任何能够做到这一点的攻击者已经拥有足够的权限来执行任意本机代码，并且所有的赌注都已关闭。 以下是我们的Elvis示例如何看作枚举：

```java
// Enum singleton - the preferred approach
public enum Elvis {
    INSTANCE;
    private String[] favoriteSongs =
    		{ "Hound Dog", "Heartbreak Hotel" };
    
    public void printFavorites() {
    	System.out.println(Arrays.toString(favoriteSongs));
    }
}
```

使用readResolve进行实例控制并不是过时的。 如果必须编写一个可序列化的实例控制类，其实例在编译时是未知的，那么您将无法将该类表示为枚举类型。

readResolve的可访问性非常重要。 如果在最终类上放置readResolve方法，它应该是私有的。 如果将readResolve方法放在非最终类上，则必须仔细考虑其可访问性。 如果它是私有的，则不适用于任何子类。 如果它是包私有的，它将仅适用于同一包中的子类。 如果它是受保护的或公开的，它将适用于所有不覆盖它的子类。 如果readResolve方法受保护或公共，并且子类不覆盖它，则反序列化子类实例将生成一个超类实例，这可能会导致ClassCastException。

总而言之，使用枚举类型尽可能强制实例控制不变量。 如果这是不可能的，并且您需要一个类可序列化和实例控制，则必须提供readResolve方法并确保所有类的实例字段都是原始的或瞬态的。

