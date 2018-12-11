---
layout: post
title: 《Effective Java》学习日志（七）54：返回集合或者数组而不是null
date: 2018-12-11 18:43:04
author: admin
comments: true
categories: [Java]
tags: [Java,Effective Java]
---



<!-- more -->

---

学习资料主要参考： 《Effective Java Third Edition》，作者：Joshua Bloch

---

## 目录
{:.no_toc}

* 目录
{:toc}
---

类似这样的方法并不罕见：

    // Returns null to indicate an empty collection. Don’t do this!
    private final List<Cheese> cheesesInStock = ...;
    /**
    * @return a list containing all of the cheeses in the shop,
    * or null if no cheeses are available for purchase.
    */
    public List<Cheese> getCheeses() {
        return cheesesInStock.isEmpty() ? null
                : new ArrayList<>(cheesesInStock);
    }
    
在没有奶酪可供购买的情况下，没有理由将它处理为特殊情况。 
这样做需要客户端中的额外代码来处理可能为null的返回值，例如：

    List<Cheese> cheeses = shop.getCheeses();
    if (cheeses != null && cheeses.contains(Cheese.STILTON))
        System.out.println("Jolly good, just the thing.");

几乎每次使用一个返回null的方法代替空集合或数组时，都需要这种验证。
它容易出错，因为编写客户端的程序员可能忘记编写特殊情况代码来处理null返回。
多年来这种错误可能会被忽视，因为这种方法通常会返回一个或多个对象。
此外，返回null代替空容器会使返回容器的方法的实现变得复杂。

有时候认为空返回值比空集合或数组更可取，因为它避免了分配空容器的费用。
这个论点在两个方面失败了。
首先，除非测量结果显示所讨论的分配是性能问题的真正原因，否则不建议担心此级别的性能（第67项）。
其次，可以在不分配空集合和数组的情况下返回它们。

以下是返回可能为空的集合的典型代码。
通常，这就是您所需要的：

    //The right way to return a possibly empty collection
    public List<Cheese> getCheeses() {
        return new ArrayList<>(cheesesInStock);
    }

如果您有证据表明分配空集合会损害性能，则可以通过重复返回相同的不可变空集合来避免分配，因为不可变对象可以自由共享（第17项）。 

下面是使用Collections.emptyList方法执行此操作的代码。 
如果你要返回一个集合，你将使用Collections.emptySet; 如果您要返回Map，则使用Collections.emptyMap。 
但请记住，这是一个优化，而且很少需要它。 
如果您认为需要它，请测量前后的性能，以确保它实际上有所帮助：

    // Optimization - avoids allocating empty collections
    public List<Cheese> getCheeses() {
        return cheesesInStock.isEmpty() ? Collections.emptyList()
            : new ArrayList<>(cheesesInStock);
    }

数组的情况与集合的情况相同。 
永远不要返回null而不是零长度数组。 
通常，您应该只返回一个正确长度的数组，该数组可能为零。 
请注意，我们将一个零长度数组传递给toArray方法以指示所需的返回类型，即Cheese[]：

    //The right way to return a possibly empty array
    public Cheese[] getCheeses() {
        return cheesesInStock.toArray(new Cheese[0]);
    }

如果您认为分配零长度数组会损害性能，则可以重复返回相同的零长度数组，因为所有零长度数组都是不可变的：

    // Optimization - avoids allocating empty arrays
    private static final Cheese[] EMPTY_CHEESE_ARRAY = new Cheese[0];
    
    public Cheese[] getCheeses() {
            return cheesesInStock.toArray(EMPTY_CHEESE_ARRAY);
    }

在优化版本中，我们将相同的空数组传递给每个toArray调用，并且只要cheesesInStock为空，就会从getCheeses返回此数组。 
不要预先传递传递给toArray的数组，以期提高性能。 
研究表明它适得其反：

    // Don’t do this - preallocating the array harms performance!
    return cheesesInStock.toArray(new Cheese[cheesesInStock.size()]);

总之，**永远不会返回null来代替空数组或集合**。 
它使您的API更难以使用并且更容易出错，并且它没有性能优势。
