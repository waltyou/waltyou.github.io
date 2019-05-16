---
layout: post
title: 《Effective Java》学习日志（三）25：将源文件限制为单个顶级类
date: 2018-11-04 15:41:04
author: admin
comments: true
categories: [Java]
tags: [Java,Effective Java]
---

虽然Java编译器允许在单个源文件中定义多个顶级类，但这样做没有任何好处，并且存在重大风险。 

<!-- more -->

---

学习资料主要参考： 《Effective Java Third Edition》，作者：Joshua Bloch

---




* 目录
{:toc}

---

风险源于在源文件中定义多个顶级类使得为类提供多个定义成为可能。 

使用哪个定义会受到源文件传递给编译器的顺序的影响。

# 例子 

为了具体说明，请考虑下面源文件，其中只包含一个引用其他两个顶级类（Utensil和Dessert类）的成员的Main类：

```java
public class Main {

    public static void main(String[] args) {
        System.out.println(Utensil.NAME + Dessert.NAME);
    }

}
```

现在假设在Utensil.java的源文件中同时定义了Utensil和Dessert：

```java
// Two classes defined in one file. Don't ever do this!
class Utensil {
    static final String NAME = "pan";
}

class Dessert {
    static final String NAME = "cake";

}
```

当然，main方法会打印pancake。

现在假设你不小心创建了另一个名为Dessert.java的源文件，它定义了相同的两个类：

```java
// Two classes defined in one file. Don't ever do this!
class Utensil {
    static final String NAME = "pot";
}

class Dessert {
    static final String NAME = "pie";
}
```

如果你足够幸运，使用命令javac Main.java Dessert.java编译程序，编译将失败，编译器会告诉你，你已经多次定义了类Utensil和Dessert。 

这是因为编译器首先编译Main.java，当它看到对Utensil的引用（它在Dessert的引用之前）时，它将在Utensil.java中查找这个类并找到Utensil和Dessert。 

当编译器在命令行上遇到Dessert.java时，它也将拉入该文件，导致它遇到Utensil和Dessert的定义。

如果使用命令javac Main.java或javac Main.java Utensil.java编译程序，它的行为与在编写Dessert.java文件（即打印pancake）之前的行为相同。 
但是，如果使用命令javac Dessert.java Main.java编译程序，它将打印potpie。 

程序的行为因此受到源文件传递给编译器的顺序的影响，这显然是不可接受的。

# 静态成员类

解决这个问题很简单，将顶层类（如我们的例子中的Utensil和Dessert）分割成单独的源文件。 

如果试图将多个顶级类放入单个源文件中，请考虑使用静态成员类（条目 24）作为将类拆分为单独的源文件的替代方法。 

如果这些类从属于另一个类，那么将它们变成静态成员类通常是更好的选择，因为它提高了可读性，并且可以通过声明它们为私有（条目 15）来减少类的可访问性。

下面是我们的例子看起来如何使用静态成员类：

```java
// Static member classes instead of multiple top-level classes
public class Test {

    public static void main(String[] args) {
        System.out.println(Utensil.NAME + [Dessert.NAME](http://Dessert.NAME));
    }

    private static class Utensil {
        static final String NAME = "pan";
    }

    private static class Dessert {

        static final String NAME = "cake";

    }

}
```

这个教训很清楚：**永远不要将多个顶级类或接口放在一个源文件中**。 

遵循这个规则保证在编译时不能有多个定义。 这又保证了编译生成的类文件以及生成的程序的行为与源文件传递给编译器的顺序无关。










