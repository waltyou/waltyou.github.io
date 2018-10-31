---
layout: post
title: 《Effective Java》学习日志（三）22：仅使用接口来定义类型
date: 2018-10-30 10:51:04
author: admin
comments: true
categories: [Java]
tags: [Java,Effective Java]
---

当类实现接口时，接口可以作为一个类型来引用该类实例。也就是说，一个实现接口的类，就是向客户端声明可以对该类的实例做出某些操作。
除此之外，为任何其他目的，来定义接口都是不合适的。

<!-- more -->
---

学习资料主要参考： 《Effective Java Third Edition》，作者：Joshua Bloch

---
## 目录
{:.no_toc}

* 目录
{:toc}

---

# 失败的例子： constant interface

一个不满足本节要求的接口是：constant interface（常量接口）。
这样的接口不包含任何方法; 它仅由静态最终字段组成，每个字段都输出一个常量。 
使用这些常量的类实现了接口，以避免使用类名限定常量名称。 

这是一个例子：
```java
// Constant interface antipattern - do not use!
public interface PhysicalConstants {
    // Avogadro's number (1/mol)
    static final double AVOGADROS_NUMBER = 6.022_140_857e23;
    // Boltzmann constant (J/K)
    static final double BOLTZMANN_CONSTANT = 1.380_648_52e-23;
    // Mass of the electron (kg)
    static final double ELECTRON_MASS = 9.109_383_56e-31;
}
```

**常量接口模式是接口的不良使用。**

# 为什么

类在内部使用一些常量，完全属于实现细节。
实现一个常量接口会导致这个实现细节泄漏到类的导出API中。
对类的用户来说，类实现一个常量接口是没有意义的。
事实上，它甚至可能使他们感到困惑。
更糟糕的是，它代表了一个承诺：如果在将来的版本中修改了类，不再需要使用常量，那么它仍然必须实现接口，以确保二进制兼容性。
如果一个非final类实现了常量接口，那么它的所有子类的命名空间都会被接口中的常量所污染。

Java平台类库中有多个常量接口，如java.io.ObjectStreamConstants。 这些接口应该被视为不规范的，不应该被效仿。

# 怎么替代

如果你想导出常量，有几个合理的选择方案。 
如果常量与现有的类或接口紧密相关，则应将其添加到该类或接口中。 
例如，所有数字基本类型的包装类，如Integer和Double，都会导出MIN_VALUE和MAX_VALUE常量。 
如果常量最好被看作枚举类型的成员，则应该使用枚举类型（条目 34）导出它们。 
否则，你应该用一个不可实例化的工具类来导出常量（条目 4）。 
下是前面所示的PhysicalConstants示例的工具类的版本：

```java
// Constant utility class
package com.effectivejava.science;

public class PhysicalConstants {
  private PhysicalConstants() { }  // Prevents instantiation

  public static final double AVOGADROS_NUMBER = 6.022_140_857e23;
  public static final double BOLTZMANN_CONST  = 1.380_648_52e-23;
  public static final double ELECTRON_MASS    = 9.109_383_56e-31;
}
```

顺便提一下，请注意在数字文字中使用下划线字符（_）。 
从Java 7开始，合法的下划线对数字字面量的值没有影响，但是如果使用得当的话可以使它们更容易阅读。 
无论是固定的浮点数，如果他们包含五个或更多的连续数字，考虑将下划线添加到数字字面量中。 
对于底数为10的数字，无论是整型还是浮点型的，都应该用下划线将数字分成三个数字组，表示一千的正负幂。

通常，实用工具类要求客户端使用类名来限定常量名，例如PhysicalConstants.AVOGADROS_NUMBER。 

**如果大量使用实用工具类导出的常量，则通过使用静态导入来限定具有类名的常量**：

```java
// Use of static import to avoid qualifying constants
import static com.effectivejava.science.PhysicalConstants.*;

public class Test {
    double  atoms(double mols) {
        return AVOGADROS_NUMBER * mols;
    }
    ...
    // Many more uses of PhysicalConstants justify static import
}
```

总之，接口只能用于定义类型。 它们不应该仅用于导出常量。

