---
layout: post
title: 《Effective Java》学习日志（四）27：消除非检查警告
date: 2018-11-06 18:41:04
author: admin
comments: true
categories: [Java]
tags: [Java,Effective Java]
---

使用泛型编程时，会看到许多编译器警告：未经检查的强制转换警告，未经检查的方法调用警告，未经检查的参数化可变长度类型警告以及未经检查的转换警告。

要尽力消除。

<!-- more -->

---

学习资料主要参考： 《Effective Java Third Edition》，作者：Joshua Bloch

---

## 目录
{:.no_toc}

* 目录
{:toc}

---

# 异常消除的简单例子

许多未经检查的警告很容易消除。 例如，假设你不小心写了以下声明：

    Set<Lark> exaltation = new HashSet();
    
编译器会提醒你你做错了什么：

    Venery.java:4: warning: [unchecked] unchecked conversion
            Set<Lark> exaltation = new HashSet();
                                   ^
      required: Set<Lark>
      found:    HashSet
      
然后可以进行指示修正，让警告消失。 

请注意，实际上并不需要指定类型参数，只是为了表明它与Java 7中引入的钻石运算符（"<>"）一同出现。
然后编译器会推断出正确的实际类型参数（在本例中为Lark）：

    Set<Lark> exaltation = new HashSet<>();
    

# 尽量消除未经检查的警告

但一些警告更难以消除。 本章充满了这种警告的例子。当你收到需要进一步思考的警告时，坚持不懈！ 

**尽可能地消除每一个未经检查的警告。** 
如果你消除所有的警告，你可以放心，你的代码是类型安全的，这是一件非常好的事情。 
这意味着在运行时你将不会得到一个ClassCastException异常，并且增加了你的程序将按照你的意图行事的信心。

**如果你不能消除警告，但你可以证明引发警告的代码是类型安全的，那么（并且只能这样）用@SuppressWarnings(“unchecked”)注解来抑制警告。** 

如果你在没有首先证明代码是类型安全的情况下压制警告，那么你给自己一个错误的安全感。 
代码可能会在不发出任何警告的情况下进行编译，但是它仍然可以在运行时抛出ClassCastException异常。 

但是，如果你忽略了你认为是安全的未经检查的警告（而不是抑制它们），那么当一个新的警告出现时，你将不会注意到这是一个真正的问题。 

新出现的警告就会淹没在所有的错误警告当中。

SuppressWarnings注解可用于任何声明，从单个局部变量声明到整个类。 
**始终在尽可能最小的范围内使用SuppressWarnings注解。** 
通常这是一个变量声明或一个非常短的方法或构造方法。 切勿在整个类上使用SuppressWarnings注解。 这样做可能会掩盖重要的警告。

如果你发现自己在长度超过一行的方法或构造方法上使用SuppressWarnings注解，则可以将其移到局部变量声明上。 
你可能需要声明一个新的局部变量，但这是值得的。 

例如，考虑这个来自ArrayList的toArray方法：

    public <T> T[] toArray(T[] a) {
        if (a.length < size)
           return (T[]) Arrays.copyOf(elements, size, a.getClass());
        System.arraycopy(elements, 0, a, 0, size);
        if (a.length > size)
           a[size] = null;
        return a;
    }

如果编译ArrayList类，则该方法会生成此警告：

    ArrayList.java:305: warning: [unchecked] unchecked cast
           return (T[]) Arrays.copyOf(elements, size, a.getClass());
                                     ^
      required: T[]
      found:    Object[]
      
在返回语句中设置SuppressWarnings注解是非法的，因为它不是一个声明[JLS，9.7]。 
你可能会试图把注释放在整个方法上，但是不要这要做。 
相反，声明一个局部变量来保存返回值并标注它的声明，如下所示：

    // Adding local variable to reduce scope of @SuppressWarnings
    public <T> T[] toArray(T[] a) {
        if (a.length < size) {
            // This cast is correct because the array we're creating
            // is of the same type as the one passed in, which is T[].
            @SuppressWarnings("unchecked") T[] result =
                (T[]) Arrays.copyOf(elements, size, a.getClass());
            return result;
        }
        System.arraycopy(elements, 0, a, 0, size);
        if (a.length > size)
            a[size] = null;
        return a;
    }
    
所产生的方法干净地编译，并最小化未经检查的警告被抑制的范围。

**每当使用@SuppressWarnings(“unchecked”)注解时，请添加注释，说明为什么是安全的。** 
这将有助于他人理解代码，更重要的是，这将减少有人修改代码的可能性，从而使计算不安全。 
如果你觉得很难写这样的注释，请继续思考。 
毕竟，你最终可能会发现未经检查的操作是不安全的。

# 总结

总之，未经检查的警告是重要的。 不要忽视他们。 
每个未经检查的警告代表在运行时出现ClassCastException异常的可能性。 
尽你所能消除这些警告。 
如果无法消除未经检查的警告，并且可以证明引发该警告的代码是安全类型的，则可以在尽可能小的范围内使用 @SuppressWarnings(“unchecked”)注解来禁止警告。 
记录你决定在注释中抑制此警告的理由。