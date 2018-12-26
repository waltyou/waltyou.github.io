---
layout: post
title: 《Effective Java》学习日志（八）65:在反射中偏爱接口
date: 2018-12-26 18:52:04
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

核心反射工具java.lang.reflect提供对任意类的编程访问。 给定一个Class对象，您可以获得Constructor，Method和Field实例，这些实例表示由Class实例表示的类的构造函数，方法和字段。 这些对象提供对类的成员名称，字段类型，方法签名等的编程访问。

此外，Constructor，Method和Field实例允许您反思性地操作它们的底层对应物：您可以通过在Constructor，Method和Field实例上调用方法来构造实例，调用方法和访问底层类的字段。 例如，Method.invoke允许您在任何类的任何对象上调用任何方法（受通常的安全性约束）。 反射允许一个类使用另一个类，即使在编译前者时后一个类不存在。 然而，这种力量是有代价的：

- **您将失去编译时类型检查的所有好处，包括异常检查**。 如果程序试图反射性地调用不存在或不可访问的方法，则除非您采取了特殊的预防措施，否则它将在运行时失败。
- **执行反射访问所需的代码是笨拙和冗长的**。 写作和阅读困难是单调乏味的。
- **性能受损**。 反射方法调用比普通方法调用慢得多。 究竟要慢多少，因为有许多因素在起作用。 在我的机器上，当反射完成时，调用没有输入参数和int返回的方法会慢11倍。

有一些复杂的应用程序需要反射。示例包括代码分析工具和依赖注入框架。即使这些工具已经远离最近的反思，因为它的缺点变得更加清晰。如果您对应用程序是否需要反射有任何疑问，则可能不会。

**通过仅以非常有限的形式使用反射，您可以获得许多反射的好处，同时产生很少的成本**。对于许多必须使用在编译时不可用的类的程序，在编译时存在一个适当的接口或超类来引用该类（Item 64）。如果是这种情况，您可以**反射创建实例并通过其接口或超类正常访问它们**。

例如，这是一个创建Set <String>实例的程序，其实例的类由第一个命令行参数指定。程序将剩余的命令行参数插入到集合中并打印它。无论第一个参数如何，程序都会打印剩余的参数，并删除重复项。但是，打印这些参数的顺序取决于第一个参数中指定的类。如果指定java.util.HashSet，则它们以明显随机的顺序打印;如果指定java.util.TreeSet，则它们按字母顺序打印，因为TreeSet中的元素是按顺序排序的：

```java
// Reflective instantiation with interface access
public static void main(String[] args) {
    // Translate the class name into a Class object
    Class<? extends Set<String>> cl = null;
    try {
        cl = (Class<? extends Set<String>>) // Unchecked cast!
        Class.forName(args[0]);
    } catch (ClassNotFoundException e) {
    	fatalError("Class not found.");
    }
    // Get the constructor
    Constructor<? extends Set<String>> cons = null;
    try {
    	cons = cl.getDeclaredConstructor();
    } catch (NoSuchMethodException e) {
    	fatalError("No parameterless constructor");
    }
    // Instantiate the set
    Set<String> s = null;
    try {
    	s = cons.newInstance();
    } catch (IllegalAccessException e) {
    	fatalError("Constructor not accessible");
    } catch (InstantiationException e) {
    	fatalError("Class not instantiable.");
    } catch (InvocationTargetException e) {
    	fatalError("Constructor threw " + e.getCause());
    } catch (ClassCastException e) {
    	fatalError("Class doesn't implement Set");
    }
    // Exercise the set
    s.addAll(Arrays.asList(args).subList(1, args.length));
    System.out.println(s);
}

private static void fatalError(String msg) {
    System.err.println(msg);
    System.exit(1);
}
```

虽然这个程序只是一个玩具，但它演示的技术非常强大。 玩具程序可以很容易地变成一个通用集测试器，通过积极地操纵一个或多个实例并检查它们是否遵守Set契约来验证指定的Set实现。 同样，它可以变成通用的集合性能分析工具。 事实上，这种技术足以实现一个成熟的服务提供者框架（第1项）。 通常，这种技术就是你在反思中所需要的。

这个例子说明了反射的两个缺点。 首先，该示例可以在运行时生成六个不同的异常，如果不使用反射实例化，则所有这些异常都是编译时错误。 （为了好玩，您可以通过传入适当的命令行参数使程序生成六个异常中的每一个。）第二个缺点是需要二十五行繁琐的代码才能从其名称生成类的实例， 而构造函数调用可以整齐地放在一行上。 可以通过捕获ReflectiveOperationException来减少程序的长度，ReflectiveOperationException是Java 7中引入的各种反射异常的超类。这两个缺点仅限于实例化对象的程序部分。 实例化后，该集合与任何其他Set实例无法区分。 在实际程序中，大量代码因此不受这种有限的反射使用的影响。

如果您编译此程序，您将获得未经检查的强制转换警告。 这个警告是合法的，因为强制转换为Class <？ 即使命名类不是Set实现，扩展Set <String>也会成功，在这种情况下，程序在实例化类时抛出ClassCastException。 要了解有关抑制警告的信息，请阅读第27项。

合法（如果罕见）使用反射是管理类对运行时可能不存在的其他类，方法或字段的依赖性。 如果您正在编写必须针对某些其他软件包的多个版本运行的软件包，这将非常有用。 该技术是针对支持它所需的最小环境（通常是最旧的版本）编译您的软件包，并反复访问任何更新的类或方法。 要使其工作，如果在运行时不存在您尝试访问的较新类或方法，则必须采取适当的操作。 适当的行动可能包括使用一些替代手段来实现相同的目标或以减少的功能运行。

总之，反射是某些复杂系统编程任务所需的强大工具，但它有许多缺点。 如果您正在编写一个必须在编译时使用未知类的程序，那么您应该尽可能使用反射来实例化对象，并使用编译时已知的某个接口或超类来访问这些对象。

