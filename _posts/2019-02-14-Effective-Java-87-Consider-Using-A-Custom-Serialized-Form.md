---
layout: post
title: 《Effective Java》学习日志（十一）87:考虑使用自定义的序列化格式
date: 2019-02-14 13:24:00
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

当您在时间压力下编写类时，通常应该集中精力设计最佳API。 有时这意味着发布一个“一次性”实现，你知道你将在未来的版本中替换它。 通常这不是问题，但如果类实现了Serializable并使用了默认的序列化形式，那么您将无法完全逃离一次性实现。 它将永远决定序列化的形式。 这不仅仅是一个理论问题。 它发生在Java库中的几个类中，包括BigInteger。

**如果没有考虑是否合适，请不要接受默认的序列化格式**。 接受默认的序列化形式应该有意识地决定从灵活性，性能和正确性的角度来看这种编码是合理的。 一般来说，只有在与设计自定义序列化格式时所选择的编码大致相同的情况下，才应接受默认的序列化格式。

对象的默认序列化形式是以对象为起点的对象图的物理表示的合理有效编码。 换句话说，它描述了对象中以及可从此对象访问的每个对象中包含的数据。 它还描述了所有这些对象相互链接的拓扑。 对象的理想序列化形式仅包含对象表示的逻辑数据。 它独立于物理表示。



**如果对象的物理表示与其逻辑内容相同，则默认的序列化格式可能是合适的**。 例如，对于以下类，默认的序列化形式是合理的，这简单地表示一个人的姓名：

```java
// Good candidate for default serialized form
public class Name implements Serializable {
/**
* Last name. Must be non-null.
* @serial
*/
private final String lastName;
/**
* First name. Must be non-null.
* @serial
*/
private final String firstName;
/**
* Middle name, or null if there is none.
* @serial
*/
private final String middleName;
	... // Remainder omitted
}
```

从逻辑上讲，名称由三个字符串组成，这三个字符串代表姓氏，名字和中间名。 Name中的实例字段精确地镜像了这个逻辑内容。

**即使您确定默认的序列化格式是合适的，您通常也必须提供readObject方法以确保不变量和安全性**。 对于Name，readObject方法必须确保字段lastName和firstName为非null。

请注意，对lastName，firstName和middleName字段有文档注释，即使它们是私有的。 这是因为这些私有字段定义了一个公共API，它是该类的序列化形式，并且必须记录此公共API。 @serial标记的存在告诉Javadoc将此文档放在一个记录序列化格式的特殊页面上。

在Name的频谱的另一端附近，考虑下面的类，它代表一个字符串列表（忽略你可能最好使用一个标准的List实现）：

```java
// Awful candidate for default serialized form
public final class StringList implements Serializable {
	private int size = 0;
    private Entry head = null;
    
    private static class Entry implements Serializable {
        String data;
        Entry next;
        Entry previous;
    }
    ... // Remainder omitted
}
```

**当对象的物理表示与其逻辑数据内容显着不同时，使用默认的序列化格式有四个缺点**：

- **它将导出的API永久绑定到当前内部表示**。 在上面的示例中，私有StringList.Entry类成为公共API的一部分。 如果在将来的版本中更改了表示，则StringList类仍需要接受输入上的链表表示并在输出时生成它。 该类永远不会消除处理链表条目的所有代码，即使它不再使用它们。
- **它会消耗过多的空间**。 在上面的示例中，序列化格式不必要地表示链接列表中的每个条目和所有链接。 这些条目和链接仅仅是实现细节，不值得包含在序列化形式中。 由于序列化格式过大，将其写入磁盘或通过网络发送将非常慢。
- **它可能会消耗过多的时间**。 序列化逻辑不了解对象图的拓扑结构，因此必须经历昂贵的图遍历。 在上面的例子中，仅仅遵循下一个引用就足够了。
- **它可能导致堆栈溢出**。 默认的序列化过程执行对象图的递归遍历，即使对于中等大小的对象图，也可能导致堆栈溢出。 

StringList的合理序列化形式只是列表中的字符串数，后跟字符串本身。 这构成了由StringList表示的逻辑数据，剥离了其物理表示的细节。 这是StringList的修订版本，其中包含实现此序列化形式的writeObject和readObject方法。 提醒一下，transient修饰符指示要从类的默认序列化形式中省略实例字段：

```java
// StringList with a reasonable custom serialized form
public final class StringList implements Serializable {
    private transient int size = 0;
    private transient Entry head = null;
    
    // No longer Serializable!
    private static class Entry {
        String data;
        Entry next;
        Entry previous;
    }
    
    // Appends the specified string to the list
    public final void add(String s) { ... }
    /**
    * Serialize this {@code StringList} instance.
    **
    @serialData The size of the list (the number of strings
    * it contains) is emitted ({@code int}), followed by all of
    * its elements (each a {@code String}), in the proper
    * sequence.
    */
    private void writeObject(ObjectOutputStream s)
        			throws IOException {
        s.defaultWriteObject();
        s.writeInt(size);
        // Write out all elements in the proper order.
        for (Entry e = head; e != null; e = e.next)
        	s.writeObject(e.data);
    }
    private void readObject(ObjectInputStream s)
        			throws IOException, ClassNotFoundException {
        s.defaultReadObject();
        int numElements = s.readInt();
        // Read in all elements and insert them in list
        for (int i = 0; i < numElements; i++)
        	add((String) s.readObject());
    }
    ... // Remainder omitted
}
```

writeObject做的第一件事就是调用defaultWriteObject，而readObject做的第一件事就是调用defaultReadObject，即使所有StringList的字段都是瞬态的。 您可能会听到它说如果所有类的实例字段都是瞬态的，您可以省去调用defaultWriteObject和defaultReadObject，但序列化规范要求您无论如何都要调用它们。 这些调用的存在使得可以在以后的版本中添加非瞬态实例字段，同时保持向后和向前兼容性。 如果实例在更高版本中序列化并在早期版本中反序列化，则添加的字段将被忽略。 如果早期版本的readObject方法无法调用defaultReadObject，则反序列化将因StreamCorruptedException而失败。

请注意，writeObject方法有一个文档注释，即使它是私有的。 这类似于Name类中私有字段的文档注释。 此私有方法定义了一个公共API，它是序列化形式，并且应该记录公共API。 与字段的@serial标记一样，方法的@serialData标记告诉Javadoc实用程序将此文档放在序列化格式页面上。

为了给早期的性能讨论带来一定的规模感，如果平均字符串长度为10个字符，则StringList的修订版本的序列化形式占用原始序列化形式的大约一半的空间。 在我的机器上，序列化StringList的修订版本的速度是序列化原始版本的两倍，列表长度为10。 最后，修订后的格式中没有堆栈溢出问题，因此没有可串行化的StringList大小的实际上限。

虽然默认的序列化形式对于StringList来说是不好的，但是有些类会更糟糕。 对于StringList，默认的序列化形式是不灵活的并且执行得很糟糕，但是在序列化和反序列化StringList实例的意义上，它产生了原始对象的忠实副本，其所有不变量都是完整的。 对于其不变量与特定于实现的详细信息相关联的任何对象，情况并非如此。

例如，考虑哈希表的情况。 物理表示是包含键值条目的一系列散列桶。 条目所在的桶是其密钥的哈希码的函数，通常，从实现到实现，它通常不保证是相同的。 实际上，从运行到运行甚至都不能保证相同。 因此，接受哈希表的默认序列化格式将构成严重错误。 序列化和反序列化哈希表可能会产生一个不变量严重损坏的对象。

无论您是否接受默认的序列化格式，当调用defaultWriteObject方法时，每个未标记为transient的实例字段都将被序列化。 因此，每个可以声明为瞬态的实例字段都应该是。 这包括派生字段，其值可以从主数据字段计算，例如缓存的哈希值。 它还包括其值与JVM的一个特定运行相关联的字段，例如表示指向本机数据结构的指针的长字段。 **在决定使字段为 nontransient 之前，请说服自己它的值是对象逻辑状态的一部分**。 如果使用自定义序列化格式，则大多数或所有实例字段都应标记为瞬态，如上面的StringList示例中所示。

如果您使用的是默认序列化表单并且已将一个或多个字段标记为瞬态，请记住，在反序列化实例时，这些字段将初始化为其默认值：对象引用字段为null，数字基本字段为零，false为 布尔字段[JLS，4.12.5]。 如果这些值对于任何瞬态字段都是不可接受的，则必须提供一个readObject方法，该方法调用defaultReadObject方法，然后将瞬态字段恢复为可接受的值（第88项）。 或者，这些字段可以在第一次使用时进行延迟初始化（第83项）。

无论是否使用默认的序列化表单，**都必须在对象序列化上强制执行任何同步，这些同步将强加到读取对象的整个状态的任何其他方法上**。 因此，例如，如果您有一个线程安全的对象（Item 82）通过同步每个方法来实现其线程安全，并且您选择使用默认的序列化表单，请使用以下write-Object方法：

```java
// writeObject for synchronized class with default serialized form
private synchronized void writeObject(ObjectOutputStream s)
    			throws IOException {
    s.defaultWriteObject();
}
```

如果在writeObject方法中放置同步，则必须确保它遵循与其他活动相同的锁定顺序约束，否则您将面临资源排序死锁的风险。

**无论选择哪种序列化形式，在您编写的每个可序列化类中声明显式串行版本UID**。 这消除了串行版本UID作为不兼容的潜在来源（第86项）。 还有一个小的性能优势。 如果没有提供串行版本UID，则执行昂贵的计算以在运行时生成一个。

声明串行版UID很简单。 只需将此行添加到您的类中：

```java
private static final long serialVersionUID = randomLongValue;
```

如果您编写一个新类，则为randomLongValue选择的值无关紧要。 您可以通过在类上运行serialver实用程序来生成值，但也可以凭空挑选一个数字。 串行版本UID不是唯一的。 如果修改缺少串行版本UID的现有类，并且希望新版本接受现有的序列化实例，则必须使用为旧版本自动生成的值。 您可以通过在类的旧版本上运行serialver实用程序来获取此编号，该类型是存在序列化实例的类。

如果您想要创建与现有版本不兼容的类的新版本，只需更改串行版本UID声明中的值。 这将导致尝试反序列化先前版本的序列化实例以抛出InvalidClassException。 **除非您要破坏与类的所有现有序列化实例的兼容性，否则请勿更改串行版本UID。**

总而言之，如果您已确定某个类应该可序列化（第86项），请仔细考虑序列化表单应该是什么。 仅当它是对象逻辑状态的合理描述时，才使用默认的序列化形式; 否则设计一个适当描述对象的自定义序列化表单。 在分配设计导出方法时，您应该分配尽可能多的时间来设计类的序列化形式（第51项）。 正如您无法从将来的版本中删除导出的方法一样，您无法从序列化表单中删除字段; 必须永久保存它们以确保序列化兼容性。 选择错误的序列化表单会对类的复杂性和性能产生永久性的负面影响。
