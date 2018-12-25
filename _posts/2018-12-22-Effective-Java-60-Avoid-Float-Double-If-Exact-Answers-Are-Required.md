---
layout: post
title: 《Effective Java》学习日志（八）60:避免使用float和double，除非真的需要
date: 2018-12-22 22:15:04
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

float和double类型主要用于科学和工程计算。 

它们执行二进制浮点运算，经过精心设计，可在很宽的范围内快速提供准确的近似值。 但是，它们不能提供准确的结果，不应在需要确切结果的地方使用。 **浮动和双重类型特别不适合货币计算**，因为不可能将0.1（或任何其他10的负幂）表示为浮点数或双精度。

例如，假设你的口袋里有1.03美元，你花费42美分。 你还剩多少钱？ 

这是一个试图回答这个问题的程序片段：

```java
System.out.println(1.03 - 0.42);
```

不幸的是，它打印出0.6100000000000001。 

这不是一个孤立的案例。 假设你的口袋里有一美元，你买九个垫圈，每个垫圈的价格为10美分。 你得到多少变化？

```java
System.out.println(1.00 - 9 * 0.10);
```

根据这个程序片段，你得到$ 0.09999999999999998。

您可能认为问题只能通过在打印前舍入结果来解决，但不幸的是，这并不总是有效。 

例如，假设你的口袋里有一块钱，而且你看到一个架子上有一排美味的糖果，价格分别为10美分，20美分，30美分等等，最高可达1美元。 你买一个糖果，从一个10美分的糖果开始，直到你买不起货架上的下一个糖果。 你买了多少个糖果，你有多少变化？ 这是一个旨在解决这个问题的天真程序：

```java
// Broken - uses floating point for monetary calculation!
public static void main(String[] args) {
    double funds = 1.00;
    int itemsBought = 0;
    for (double price = 0.10; funds >= price; price += 0.10) {
        funds -= price;
        itemsBought++;
    }
    System.out.println(itemsBought + " items bought.");
    System.out.println("Change: $" + funds);
}
```

如果你运行该程序，你会发现你可以买三块糖果，剩下$ 0.3999999999999999。 这是错误的答案！ **解决此问题的正确方法是使用BigDecimal，int或long进行货币计算**。

这是对前一个程序的直接转换，使用BigDecimal类型代替double。 请注意，使用BigDecimal的String构造函数而不是其双构造函数。 这是必要的，以避免在计算中引入不准确的值[Bloch05，Puzzle 2]：

```java
public static void main(String[] args) {
    final BigDecimal TEN_CENTS = new BigDecimal(".10");
    
    int itemsBought = 0;
    BigDecimal funds = new BigDecimal("1.00");
    for (BigDecimal price = TEN_CENTS;
            funds.compareTo(price) >= 0;
            price = price.add(TEN_CENTS)) {
        funds = funds.subtract(price);
        itemsBought++;
    }
    System.out.println(itemsBought + " items bought.");
    System.out.println("Money left over: $" + funds);
}
```

如果你运行修改后的程序，你会发现你可以买到四块糖果，剩下0.00美元。 这是正确的答案。

但是，使用BigDecimal有两个缺点：它比使用原始算术类型方便得多，而且速度要慢得多。 如果你解决一个短暂的问题，后一个缺点是无关紧要的，但前者可能会惹恼你。

使用BigDecimal的另一种方法是使用int或long，具体取决于所涉及的数量，并自己跟踪小数点。 在这个例子中，显而易见的方法是以美分而不是美元进行所有计算。 这是采用这种方法的直接转换：

```java
public static void main(String[] args) {
	int itemsBought = 0;
    int funds = 100;
    for (int price = 10; funds >= price; price += 10) {
        funds -= price;
        itemsBought++;
    }
    System.out.println(itemsBought + " items bought.");
    System.out.println("Cash left over: " + funds + " cents");
}
```

总之，不要对任何需要精确答案的计算使用float或double。 

如果您希望系统跟踪小数点，请使用BigDecimal，并且不介意不使用基本类型的不便和成本。 使用BigDecimal具有额外的优势，它可以让您完全控制舍入，只要执行需要舍入的操作，就可以从八种舍入模式中进行选择。 如果您使用法律规定的舍入行为执行业务计算，这会派上用场。 如果性能至关重要，您不介意自己跟踪小数点，并且数量不是太大，请使用int或long。 如果数量不超过九位十进制数，则可以使用int; 如果他们不超过十八位数，你可以使用长。 如果数量可能超过十八位，请使用BigDecimal。