---
layout: post
title: 《Effective Java》学习日志（四）30：偏爱泛型方法
date: 2018-11-13 18:11:04
author: admin
comments: true
categories: [Java]
tags: [Java,Effective Java]
---

类可以是泛型的，那么方法也可以是泛型。

<!-- more -->

---

学习资料主要参考： 《Effective Java Third Edition》，作者：Joshua Bloch

---




* 目录
{:toc}

---

# 集合中的泛型方法

对参数化类型进行操作的静态工具方法通常都是泛型的。 集合中的所有“算法”方法（如binarySearch和sort）都是泛型的。

编写泛型方法类似于编写泛型类型。 考虑这个方法，它返回两个集合的并集：

    // Uses raw types - unacceptable! [Item 26]
    public static Set union(Set s1, Set s2) {
        Set result = new HashSet(s1);
        result.addAll(s2);
        return result;
    }
    
此方法可以编译但有两个警告：

    Union.java:5: warning: [unchecked] unchecked call to
    HashSet(Collection<? extends E>) as a member of raw type HashSet
    Set result = new HashSet(s1);
    ^
    Union.java:6: warning: [unchecked] unchecked call to
    addAll(Collection<? extends E>) as a member of raw type Set
    result.addAll(s2);
     ^
     
要修复这些警告并使方法类型安全，请修改其声明以声明表示三个集合（两个参数和返回值）的元素类型的类型参数，并在整个方法中使用此类型参数。 

声明类型参数的类型参数列表位于方法的修饰符和返回类型之间。 

在这个例子中，类型参数列表是<E>，返回类型是Set<E>。 类型参数的命名约定对于泛型方法和泛型类型是相同的（Item 29和68）：

    // Generic method
    public static <E> Set<E> union(Set<E> s1, Set<E> s2) {
        Set<E> result = new HashSet<>(s1);
        result.addAll(s2);
        return result;
    }
    
至少对于简单的泛型方法来说，就是这样。 
此方法编译时不会生成任何警告，并提供类型安全性和易用性。 这是一个简单的程序来运行该方法。 这个程序不包含强制转换和编译时没有错误或警告：

    // Simple program to exercise generic method
    public static void main(String[] args) {
        Set<String> guys = Set.of("Tom", "Dick", "Harry");
        Set<String> stooges = Set.of("Larry", "Moe", "Curly");
        Set<String> aflCio = union(guys, stooges);
        System.out.println(aflCio);
    }
    
当运行这个程序时，它会打印[Moe, Tom, Harry, Larry, Curly, Dick]（输出中元素的顺序依赖于具体实现。）

union方法的一个限制是所有三个集合（输入参数和返回值）的类型必须完全相同。 
通过使用限定通配符类型（ bounded wildcard types）（Item 31），可以使该方法更加灵活。

# 泛型单例工厂

有时，需要创建一个不可改变但适用于许多不同类型的对象。 
因为泛型是通过擦除来实现的（Item 28），所以可以使用单个对象进行所有必需的类型参数化，但是需要编写一个静态工厂方法来重复地为每个请求的类型参数化分配对象。 
这种称为泛型单例工厂（generic singleton factory）的模式用于方法对象（ function objects）（Item 42），
比如Collections.reverseOrder方法，偶尔也用于Collections.emptySet之类的集合。

假设你想写一个恒等方法分配器（ identity function dispenser）。 
类库提供了Function.identity方法，所以没有理由编写你自己的实现（Item 59），但它是有启发性的。 
如果每次要求的时候都去创建一个新的恒等方法对象是浪费的，因为它是无状态的。 
如果Java的泛型被具体化，那么每个类型都需要一个恒等方法，但是由于它们被擦除以后，所以泛型的单例就足够了。 

以下是它的实例：

    // Generic singleton factory pattern
    private static UnaryOperator<Object> IDENTITY_FN = (t) -> t;
    
    @SuppressWarnings("unchecked")
    public static <T> UnaryOperator<T> identityFunction() {
        return (UnaryOperator<T>) IDENTITY_FN;
    }
    
将IDENTITY_FN转换为(UnaryFunction<T>)会生成一个未经检查的强制转换警告，因为UnaryOperator<Object>对于每个T都不是一个UnaryOperator<T>。
但是恒等方法是特殊的：它返回未修改的参数，所以我们知道，使用它作为一个UnaryFunction <T>是类型安全的，无论T的值是多少。
因此，我们可以放心地抑制由这个强制生成的未经检查的强制转换警告。 一旦我们完成了这些，代码编译没有错误或警告。

## 示例

下面是一个示例程序，它使用我们的泛型单例作为 UnaryOperator<String> 和 UnaryOperator<Number>。 
像往常一样，它不包含强制转化，编译时也没有错误和警告：

    // Sample program to exercise generic singleton
    public static void main(String[] args) {
    
        String[] strings = { "jute", "hemp", "nylon" };
        UnaryOperator<String> sameString = identityFunction();
        for (String s : strings)
            System.out.println(sameString.apply(s));
    
        Number[] numbers = { 1, 2.0, 3L };
        UnaryOperator<Number> sameNumber = identityFunction();
    
        for (Number n : numbers)
            System.out.println(sameNumber.apply(n));
    
    }
    
# 递归类型限制

虽然相对较少，类型参数受涉及该类型参数本身的某种表达式限制是允许的。 
这就是所谓的递归类型限制（recursive type bound）。 递归类型限制的常见用法与Comparable接口有关，它定义了一个类型的自然顺序（Item 14）。 

这个接口如下所示：

    public interface Comparable<T> {
        int compareTo(T o);
    }

类型参数T定义了实现Comparable<T>的类型的元素可以比较的类型。 

在实际中，几乎所有类型都只能与自己类型的元素进行比较。 

所以，例如，String类实现了Comparable<String>，Integer类实现了Comparable <Integer>等等。

许多方法采用实现Comparable的元素的集合来对其进行排序，在其中进行搜索，计算其最小值或最大值等。 
要做到这一点，要求集合中的每一个元素都可以与其中的每一个元素相比，换言之，这个元素是可以相互比较的。 

以下是如何表达这一约束：

    // Using a recursive type bound to express mutual comparability
    public static <E extends Comparable<E>> E max(Collection<E> c);

限定的类型 <E extends Comparable<E>> 可以理解为“任何可以与自己比较的类型E”，这或多或少精确地对应于相互可比性的概念。

这里有一个与前面的声明相匹配的方法。它根据其元素的自然顺序来计算集合中的最大值，并编译没有错误或警告：

    // Returns max value in a collection - uses recursive type bound
    public static <E extends Comparable<E>> E max(Collection<E> c) {
        if (c.isEmpty())
            throw new IllegalArgumentException("Empty collection");
    
        E result = null;
        for (E e : c)
            if (result == null || e.compareTo(result) > 0)
                result = Objects.requireNonNull(e);
        return result;
    }
    
请注意，如果列表为空，则此方法将引发IllegalArgumentException异常。 
更好的选择是返回一个 Optional<E>（Item 55）。

递归类型限制可能变得复杂得多，但幸运的是他们很少这样做。 
如果你理解了这个习惯用法，它的通配符变体（Item 31）和模拟的自我类型用法（Item 2），你将能够处理在实践中遇到的大多数递归类型限制。

# 总结

总之，像泛型类型一样，泛型方法比需要客户端对输入参数和返回值进行显式强制转换的方法更安全，更易于使用。 
像类型一样，你应该确保你的方法可以不用强制转换，这通常意味着它们是泛型的。 

应该泛型化现有的方法，其使用需要强制转换。 这使得新用户的使用更容易，而不会破坏现有的客户端（Item 26）。
