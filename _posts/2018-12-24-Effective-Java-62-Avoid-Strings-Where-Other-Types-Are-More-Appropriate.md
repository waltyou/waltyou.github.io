---
layout: post
title: 《Effective Java》学习日志（八）62:当其他类型更合适的时候不选择String
date: 2018-12-24 19:11:04
author: admin
comments: true
categories: [Java]
tags: [Java,Effective Java]

---

字符串旨在表示文本，并且它们可以很好地完成它。因为字符串是如此常见并且语言得到很好的支持，所以自然倾向于将字符串用于除设计字符串以外的目的。这个项目讨论了一些你不应该对字符串做的事情。

<!-- more -->

------

学习资料主要参考： 《Effective Java Third Edition》，作者：Joshua Bloch

------

## 目录
{:.no_toc}

* 目录
{:toc}

------

## 问题场景

### 字符串是其他值类型的不良替代品

当一段数据从文件，网络或键盘输入进入程序时，它通常是字符串形式。有一种自然倾向，就是这样，但只有当数据本质上是文本性的时候，这种趋势才是合理的。如果它是数字，则应将其转换为适当的数字类型，例如int，float或BigInteger。如果它是一个是或否的问题的答案，它应该被翻译成适当的枚举类型或布尔值。更一般地说，如果有适当的值类型，无论是原始值还是对象引用，都应该使用它;如果没有，你应该写一个。虽然这个建议似乎很明显，但它经常被违反。

### 字符串是枚举类型的不良替代品

正如第34项中所讨论的，枚举使得枚举类型常量远远超过字符串。

### 字符串是聚合类型的不良替代品

如果实体具有多个组件，则将其表示为单个字符串通常是个坏主意。例如，这里是来自真实系统的一行代码 - 标识符名称已被更改以保护有罪：

```java
// Inappropriate use of string as aggregate type
String compoundKey = className + "#" + i.next();
```

这种方法有许多缺点。 如果用于分隔字段的字符出现在其中一个字段中，则可能会导致混乱。 要访问单个字段，您必须解析字符串，这是缓慢，乏味和容易出错的。 您不能提供equals，toString或compareTo方法，但必须接受String提供的行为。 更好的方法是编写一个类来表示聚合，通常是私有静态成员类（第24项）。

### 字符串是功能的不良替代品

有时，字符串用于授予对某些功能的访问权限。 例如，考虑线程局部变量工具的设计。 这样的工具提供了每个线程都有自己值的变量。 从版本1.2开始，Java库就有了一个线程局部变量设备，但在此之前，程序员必须自己动手。 当多年前遇到设计这样一个设施的任务时，有几个人独立地提出了相同的设计，其中客户提供的字符串键用于识别每个线程局部变量：

```java
// Broken - inappropriate use of string as capability!
public class ThreadLocal {
    private ThreadLocal() { } // Noninstantiable
    // Sets the current thread's value for the named variable.
    public static void set(String key, Object value);
    // Returns the current thread's value for the named variable.
    public static Object get(String key);
}
```

这种方法的问题是字符串键表示线程局部变量的共享全局命名空间。 为了使方法起作用，客户端提供的字符串键必须是唯一的：如果两个客户端独立决定对其线程局部变量使用相同的名称，则它们无意中共享一个变量，这通常会导致两个客户端 失败。 而且，安全性很差。 恶意客户端可能故意使用与另一个客户端相同的字符串密钥来获取对其他客户端数据的非法访问权限。

可以通过使用不可伪造的密钥（有时称为capability）替换字符串来修复此API：

```java
public class ThreadLocal {
    private ThreadLocal() { } // Noninstantiable
    
    public static class Key { // (Capability)
    	Key() { }
    }
    
    // Generates a unique, unforgeable key
    public static Key getKey() {
    	return new Key();
    }
    
    public static void set(Key key, Object value);
    public static Object get(Key key);
}
```

虽然这解决了基于字符串的API的两个问题，但您可以做得更好。 你不再需要静态方法了。 它们可以成为键上的实例方法，此时键不再是线程局部变量的键：它是线程局部变量。 此时，顶级类不再为您做任何事情了，所以您可以摆脱它并将嵌套类重命名为ThreadLocal：

```java
public final class ThreadLocal {
    public ThreadLocal();
    public void set(Object value);
    public Object get();
}
```

此API不是类型安全的，因为当您从线程局部变量检索它时，必须将值从Object转换为其实际类型。 不可能使原始的基于String的API类型安全且难以使基于密钥的API类型安全，但通过使ThreadLocal成为参数化类（第29项）来使这种API类型安全是一件简单的事情：

```java
public final class ThreadLocal<T> {
    public ThreadLocal();
    public void set(T value);
    public T get();
}
```

粗略地说，这是java.lang.ThreadLocal提供的API。 除了解决基于字符串的API的问题之外，它还比任何基于密钥的API更快，更优雅。

## 总结

总而言之，当存在或可以编写更好的数据类型时，避免将对象表示为字符串的自然倾向。 使用不当，字符串比其他类型更麻烦，更灵活，更慢，更容易出错。 字符串通常被滥用的类型包括基本类型，枚举和聚合类型。