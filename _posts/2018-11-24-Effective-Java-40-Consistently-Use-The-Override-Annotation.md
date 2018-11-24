---
layout: post
title: 《Effective Java》学习日志（五）40：总是使用Override注解
date: 2018-11-24 11:50:04
author: admin
comments: true
categories: [Java]
tags: [Java,Effective Java]
---

Java类库包含几个注解类型。
对于典型的程序员来说，最重要的是@Override。

<!-- more -->

---

学习资料主要参考： 《Effective Java Third Edition》，作者：Joshua Bloch

---

## 目录
{:.no_toc}

* 目录
{:toc}

---

此注解只能在方法声明上使用，它表明带此注解的方法声明重写了父类的声明。
如果始终使用这个注解，它将避免产生大量的恶意bug。

考虑这个程序，在这个程序中，类Bigram表示双字母组合，或者是有序的一对字母：

    // Can you spot the bug?
    public class Bigram {
        private final char first;
        private final char second;
    
        public Bigram(char first, char second) {
            this.first  = first;
            this.second = second;
        }
    
        public boolean equals(Bigram b) {
            return b.first == first && b.second == second;
        }
    
        public int hashCode() {
            return 31 * first + second;
        }
    
        public static void main(String[] args) {
            Set<Bigram> s = new HashSet<>();
            for (int i = 0; i < 10; i++)
                for (char ch = 'a'; ch <= 'z'; ch++)
                    s.add(new Bigram(ch, ch));
            System.out.println(s.size());
        }
    }
    
主程序重复添加二十六个双字母组合到集合中，每个双字母组合由两个相同的小写字母组成。 
然后它会打印集合的大小。 
你可能希望程序打印26，因为集合不能包含重复项。 
如果你尝试运行程序，你会发现它打印的不是26，而是260。

它有什么问题？

显然，Bigram类的作者打算重写equals方法（Item 10），甚至记得重写hashCode（Item 11）。 
不幸的是，我们倒霉的程序员没有重写equals，而是重载它（Item 52）。 
要重写Object.equals，必须定义一个equals方法，其参数的类型为Object，但Bigram的equals方法的参数不是Object类型的，因此Bigram继承Object的equals方法，这个equals方法测试对象的引用是否是同一个，就像==运算符一样。 
每个祖母组合的10个副本中的每一个都与其他9个副本不同，所以它们被Object.equals视为不相等，这就解释了程序打印260的原因。

幸运的是，编译器可以帮助你找到这个错误，但只有当你通过告诉它你打算重写Object.equals来帮助你。 
要做到这一点，用@Override注解Bigram.equals方法，如下所示：

    @Override 
    public boolean equals(Bigram b) {
        return b.first == first && b.second == second;
    }
    
如果插入此注解并尝试重新编译该程序，编译器将生成如下错误消息：

    Bigram.java:10: method does not override or implement a method
    from a supertype
    @Override public boolean equals(Bigram b) {
    ^
    
你会立刻意识到你做错了什么，在额头上狠狠地打了一下，用一个正确的（Item 10）来替换出错的equals实现：

    @Override public boolean equals(Object o) {
        if (!(o instanceof Bigram))
            return false;
        Bigram b = (Bigram) o;
        return b.first == first && b.second == second;
    }
    
因此，应该在你认为要重写父类声明的每个方法声明上使用Override注解。 

这条规则有一个小例外。 
如果正在编写一个没有标记为抽象的类，并且确信它重写了其父类中的抽象方法，则无需将Override注解放在该方法上。 
在没有声明为抽象的类中，如果无法重写抽象父类方法，编译器将发出错误消息。 
但是，你可能希望关注类中所有重写父类方法的方法，在这种情况下，也应该随时注解这些方法。 
大多数IDE可以设置为在选择重写方法时自动插入Override注解。

大多数IDE提供了是种使用Override注解的另一个理由。 
如果启用适当的检查功能，如果有一个方法没有Override注解但是重写父类方法，则IDE将生成一个警告。 
如果始终使用Override注解，这些警告将提醒你无意识的重写。 
它们补充了编译器的错误消息，这些消息会提醒你无意识重写失败。 
IDE和编译器，可以确保你在任何你想要的地方和其他地方重写方法，万无一失。

Override注解可用于重写来自接口和类的方法声明。 
随着default默认方法的出现，在接口方法的具体实现上使用Override以确保签名是正确的是一个好习惯。 
如果知道某个接口没有默认方法，可以选择忽略接口方法的具体实现上的Override注解以减少混乱。

然而，在一个抽象类或接口中，值得标记的是你认为重写父类或父接口方法的所有方法，无论是具体的还是抽象的。 
例如，Set接口不会向Collection接口添加新方法，因此它应该在其所有方法声明中包含Override注解以确保它不会意外地向Collection接口添加任何新方法。

总之，如果在每个方法声明中使用Override注解，并且认为要重写父类声明，那么编译器可以保护免受很多错误的影响，但有一个例外。 
在具体的类中，不需要注解标记你确信可以重写抽象方法声明的方法（尽管这样做也没有坏处）。

