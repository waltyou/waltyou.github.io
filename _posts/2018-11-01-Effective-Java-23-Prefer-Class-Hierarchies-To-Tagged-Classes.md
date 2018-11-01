---
layout: post
title: 《Effective Java》学习日志（三）23：首选类层次结构来标记类
date: 2018-11-01 09:51:04
author: admin
comments: true
categories: [Java]
tags: [Java,Effective Java]
---

有时，我们可能会遇到一个类，它的实例有两种或更多种类，并包含一个标记字段，指示实例的种类。
但是这种类有一些短板，我们可以用类层次结构来避免。

<!-- more -->
---

学习资料主要参考： 《Effective Java Third Edition》，作者：Joshua Bloch

---
## 目录
{:.no_toc}

* 目录
{:toc}

---

# 举个例子

```java
// Tagged class - vastly inferior to a class hierarchy!
class Figure {
    
    enum Shape { RECTANGLE, CIRCLE };
    
    // Tag field - the shape of this figure
    final Shape shape;
    
    // These fields are used only if shape is RECTANGLE
    double length;
    double width;
    
    // This field is used only if shape is CIRCLE
    double radius;
    
    // Constructor for circle
    Figure(double radius) {
        shape = Shape.CIRCLE;
        this.radius = radius;
    }
    
    // Constructor for rectangle
    Figure(double length, double width) {
        shape = Shape.RECTANGLE;
        this.length = length;
        this.width = width;
    }
    
    double area() {
        switch(shape) {
            case RECTANGLE:
                return length * width;
            case CIRCLE:
                return Math.PI * (radius * radius);
            default:
                throw new AssertionError(shape);
        }
    }
}
```

这种标记的类有许多缺点。 

- 它们杂乱无章，包括枚举声明，标记字段和switch语句。 可读性进一步受到损害，因为多个实现在一个类中混杂在一起。 
- 内存占用量增加，因为实例负担了属于其他风格的不相关字段。 除非构造函数初始化不相关的字段，否则不能将字段设为final，从而产生更多的样板。 
- 构造函数必须设置标记字段并在没有编译器帮助的情况下初始化正确的数据字段：如果初始化错误的字段，程序将在运行时失败。 
- 除非可以修改其源文件，否则不要为标记类添加标记。如果添加了一个flavor，则必须记住在每个switch语句中添加一个case，否则该类将在运行时失败。 
- 实例的数据类型没有给出其标记的线索。 

**简而言之，标记类是冗长的，容易出错且效率低下的。**

# 类层次结构

幸运的是，Java等面向对象语言为定义能够表示多种类型对象的单一数据类型提供了更好的选择：子类型。 

**标记类只是对类层次结构的苍白模仿。**

那该怎么修改上面的例子呢？

## 第一步：定义抽象类

要将标记类转换为类层次结构，首先要为标记类中的每个方法定义一个抽象类，该方法的行为取决于标记值。

在Figure类中，只有一个这样的方法，即area。

此抽象类是类层次结构的根。如果有任何方法的行为不依赖于标记的值，请将它们放在此类中。
同样，如果所有flavor都使用了任何数据字段，请将它们放在此类中。

图类中没有这种与 flavor 无关的方法或字段。

## 第二步：定义子类

接下来，为原始标记类的每个flavor定义根类的具体子类。

在我们的示例中，有两个：圆和矩形。在每个子类中包含特定于其风格的数据字段。
在我们的示例中，半径特定于圆，长度和宽度特定于矩形。还在每个子类中包含根类中每个抽象方法的适当实现。

## 结果

这是与原始Figure类对应的类层次结构：

```java
// Class hierarchy replacement for a tagged class
abstract class Figure {
    abstract double area();
}

class Circle extends Figure {
    final double radius;
    Circle(double radius) { this.radius = radius; }
    @Override double area() { return Math.PI * (radius * radius); }
}

class Rectangle extends Figure {
    final double length;
    final double width;
    Rectangle(double length, double width) {
        this.length = length;
        this.width = width;
    }
    @Override double area() { return length * width; }
}
```

这个类层次纠正了之前提到的标签类的每个缺点。 代码简单明了，不包含原文中的样板文件。 

每种类型的实现都是由自己的类来分配的，而这些类都没有被无关的数据属性所占用。 
所有的属性是final的。 

编译器确保每个类的构造方法初始化其数据属性，并且每个类都有一个针对在根类中声明的每个抽象方法的实现。
这消除了由于缺少switch-case语句而导致的运行时失败的可能性。 
 
多个程序员可以独立地继承类层次，并且可以相互操作，而无需访问根类的源代码。 

每种类型都有一个独立的数据类型与之相关联，允许程序员指出变量的类型，并将变量和输入参数限制为特定的类型。

类层次的另一个优点是可以使它们反映类型之间的自然层次关系，从而提高了灵活性，并提高了编译时类型检查的效率。 
假设原始示例中的标签类也允许使用正方形。 
类层次可以用来反映一个正方形是一种特殊的矩形（假设它们是不可变的）：

```java
class Square extends Rectangle {
    Square(double side) {
        super(side, side);
    }
}
```
请注意，上述层中的属性是直接访问的，而不是访问方法。 
这里是为了简洁起见，如果类层次是公开的（条目16），这将是一个糟糕的设计。

总之，标签类很少有适用的情况。 
如果你想写一个带有明显标签属性的类，请考虑标签属性是否可以被删除，而类是否被类层次替换。 
当遇到一个带有标签属性的现有类时，可以考虑将其重构为一个类层次中。




