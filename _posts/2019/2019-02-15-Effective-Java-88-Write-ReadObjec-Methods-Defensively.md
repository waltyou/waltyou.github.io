---
layout: post
title: 《Effective Java》学习日志（十一）88:防御性地编写readObject方法
date: 2019-02-15 13:52:00
author: admin
comments: true
categories: [Java]
tags: [Java,Effective Java]

---



<!-- more -->

------

学习资料主要参考： 《Effective Java Third Edition》，作者：Joshua Bloch

------




* 目录
{:toc}

------

Item 50包含一个带有可变私有Date字段的不可变日期范围类。 该类通过在其构造函数和访问器中防御性地复制Date对象，竭尽全力保留其不变量和不变性。 这个类如下所示：

```java
// Immutable class that uses defensive copying
public final class Period {
    private final Date start;
    private final Date end;
    /**
    * @param start the beginning of the period
    * @param end the end of the period; must not precede start
    * @throws IllegalArgumentException if start is after end
    * @throws NullPointerException if start or end is null
    */
    public Period(Date start, Date end) {
        this.start = new Date(start.getTime());
        this.end = new Date(end.getTime());
        if (this.start.compareTo(this.end) > 0)
        	throw new IllegalArgumentException(
        		start + " after " + end);
    }
    
    public Date start () { return new Date(start.getTime()); }
    public Date end () { return new Date(end.getTime()); }
    public String toString() { return start + " - " + end; }
    ... // Remainder omitted
}
```

假设您决定要将此类序列化。 由于Period对象的物理表示形式恰好反映了其逻辑数据内容，因此使用默认的序列化表单并不合理（第87项）。 因此，似乎要使类可序列化所需要做的就是将实现Serializable的单词添加到类声明中。 但是，如果你这样做了，那么这个类将不再保证它的关键不变量。

问题是readObject方法实际上是另一个公共构造函数，它需要与任何其他构造函数一样的小心。 正如构造函数必须检查其参数的有效性（第49项）并在适当的地方制作参数的防御性副本（第50项），因此必须使用readObject方法。 如果readObject方法无法执行这些操作中的任何一个，则攻击者违反类的不变量是相对简单的事情。

简而言之，readObject是一个构造函数，它将字节流作为唯一参数。 在正常使用中，字节流是通过序列化正常构造的实例生成的。 当readObject被呈现为字节流时，问题出现了，该字节流被人工构造以生成违反其类的不变量的对象。 这样的字节流可用于创建一个不可能的对象：该对象无法使用普通构造函数创建。

假设我们只是将工具Serializable添加到Period的类声明中。 然后，这个丑陋的程序将生成一个Period实例，其结束在其开始之前。 设置高顺序位的字节值的强制转换是Java缺少字节文字的结果，并且不幸的决定使字节类型符号化：

```java
public class BogusPeriod {
    // Byte stream couldn't have come from a real Period instance!
    private static final byte[] serializedForm = {
        (byte)0xac, (byte)0xed, 0x00, 0x05, 0x73, 0x72, 0x00, 0x06,
        0x50, 0x65, 0x72, 0x69, 0x6f, 0x64, 0x40, 0x7e, (byte)0xf8,
        0x2b, 0x4f, 0x46, (byte)0xc0, (byte)0xf4, 0x02, 0x00, 0x02,
        0x4c, 0x00, 0x03, 0x65, 0x6e, 0x64, 0x74, 0x00, 0x10, 0x4c,
        0x6a, 0x61, 0x76, 0x61, 0x2f, 0x75, 0x74, 0x69, 0x6c, 0x2f,
        0x44, 0x61, 0x74, 0x65, 0x3b, 0x4c, 0x00, 0x05, 0x73, 0x74,
        0x61, 0x72, 0x74, 0x71, 0x00, 0x7e, 0x00, 0x01, 0x78, 0x70,
        0x73, 0x72, 0x00, 0x0e, 0x6a, 0x61, 0x76, 0x61, 0x2e, 0x75,
        0x74, 0x69, 0x6c, 0x2e, 0x44, 0x61, 0x74, 0x65, 0x68, 0x6a,
        (byte)0x81, 0x01, 0x4b, 0x59, 0x74, 0x19, 0x03, 0x00, 0x00,
        0x78, 0x70, 0x77, 0x08, 0x00, 0x00, 0x00, 0x66, (byte)0xdf,
        0x6e, 0x1e, 0x00, 0x78, 0x73, 0x71, 0x00, 0x7e, 0x00, 0x03,
        0x77, 0x08, 0x00, 0x00, 0x00, (byte)0xd5, 0x17, 0x69, 0x22,
        0x00, 0x78
    };
    public static void main(String[] args) {
        Period p = (Period) deserialize(serializedForm);
        System.out.println(p);
    }
    // Returns the object with the specified serialized form
    static Object deserialize(byte[] sf) {
        try {
            return new ObjectInputStream(
            	new ByteArrayInputStream(sf)).readObject();
        } catch (IOException | ClassNotFoundException e) {
        	throw new IllegalArgumentException(e);
        }
    }
}
```

用于初始化serializedForm的字节数组文字是通过序列化正常的Period实例并手动编辑生成的字节流生成的。 流的细节对于该示例并不重要，但是如果您好奇，则在Java对象序列化规范[序列化，6]中描述了序列化字节流格式。 如果您运行此程序，它将打印“Fri Jan 01 12:00:00 PST 1999  -  Sun Jan 01 12:00:00 PST 1984”.只需声明Period serializable，我们就可以创建一个违反其类不变量的对象。

要解决此问题，请为Period调用defaultReadObject提供readObject方法，然后检查反序列化对象的有效性。 如果有效性检查失败，则readObject方法将抛出InvalidObjectException，从而阻止反序列化完成：

```java
// readObject method with validity checking - insufficient!
private void readObject(ObjectInputStream s)
    		throws IOException, ClassNotFoundException {
    s.defaultReadObject();
    // Check that our invariants are satisfied
    if (start.compareTo(end) > 0)
    	throw new InvalidObjectException(start +" after "+ end);
}
```

虽然这可以防止攻击者创建无效的Period实例，但仍然存在潜在的更微妙的问题。 可以通过构造以有效Period实例开头的字节流来创建可变Period周期实例，然后将额外引用附加到Period实例内部的私有Date字段。 攻击者从ObjectInputStream中读取Period实例，然后读取附加到流的“恶意对象引用”。 这些引用使攻击者可以访问Period对象中私有Date字段引用的对象。 通过改变这些Date实例，攻击者可以改变Period实例。 以下类演示了此攻击：

```java
public class MutablePeriod {
    // A period instance
    public final Period period;
    // period's start field, to which we shouldn't have access
    public final Date start;
    // period's end field, to which we shouldn't have access
    public final Date end;
    
    public MutablePeriod() {
        try {
            ByteArrayOutputStream bos = new ByteArrayOutputStream();
            ObjectOutputStream out = new ObjectOutputStream(bos);
            // Serialize a valid Period instance
            out.writeObject(new Period(new Date(), new Date()));
            /*
            * Append rogue "previous object refs" for internal
            * Date fields in Period. For details, see "Java
            * Object Serialization Specification," Section 6.4.
            */
            byte[] ref = { 0x71, 0, 0x7e, 0, 5 }; // Ref #5
            bos.write(ref); // The start field
            ref[4] = 4; // Ref # 4
            bos.write(ref); // The end field
            // Deserialize Period and "stolen" Date references
            ObjectInputStream in = new ObjectInputStream( 
            				new ByteArrayInputStream(bos.toByteArray()));
            period = (Period) in.readObject();
            start = (Date) in.readObject();
            end = (Date) in.readObject();
        } catch (IOException | ClassNotFoundException e) {
        	throw new AssertionError(e);
        }
    }
}
```

要查看正在进行的攻击，请运行以下程序：

```java
public static void main(String[] args) {
    MutablePeriod mp = new MutablePeriod();
    Period p = mp.period;
    Date pEnd = mp.end;
    // Let's turn back the clock
    pEnd.setYear(78);
    System.out.println(p);
    // Bring back the 60s!
    pEnd.setYear(69);
    System.out.println(p);
}
```

在我的语言环境中，运行此程序会产生以下输出：

> Wed Nov 22 00:21:29 PST 2017 - Wed Nov 22 00:21:29 PST 1978
> Wed Nov 22 00:21:29 PST 2017 - Sat Nov 22 00:21:29 PST 1969

虽然创建了Period实例且其不变量保持不变，但可以随意修改其内部组件。 一旦拥有可变的Period实例，攻击者可能会通过将实例传递给依赖于Period的安全性不变性的类来造成巨大的伤害。 这不是那么牵强：有些类依赖于String的安全性不变性。

问题的根源是Period的readObject方法没有做足够的防御性复制。 **对象反序列化时，防御性地复制包含客户端不得拥有的对象引用的任何字段至关重要**。 因此，每个包含私有可变组件的可序列化不可变类必须在其readObject方法中防御性地复制这些组件。 以下readObject方法足以确保Period的不变量并保持其不变性：

```java
// readObject method with defensive copying and validity checking
private void readObject(ObjectInputStream s)
    			throws IOException, ClassNotFoundException {
    s.defaultReadObject();
    // Defensively copy our mutable components
    start = new Date(start.getTime());
    end = new Date(end.getTime());
    // Check that our invariants are satisfied
    if (start.compareTo(end) > 0)
        throw new InvalidObjectException(start +" after "+ end);
}
```

请注意，防御性副本在有效性检查之前执行，并且我们没有使用Date的克隆方法来执行防御性副本。 需要这两个细节来保护Period免受攻击（第50项）。 另请注意，最终字段无法进行防御性复制。 要使用readObject方法，我们必须使start和end字段为非最终字段。 这是不幸的，但它是两个邪恶中较小的一个。 使用新的readObject方法并从开始和结束字段中删除最终修饰符后，MutablePeriod类将呈现无效。 上面的攻击程序现在生成此输出：

> Wed Nov 22 00:23:41 PST 2017 - Wed Nov 22 00:23:41 PST 2017
> Wed Nov 22 00:23:41 PST 2017 - Wed Nov 22 00:23:41 PST 2017

这是一个简单的试金石，用于判断默认的readObject方法是否适用于某个类：您是否愿意添加一个公共构造函数，该构造函数将对象中每个非瞬态字段的值作为参数，并将值存储在字段中而不进行验证 任何？ 如果没有，则必须提供readObject方法，并且必须执行构造函数所需的所有有效性检查和防御性复制。 或者，您可以使用序列化代理模式（项目90）。 强烈建议使用此模式，因为它需要花费大量精力进行安全反序列化。

readObject方法和构造函数之间还有一个相似之处，它们适用于非最终可序列化类。 与构造函数一样，readObject方法不能直接或间接调用可覆盖的方法（第19项）。 如果违反此规则并且重写了相关方法，则重写方法将在子类的状态被反序列化之前运行。 可能会导致程序失败。

总而言之，无论何时编写readObject方法，都要采用您正在编写公共构造函数的思维模式，该构造函数必须生成有效的实例，而不管它给出了什么字节流。 不要假设字节流表示实际的序列化实例。 虽然此项中的示例涉及使用默认序列化表单的类，但所有引发的问题同样适用于具有自定义序列化表单的类。 这里，以摘要形式，是编写readObject方法的指南：

- 对于具有必须保持私有的对象引用字段的类，防御性地复制此类字段中的每个对象。 不可变类的可变组件属于此类。
- 检查任何不变量，如果检查失败，则抛出InvalidObjectException。 检查应遵循任何防御性复制。
- 如果在反序列化后必须验证整个对象图，请使用ObjectInputValidation接口（本书未讨论）。
- 不要直接或间接调用类中的任何可覆盖方法。

