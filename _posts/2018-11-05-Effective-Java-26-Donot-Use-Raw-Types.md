---
layout: post
title: 《Effective Java》学习日志（四）26：不使用原始类型
date: 2018-11-05 18:41:04
author: admin
comments: true
categories: [Java]
tags: [Java,Effective Java]
---

接下来，进入了新的一大章节：泛型。 它会对进入集合的元素进行类型检查，更好的帮助了代码运行时的安全性。
但是获取这些好处是需要付出一定代价的。

<!-- more -->

---

学习资料主要参考： 《Effective Java Third Edition》，作者：Joshua Bloch

---

## 目录
{:.no_toc}

* 目录
{:toc}

---

# 泛型类或接口

声明具有一个或多个类型参数的类或接口是泛型类或接口[JLS，8.1.2,9.1.2]。 

例如，List接口有一个类型参数E，表示其元素类型。 

接口的全名是List <E>（读作“E列表”），但人们通常称之为List。 

通用类和接口统称为泛型类型。

# 形式

每个泛型定义了一组参数化类型（parameterized types），它们由类或接口名称组成，后跟一个与泛型类型的形式类型参数相对应的实际类型参数的尖括号“<>”列表。 

例如，List<String>（读作“字符串列表”）是一个参数化类型，表示其元素类型为String的列表。 （String是与形式类型参数E相对应的实际类型参数）。

最后，每个泛型定义了一个原始类型（ raw type），它是没有任何类型参数的泛型类型的名称。

例如，对应于List<E>的原始类型是List。 原始类型的行为就像所有的泛型类型信息都从类型声明中被清除一样。 它们的存在主要是为了与没有泛型之前的代码相兼容。

# 集合声明

在泛型被添加到Java之前，这是一个典型的集合声明。 从Java 9开始，它仍然是合法的，但并不是典型的声明方式了：

```java
// Raw collection type - don't do this!
// My stamp collection. Contains only Stamp instances.
private final Collection stamps = ... ;
```

如果你今天使用这个声明，然后不小心把coin实例放入你的stamp集合中，错误的插入编译和运行没有错误（尽管编译器发出一个模糊的警告）：

```java
// Erroneous insertion of coin into stamp collection
stamps.add(new Coin( ... )); // Emits "unchecked call" warning
```

直到您尝试从stamp集合中检索coin实例时才会发生错误：

```java
// Raw iterator type - don't do this!
for (Iterator i = stamps.iterator(); i.hasNext(); )
    Stamp stamp = (Stamp) i.next(); // Throws ClassCastException
        stamp.cancel();
```

正如本书所提到的，在编译完成之后尽快发现错误是值得的，理想情况是在编译时。 

在这种情况下，直到运行时才发现错误，在错误发生后的很长一段时间，以及可能远离包含错误的代码的代码中。 

一旦看到ClassCastException，就必须搜索代码类库，查找将coin实例放入stamp集合的方法调用。 

编译器不能帮助你，因为它不能理解那个说“仅包含stamp实例”的注释。

对于泛型，类型声明包含的信息，而不是注释：

```java
// Parameterized collection type - typesafe
private final Collection<Stamp> stamps = ... ;
```

从这个声明中，编译器知道stamps集合应该只包含Stamp实例，并保证它是true，假设你的整个代码类库编译时不发出（或者抑制;参见条目27）任何警告。 
当使用参数化类型声明声明stamps时，错误的插入会生成一个编译时错误消息，告诉你到底发生了什么错误：

```java
Test.java:9: error: incompatible types: Coin cannot be converted
to Stamp
    c.add(new Coin());
```

当从集合中检索元素时，编译器会为你插入不可见的强制转换，并保证它们不会失败（再假设你的所有代码都不会生成或禁止任何编译器警告）。 

虽然意外地将coin实例插入stamp集合的预期可能看起来很牵强，但这个问题是真实的。 
例如，很容易想象将BigInteger放入一个只包含BigDecimal实例的集合中。

# 不要使用原始类型

如前所述，使用原始类型（没有类型参数的泛型）是合法的，但是你不应该这样做。 

**如果你使用原始类型，则会丧失泛型的所有安全性和表达上的优势。** 

鉴于你不应该使用它们，为什么语言设计者首先允许原始类型呢？ 答案是为了兼容性。 

泛型被添加时，Java即将进入第二个十年，并且有大量的代码没有使用泛型。 
所有这些代码都是合法的，并且与使用泛型的新代码进行交互操作被认为是至关重要的。 
将参数化类型的实例传递给为原始类型设计的方法必须是合法的，反之亦然。 
这个需求，被称为**迁移兼容性**，驱使决策支持原始类型，并使用擦除来实现泛型（条目 28）。

虽然不应使用诸如List之类的原始类型，但可以使用参数化类型来允许插入任意对象（如List<Object>）。 

原始类型List和参数化类型List<Object>之间有什么区别？ 

松散地说，前者已经选择了泛型类型系统，而后者明确地告诉编译器，它能够保存任何类型的对象。 
虽然可以将List<String>传递给List类型的参数，但不能将其传递给List<Object>类型的参数。 

泛型有子类型的规则，List<String>是原始类型List的子类型，但不是参数化类型List<Object>的子类型（条目 28）。 

因此，如果使用诸如List之类的原始类型，则会丢失类型安全性，但是如果使用参数化类型（例如List <Object>）则不会。

为了具体说明，请考虑以下程序：

```java
// Fails at runtime - unsafeAdd method uses a raw type (List)!
public static void main(String[] args) {
    List<String> strings = new ArrayList<>();
    unsafeAdd(strings, Integer.valueOf(42));
    String s = strings.get(0); // Has compiler-generated cast
}

private static void unsafeAdd(List list, Object o) {
    list.add(o);
}
```

此程序可以编译，它使用原始类型列表，但会收到警告：

```java
Test.java:10: warning: [unchecked] unchecked call to add(E) as a
member of the raw type List
    list.add(o);
```

实际上，如果运行该程序，则当程序尝试调用strings.get(0)的结果（一个Integer）转换为一个String时，会得到ClassCastException异常。 
这是一个编译器生成的强制转换，因此通常会保证成功，但在这种情况下，我们忽略了编译器警告并付出了代价。

如果用unsafeAdd声明中的参数化类型List <Object>替换原始类型List，并尝试重新编译该程序，则会发现它不再编译，而是发出错误消息：

```java
Test.java:5: error: incompatible types: List<String> cannot be
converted to List<Object>
    unsafeAdd(strings, Integer.valueOf(42));
```

你可能会试图使用原始类型来处理元素类型未知且无关紧要的集合。 例如，假设你想编写一个方法，它需要两个集合并返回它们共同拥有的元素的数量。 如果是泛型新手，那么您可以这样写：

```java
// Use of raw type for unknown element type - don't do this!
static int numElementsInCommon(Set s1, Set s2) {
    int result = 0;
    for (Object o1 : s1)
        if (s2.contains(o1))
            result++;
    return result;
}
```

这种方法可以工作，但它使用原始类型，这是危险的。 

安全替代方式是使用**无限制通配符类型**（unbounded wildcard types）。 
如果要使用泛型类型，但不知道或关心实际类型参数是什么，则可以使用问号来代替。 
例如，泛型类型Set<E>的无限制通配符类型是Set <?>（读取“某种类型的集合”）。 
它是最通用的参数化的Set类型，能够保持任何集合。 

下面是numElementsInCommon方法使用无限制通配符类型声明的情况：

```java
// Uses unbounded wildcard type - typesafe and flexible
static int numElementsInCommon(Set<?> s1, Set<?> s2) { ... }
```

无限制通配符Set <?>与原始类型Set之间有什么区别？ 问号真的给你放任何东西吗？ 

这不是要点，但通配符类型是安全的，原始类型不是。 
你可以将任何元素放入具有原始类型的集合中，轻易破坏集合的类型不变性（如第119页上的unsafeAdd方法所示）; **你不能把任何元素（除null之外）放入一个Collection <?>中**。 

试图这样做会产生一个像这样的编译时错误消息：

```java
WildCard.java:13: error: incompatible types: String cannot be
converted to CAP#1
    c.add("verboten");
          ^
  where CAP#1 is a fresh type-variable:
    CAP#1 extends Object from capture of ?
```

不可否认的是，这个错误信息留下了一些需要的东西，但是编译器已经完成了它的工作，不管它的元素类型是什么，都不会破坏集合的类型不变性。 
你不仅可以将任何元素（除null以外）放入一个Collection <?>中，但是不能保证你所得到的对象的类型。 

如果这些限制是不可接受的，可以使用泛型方法（条目 30）或有限制配符类型（条目 31）。

对于不应该使用原始类型的规则，有一些小例外。 
**你必须在类字面值（class literals）中使用原始类型**。 
规范中不允许使用参数化类型（尽管它允许数组类型和基本类型）[JLS，15.8.2]。 

换句话说，List.class , String[].class 和 int.class 都是合法的，但 List<String>.class 和 List<?>.class不是合法的。

规则的第二个例外涉及instanceof操作符。 

因为泛型类型信息在运行时被删除，所以在无限制通配符类型以外的参数化类型上使用instanceof运算符是非法的。 
使用无限制通配符类型代替原始类型不会以任何方式影响instanceof运算符的行为。 
在这种情况下，尖括号和问号就显得多余。 

以下是使用泛型类型的instanceof运算符的首选方法：

```java
// Legitimate use of raw type - instanceof operator
if (o instanceof Set) {       // Raw type
    Set<?> s = (Set<?>) o;    // Wildcard type
    ...
}
```

请注意，一旦确定o对象是一个Set，则必须将其转换为通配符Set <?>，而不是原始类型Set。 
这是一个强制转换，所以不会导致编译器警告。

总之，使用原始类型可能导致运行时异常，所以不要使用它们。 
它们仅用于与泛型引入之前的传统代码的兼容性和互操作性。 

作为一个快速回顾，Set<Object>是一个参数化类型，表示一个可以包含任何类型对象的集合，Set<?>是一个通配符类型，表示一个只能包含某些未知类型对象的集合，Set是一个原始类型，它不在泛型类型系统之列。 
前两个类型是安全的，最后一个不是。

