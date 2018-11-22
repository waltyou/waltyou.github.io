---
layout: post
title: 《Effective Java》学习日志（五）38：使用接口模拟可扩展的枚举
date: 2018-11-22 14:10:04
author: admin
comments: true
categories: [Java]
tags: [Java,Effective Java]
---


<!-- more -->

---

学习资料主要参考： 《Effective Java Third Edition》，作者：Joshua Bloch

---

## 目录
{:.no_toc}

* 目录
{:toc}

---

在几乎所有方面，枚举类型都优于本书第一版中描述的类型安全模式[Bloch01]。 

从表面上看，一个例外涉及可扩展性，这在原始模式下是可能的，但不受语言结构支持。 
换句话说，使用该模式，有可能使一个枚举类型扩展为另一个; 使用语言功能特性，它不能这样做。 
这不是偶然的。 大多数情况下，枚举的可扩展性是一个糟糕的主意。 
令人困惑的是，扩展类型的元素是基类型的实例，反之亦然。 
枚举基本类型及其扩展的所有元素没有好的方法。 
最后，可扩展性会使设计和实现的很多方面复杂化。 

也就是说，对于可扩展枚举类型至少有一个有说服力的用例，这就是操作码（ operation codes），也称为opcodes。 
操作码是枚举类型，其元素表示某些机器上的操作，例如Item 34中的`Operation`类型，它表示简单计算器上的功能。 
有时需要让API的用户提供他们自己的操作，从而有效地扩展API提供的操作集。 

幸运的是，使用枚举类型有一个很好的方法来实现这种效果。
基本思想是利用枚举类型可以通过为opcode类型定义一个接口，并实现任意接口。

例如，这里是来自Item 34的`Operation`类型的可扩展版本： 

```java
// Emulated extensible enum using an interface 
public interface Operation { 
    double apply(double x, double y); 
} 

public enum BasicOperation implements Operation { 
    PLUS("+") { 
        public double apply(double x, double y) { return x + y; } 
    }, 
    MINUS("-") { 
        public double apply(double x, double y) { return x - y; } 
    }, 
    TIMES("*") { 
        public double apply(double x, double y) { return x * y; } 
    }, 
    DIVIDE("/") { 
        public double apply(double x, double y) { return x / y; } 
    }; 
    
    private final String symbol; 
    
    BasicOperation(String symbol) { 
        this.symbol = symbol; 
    } 
    
    @Override public String toString() { 
        return symbol; 
    } 
} 
``` 

虽然枚举类型（`BasicOperation`）不可扩展，但接口类型（`Operation`）是可以扩展的，并且它是用于表示API中的操作的接口类型。
你可以定义另一个实现此接口的枚举类型，并使用此新类型的实例来代替基本类型。 

例如，假设想要定义前面所示的操作类型的扩展，包括指数运算和余数运算。 
你所要做的就是编写一个实现`Operation`接口的枚举类型： 
```java
// Emulated extension enum
public enum ExtendedOperation implements Operation {
    EXP("^") {
    public double apply(double x, double y) {
    return Math.pow(x, y);
    }}
    ,
    REMAINDER("%") {
    public double apply(double x, double y) {
    return x % y;
    }}
    ;
    
    private final String symbol;
    ExtendedOperation(String symbol) {
        this.symbol = symbol;
    }
    @Override public String toString() {
        return symbol;
    }
}
```

只要API编写为接口类型（`Operation`），而不是实现（`BasicOperation`），现在就可以在任何可以使用基本操作的地方使用新操作。
请注意，不必在枚举中声明`apply`抽象方法，就像您在具有实例特定方法实现的非扩展枚举中所做的那样（第162页）。 
这是因为抽象方法（`apply`）是接口（`Operation`）的成员。 

不仅可以在任何需要“基本枚举”的地方传递“扩展枚举”的单个实例，而且还可以传入整个扩展枚举类型，并使用其元素。 
例如，这里是第163页上的一个测试程序版本，它执行之前定义的所有扩展操作： 
```java
public static void main(String[] args) {
    double x = Double.parseDouble(args[0]);
    double y = Double.parseDouble(args[1]);
    test(ExtendedOperation.class, x, y);
}

private static <T extends Enum<T> & Operation> void test(
    Class<T> opEnumType, double x, double y) {
    for (Operation op : opEnumType.getEnumConstants())
        System.out.printf("%f %s %f = %f%n", x, op, y, op.apply(x, y));
}
```

注意，扩展的操作类型的类字面文字（`ExtendedOperation.class`）从`main`方法里传递给了`test`方法，用来描述扩展操作的集合。
这个类的字面文字用作限定的类型令牌（Item 33）。
`opEnumType`参数中复杂的声明（` & Operation> Class`）确保了Class对象既是枚举又是`Operation`的子类，这正是遍历元素和执行每个元素相关联的操作时所需要的。 

第二种方式是传递一个` Collection`，这是一个限定通配符类型（Item 31），而不是传递了一个class对象： 
```java
public static void main(String[] args) {
    double x = Double.parseDouble(args[0]);
    double y = Double.parseDouble(args[1]);
    test(Arrays.asList(ExtendedOperation.values()), x, y);
}
private static void test(Collection<? extends Operation> opSet,
    double x, double y) {
    for (Operation op : opSet)
        System.out.printf("%f %s %f = %f%n", x, op, y, op.apply(x, y));
}
```

生成的代码稍微不那么复杂，`test`方法灵活一点：它允许调用者将多个实现类型的操作组合在一起。
另一方面，也放弃了在指定操作上使用`EnumSet`(Item 36)和`EnumMap`(Item 37)的能力。 

上面的两个程序在运行命令行输入参数4和2时生成以下输出： 

    4.000000 ^ 2.000000 = 16.000000
    4.000000 % 2.000000 = 0.000000
    

 2.000000 = 16.000000 4.000000 % 2.000000 = 0.000000 ``` 
 
使用接口来模拟可扩展枚举的一个小缺点是，实现不能从一个枚举类型继承到另一个枚举类型。
如果实现代码不依赖于任何状态，则可以使用默认实现(Item 20)将其放置在接口中。
在我们的`Operation`示例中，存储和检索与操作关联的符号的逻辑必须在`BasicOperation`和`ExtendedOperation`中重复。
在这种情况下，这并不重要，因为很少的代码是冗余的。
如果有更多的共享功能，可以将其封装在辅助类或静态辅助方法中，以消除代码冗余。 

该Item中描述的模式在Java类库中有所使用。
例如，`java.nio.file.LinkOption`枚举类型实现了`CopyOption`和`OpenOption`接口。 

总之，**虽然不能编写可扩展的枚举类型，但是你可以编写一个接口来配合实现接口的基本的枚举类型，来对它进行模拟**。
这允许客户端编写自己的枚举（或其它类型）来实现接口。
如果API是根据接口编写的，那么在任何使用基本枚举类型实例的地方，都可以使用这些枚举类型实例。
