---
layout: post
title: 《Effective Java》学习日志（四）28：优先使用Lists而不是Arrays
date: 2018-11-07 19:01:04
author: admin
comments: true
categories: [Java]
tags: [Java,Effective Java]
---


<!-- more -->

---

学习资料主要参考： 《Effective Java Third Edition》，作者：Joshua Bloch

---




* 目录
{:toc}

---

# Arrays 与 List 的不同之处

## 1. Arrays 是协变的

协变的意思就是，如果一个 Sub 类是 Super 类的子类，那么 Sub[] 也是 Super[] 的子类。

泛型就是不协变的，对于两个不同的类型 Type1、Type2， List<Type1> 即不是 List<Type2> 的子类也不是它的父类。

乍看起来，泛型有所缺陷，但其实有缺陷的是 Arrays。

下面代码是合法的：
    
    // Fails at runtime!
    Object[] objectArray = new Long[1];
    objectArray[0] = "I don't fit in"; // Throws ArrayStoreException

但是这个就是对的：

    // Won't compile!
    List<Object> ol = new ArrayList<Long>(); // Incompatible types
    ol.add("I don't fit in");

虽然上面两段代码都不可能把一个 string 放入一个 Long 的容器中，但是区别在于，Arrays 不会在编译期报错，你只能在运行期间才能发现错误。

而泛型就不一样，它根本无法通过编译期。


## 2. Arrays 是具体化的

这意味着数组在运行时知道并强制执行它们的元素类型。 
如前所述，如果尝试将一个String放入Long数组中，得到一个ArrayStoreException异常。 

相反，泛型通过擦除（erasure）来实现[JLS，4.6]。 这意味着它们只在编译时执行类型约束，并在运行时丢弃（或擦除）它们的元素类型信息。 
擦除是允许泛型类型与不使用泛型的遗留代码自由互操作（条目 26），从而确保在Java 5中平滑过渡到泛型。

由于这些基本差异，数组和泛型不能很好地在一起混合使用。 

例如，创建泛型类型的数组，参数化类型的数组，以及类型参数的数组都是非法的。 
因此，这些数组创建表达式都不合法：new List <E> []，new List <String> []，new E []。 
这些都会在编译时导致泛型数组创建错误。

为什么创建一个泛型数组是非法的？ 因为它不是类型安全的。 
如果这是合法的，编译器生成的强制转换程序在运行时可能会因为ClassCastException异常而失败。 
这将违反泛型类型系统提供的基本保证。

为了具体说明，请考虑下面的代码片段：

    // Why generic array creation is illegal - won't compile!
    List<String>[] stringLists = new List<String>[1];  // (1)
    List<Integer> intList = List.of(42);               // (2)
    Object[] objects = stringLists;                    // (3)
    objects[0] = intList;                              // (4)
    String s = stringLists[0].get(0);                  // (5)
    
让我们假设第1行创建一个泛型数组是合法的。
第2行创建并初始化包含单个元素的List<Integer>。
第3行将List<String>数组存储到Object数组变量中，这是合法的，因为数组是协变的。
第4行将List<Integer>存储在Object数组的唯一元素中，这是因为泛型是通过擦除来实现的：List<Integer>实例的运行时类型仅仅是List，而List<String> []实例是List []，所以这个赋值不会产生ArrayStoreException异常。

现在我们遇到了麻烦。将一个List<Integer>实例存储到一个声明为仅保存List<String>实例的数组中。
在第5行中，我们从这个数组的唯一列表中检索唯一的元素。编译器自动将检索到的元素转换为String，但它是一个Integer，所以我们在运行时得到一个ClassCastException异常。
为了防止发生这种情况，第1行（创建一个泛型数组）必须产生一个编译时错误。

# 不可具体化的类型

类型E，List<E>和List<String>等在技术上被称为不可具体化的类型（nonreifiable types）[JLS，4.7]。 
直观地说，不可具体化的类型是其运行时表示包含的信息少于其编译时表示的类型。 
由于擦除，可唯一确定的参数化类型是无限定通配符类型，如List <?>和Map <?, ?>（条目 26）。 
尽管很少有用，创建无限定通配符类型的数组是合法的。

禁止泛型数组的创建可能会很恼人的。 
这意味着，例如，泛型集合通常不可能返回其元素类型的数组（但是参见条目 33中的部分解决方案）。 
这也意味着，当使用可变参数方法（条目 53）和泛型时，会产生令人困惑的警告。 
这是因为每次调用可变参数方法时，都会创建一个数组来保存可变参数。 如果此数组的元素类型不可确定，则会收到警告。 
SafeVarargs注解可以用来解决这个问题（条目 32）。

当你在强制转换为数组类型时，得到泛型数组创建错误，或是未经检查的强制转换警告时，最佳解决方案通常是使用集合类型List <E>而不是数组类型E []。 
这样可能会牺牲一些简洁性或性能，但作为交换，你会获得更好的类型安全性和互操作性。

例如，假设你想用带有集合的构造方法来编写一个Chooser类，并且有个方法返回随机选择的集合的一个元素。 
根据传递给构造方法的集合，可以使用选择器作为游戏模具，魔术8球或数据源进行蒙特卡罗模拟。 这是一个没有泛型的简单实现：

    // Chooser - a class badly in need of generics!
    public class Chooser {
        private final Object[] choiceArray;
    
        public Chooser(Collection choices) {
            choiceArray = choices.toArray();
        }
    
        public Object choose() {
            Random rnd = ThreadLocalRandom.current();
            return choiceArray[rnd.nextInt(choiceArray.length)];
        }
    }
    
要使用这个类，每次调用方法时，都必须将Object的choose方法的返回值转换为所需的类型，如果类型错误，则转换在运行时失败。 

我们先根据条目 29的建议，试图修改Chooser类，使其成为泛型的。

    // A first cut at making Chooser generic - won't compile
    public class Chooser<T> {
        private final T[] choiceArray;
    
        public Chooser(Collection<T> choices) {
            choiceArray = choices.toArray();
        }
    
        // choose method unchanged
    }
    
如果你尝试编译这个类，会得到这个错误信息：

    Chooser.java:9: error: incompatible types: Object[] cannot be
    converted to T[]
            choiceArray = choices.toArray();
                                         ^
      where T is a type-variable:
        T extends Object declared in class Chooser
        
没什么大不了的，将Object数组转换为T数组：

    choiceArray = (T[]) choices.toArray();
    
这没有了错误，而是得到一个警告：

    Chooser.java:9: warning: [unchecked] unchecked cast
            choiceArray = (T[]) choices.toArray();
                                               ^
      required: T[], found: Object[]
      where T is a type-variable:
    T extends Object declared in class Chooser
    
编译器告诉你在运行时不能保证强制转换的安全性，因为程序不会知道T代表什么类型——记住，元素类型信息在运行时会被泛型删除。 
该程序可以正常工作吗？ 是的，但编译器不能证明这一点。 
你可以证明这一点，在注释中提出证据，并用注解来抑制警告，但最好是消除警告的原因（条目 27）。

要消除未经检查的强制转换警告，请使用列表而不是数组。 下面是另一个版本的Chooser类，编译时没有错误或警告：

    // List-based Chooser - typesafe
    public class Chooser<T> {
        private final List<T> choiceList;
    
        public Chooser(Collection<T> choices) {
            choiceList = new ArrayList<>(choices);
        }
    
        public T choose() {
            Random rnd = ThreadLocalRandom.current();
            return choiceList.get(rnd.nextInt(choiceList.size()));
        }
    }
    
这个版本有些冗长，也许运行比较慢，但是值得一提的是，在运行时不会得到ClassCastException异常。

总之，数组和泛型具有非常不同的类型规则。 
数组是协变和具体化的; 泛型是不变的，类型擦除的。 因此，数组提供运行时类型的安全性，但不提供编译时类型的安全性，反之亦然。 
一般来说，数组和泛型不能很好地混合工作。 如果你发现把它们混合在一起，得到编译时错误或者警告，你的第一个冲动应该是用列表来替换数组。

