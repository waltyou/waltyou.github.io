---
layout: post
title: 《Effective Java》学习日志（八）58:偏爱使用 for-each 循环来替代传统循环
date: 2018-12-20 18:55:04
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

如第45项所述，某些任务最好用流完成，其他任务最好用迭代完成。 

这是一个迭代集合的传统for循环：

```java
// Not the best way to iterate over a collection!
for (Iterator<Element> i = c.iterator(); i.hasNext(); ) {
	Element e = i.next();
	... // Do something with e
}
```

这是一个传统的for循环迭代数组：

```java
// Not the best way to iterate over an array!
for (int i = 0; i < a.length; i++) {
	... // Do something with a[i]
}
```

这些习语比while循环更好（第57项），但它们并不完美。 迭代器和索引变量都是混乱 - 你需要的只是元素。此外，它们代表了犯错的机会。 迭代器在每个循环中出现三次，索引变量为四，这使您有很多机会使用错误的变量。 如果这样做，则无法保证编译器能够解决问题。 最后，两个循环是完全不同的，不必要地注意容器的类型，并添加（轻微）麻烦来改变这种类型。

for-each循环（官方称为“增强语句”）解决了所有这些问题。 它通过隐藏迭代器或索引变量来消除混乱和错误的机会。 由此产生的习语同样适用于集合和数组，简化了将容器的实现类型从一个切换到另一个的过程：

```java
// The preferred idiom for iterating over collections and arrays
for (Element e : elements) {
	... // Do something with e
}
```

当您看到冒号(:)时，将其读作“in”。因此，上面的循环读作“对于元素中的每个元素e”。即使对于数组，使用for-each循环也没有性能损失：它们生成的内容与您手动编写的代码基本相同。

与嵌套迭代相比，for-each循环优于传统for循环的优势更大。

这是人们在进行嵌套迭代时常犯的错误：

```java
// Can you spot the bug?
enum Suit { CLUB, DIAMOND, HEART, SPADE }
enum Rank { ACE, DEUCE, THREE, FOUR, FIVE, SIX, SEVEN, EIGHT,
			NINE, TEN, JACK, QUEEN, KING }
...
static Collection<Suit> suits = Arrays.asList(Suit.values());
static Collection<Rank> ranks = Arrays.asList(Rank.values());

List<Card> deck = new ArrayList<>();
for (Iterator<Suit> i = suits.iterator(); i.hasNext(); )
    for (Iterator<Rank> j = ranks.iterator(); j.hasNext(); )
    	deck.add(new Card(i.next(), j.next()));
```

如果你没有发现这个错误，请不要心疼。 许多专家程序员曾经一次犯过这个错误。 问题是在外部集合（诉讼）的迭代器上调用下一个方法的次数太多。 它应该从外部循环调用，以便每个套装调用一次，而是从内部循环调用它，因此每个卡调用一次。 在用完套装后，循环抛出NoSuchElementException。
如果你真的不走运，外部集合的大小是内部集合大小的倍数 - 也许是因为它们是相同的集合 - 循环将正常终止，但它不会做你想要的。 例如，考虑这种错误的尝试打印一对骰子的所有可能的卷：

```java
// Same bug, different symptom!
enum Face { ONE, TWO, THREE, FOUR, FIVE, SIX }
...
Collection<Face> faces = EnumSet.allOf(Face.class);
for (Iterator<Face> i = faces.iterator(); i.hasNext(); )
    for (Iterator<Face> j = faces.iterator(); j.hasNext(); )
    	System.out.println(i.next() + " " + j.next());
```

该程序不会抛出异常，但它只打印六个“双打”（从“ONE ONE”到“SIX SIX”），而不是预期的三十六个组合。

要修复这些示例中的错误，必须在外部循环的范围中添加一个变量来保存外部元素：

```java
// Fixed, but ugly - you can do better!
for (Iterator<Suit> i = suits.iterator(); i.hasNext(); ) {
    Suit suit = i.next();
    for (Iterator<Rank> j = ranks.iterator(); j.hasNext(); )
    	deck.add(new Card(suit, j.next()));
}
```

相反，如果您使用嵌套的for-each循环，问题就会消失。 生成的代码简洁如您所愿：

```java
// Preferred idiom for nested iteration on collections and arrays
for (Suit suit : suits)
    for (Rank rank : ranks)
    	deck.add(new Card(suit, rank));
```

不幸的是，有三种常见情况你不能使用for-each：

- **破坏性过滤** - 如果需要遍历删除所选元素的集合，则需要使用显式迭代器，以便可以调用其remove方法。 您通常可以使用在Java 8中添加的Collection的removeIf方法来避免显式遍历。
- **转换** - 如果需要遍历列表或数组并替换其元素的部分或全部值，则需要列表迭代器或数组索引才能替换元素的值。
- **并行迭代** - 如果需要并行遍历多个集合，则需要对迭代器或索引变量进行显式控制，以便所有操作符或索引变量可以锁步前进（如有缺陷的卡和骰子示例中无意中所示） 以上）。

如果您发现自己处于上述任何一种情况，请使用普通的for循环并警惕此项中提到的陷阱。

for-each循环不仅可以迭代集合和数组，还可以迭代实现Iterable接口的任何对象，该接口由单个方法组成。 以下是界面的外观：

```java
public interface Iterable<E> {
    // Returns an iterator over the elements in this iterable
    Iterator<E> iterator();
}
```

如果你必须从头开始编写自己的Iterator实现，那么实现Iterable有点棘手，但如果你正在编写一个代表一组元素的类型，你应该强烈考虑让它实现Iterable，即使你选择不让它实现Collection。 这将允许您的用户使用for-each循环迭代您的类型，他们将永远感激。

总之，for-each循环在清晰度，灵活性和错误预防方面提供了超越传统for循环的引人注目的优势，而且没有性能损失。 尽可能使用for-each循环优先于for循环。