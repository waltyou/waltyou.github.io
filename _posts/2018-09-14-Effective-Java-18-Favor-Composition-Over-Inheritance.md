---
layout: post
title: 《Effective Java》学习日志（三）18：组合优于继承
date: 2018-09-14 09:41:04
author: admin
comments: true
categories: [Java]
tags: [Java,Effective Java]
---

继承是实现代码重用的有效方式，但并不总是最好的工具。。。。

<!-- more -->

---

学习资料主要参考： 《Effective Java Third Edition》，作者：Joshua Bloch

---




* 目录
{:toc}

---

# 继承的优缺点

继承是实现代码重用的有效方式，但并不总是最好的工具。
使用不当，会导致脆弱的软件。 

在包中使用继承是安全的，其中子类和父类的实现都在同一个程序员的控制之下。
对应专门为了继承而设计的，并且有文档说明的类来说（条目 19），使用继承也是安全的。 
然而，从普通的具体类跨越包级边界继承，是危险的。 

提醒一下，本书使用“继承”一词来表示实现继承（当一个类继承另一个类时）。 
在这个项目中讨论的问题不适用于接口继承（当类实现接口或当接口继承另一个接口时）。

与方法调用不同，继承打破了封装。 
换句话说，一个子类依赖于其父类的实现细节来保证其正确的功能。 
父类的实现可能会从发布版本不断变化，如果是这样，子类可能会被破坏，即使它的代码没有任何改变。 
因此，一个子类必须与其超类一起更新而变化，除非父类的作者为了继承的目的而专门设计它，并对应有文档的说明。

# 举个例子

为了具体说明，假设有一个使用HashSet的程序。 
为了调整程序的性能，需要查询HashSet，从创建它之后已经添加了多少个元素（不要和当前的元素数量混淆，当元素被删除时数量也会下降）。 
为了提供这个功能，编写了一个HashSet变体，它保留了尝试元素插入的数量，并导出了这个插入数量的一个访问方法。 
HashSet类包含两个添加元素的方法，分别是add和addAll，所以我们重写这两个方法：

```java
// Broken - Inappropriate use of inheritance!
public class InstrumentedHashSet<E> extends HashSet<E> {
    // The number of attempted element insertions
    private int addCount = 0;

    public InstrumentedHashSet() {
    }

    public InstrumentedHashSet(int initCap, float loadFactor) {
        super(initCap, loadFactor);
    }
    @Override public boolean add(E e) {
        addCount++;
        return super.add(e);
    }
    @Override public boolean addAll(Collection<? extends E> c) {
        addCount += c.size();
        return super.addAll(c);
    }
    public int getAddCount() {
        return addCount;
    }
}

```

这个类看起来很合理，但是不能正常工作。 
假设创建一个实例并使用addAll方法添加三个元素。 
顺便提一句，请注意，下面代码使用在Java 9中添加的静态工厂方法List.of来创建一个列表；
如果使用的是早期版本，请改为使用Arrays.asList：

```java
InstrumentedHashSet<String> s = new InstrumentedHashSet<>();
s.addAll(List.of("Snap", "Crackle", "Pop"));
```

我们期望getAddCount方法返回的结果是3，但实际上返回了6。
哪里出来问题？在HashSet内部，addAll方法是基于它的add方法来实现的，即使HashSet文档中没有指名其实现细节，倒也是合理的。
InstrumentedHashSet中的addAll方法首先给addCount属性设置为3，然后使用super.addAll方法调用了HashSet的addAll实现。
然后反过来又调用在InstrumentedHashSet类中重写的add方法，每个元素调用一次。
这三次调用又分别给addCount加1，所以，一共增加了6：通过addAll方法每个增加的元素都被计算了两次。

我们可以通过消除addAll方法的重写来“修复”子类。 
尽管生成的类可以正常工作，但是它依赖于它的正确方法，因为HashSet的addAll方法是在其add方法之上实现的。 
这个“自我使用（self-use）”是一个实现细节，并不保证在Java平台的所有实现中都可以适用，并且可以随发布版本而变化。 
因此，产生的InstrumentedHashSet类是脆弱的。

稍微好一点的做法是，重写addAll方法遍历指定集合，为每个元素调用add方法一次。 
不管HashSet的addAll方法是否在其add方法上实现，都会保证正确的结果，因为HashSet的addAll实现将不再被调用。
然而，这种技术并不能解决所有的问题。 

这相当于重新实现了父类方法，这样的方法可能不能确定到底是否时自用（self-use）的，实现起来也是困难的，耗时的，容易出错的，并且可能会降低性能。
此外，这种方式并不能总是奏效，因为子类无法访问一些私有属性，所以有些方法就无法实现。

导致子类脆弱的一个相关原因是，它们的父类在后续的发布版本中可以添加新的方法。
假设一个程序的安全性依赖于这样一个事实：所有被插入到集中的元素都满足一个先决条件。
可以通过对集合进行子类化，然后并重写所有添加元素的方法，以确保在添加每个元素之前满足这个先决条件，来确保这一问题。
如果在后续的版本中，父类没有新增添加元素的方法，那么这样做没有问题。
但是，一旦父类增加了这样的新方法，则很有肯能由于调用了未被重写的新方法，将非法的元素添加到子类的实例中。
这不是个纯粹的理论问题。
在把Hashtable和Vector类加入到Collections框架中的时候，就修复了几个类似性质的安全漏洞。

这两个问题都源于重写方法。 

如果仅仅添加新的方法并且不要重写现有的方法，可能会认为继承一个类是安全的。 
虽然这种扩展更为安全，但这并非没有风险。 
如果父类在后续版本中添加了一个新的方法，并且你不幸给了子类一个具有相同签名和不同返回类型的方法，那么你的子类编译失败[JLS，8.4.8.3]。 
如果已经为子类提供了一个与新的父类方法具有相同签名和返回类型的方法，那么你现在正在重写它，因此将遇到前面所述的问题。 

此外，你的方法是否会履行新的父类方法的约定，这是值得怀疑的，因为在你编写子类方法时，这个约定还没有写出来。

# 组合

幸运的是，有一种方法可以避免上述所有的问题。
不要继承一个现有的类，而应该给你的新类增加一个私有属性，该属性是 现有类的实例引用，这种设计被称为组合（composition），因为现有的类成为新类的组成部分。
新类中的每个实例方法调用现有类的包含实例上的相应方法并返回结果。
这被称为**转发（forwarding）**，而新类中的方法被称为转发方法。
由此产生的类将坚如磐石，不依赖于现有类的实现细节。
即使将新的方法添加到现有的类中，也不会对新类产生影响。
为了具体说用，下面代码使用组合和转发方法替代InstrumentedHashSet类。
请注意，实现分为两部分，类本身和一个可重用的转发类，其中包含所有的转发方法，没有别的方法：

```java
// Reusable forwarding class
import java.util.Collection;
import java.util.Iterator;
import java.util.Set;

public class ForwardingSet<E> implements Set<E> {

    private final Set<E> s;

    public ForwardingSet(Set<E> s) {
        this.s = s;
    }

    public void clear() {
        s.clear();
    }

    public boolean contains(Object o) {
        return s.contains(o);
    }

    public boolean isEmpty() {
        return s.isEmpty();
    }

    public int size() {
        return s.size();
    }

    public Iterator<E> iterator() {
        return s.iterator();
    }

    public boolean add(E e) {
        return s.add(e);
    }

    public boolean remove(Object o) {
        return s.remove(o);
    }

    public boolean containsAll(Collection<?> c) {
        return s.containsAll(c);
    }

    public boolean addAll(Collection<? extends E> c) {
        return s.addAll(c);
    }

    public boolean removeAll(Collection<?> c) {
        return s.removeAll(c);
    }

    public boolean retainAll(Collection<?> c) {
        return s.retainAll(c);
    }

    public Object[] toArray() {
        return s.toArray();
    }

    public <T> T[] toArray(T[] a) {
        return s.toArray(a);
    }

    @Override
    public boolean equals(Object o) {
        return s.equals(o);
    }

    @Override
    public int hashCode() {
        return s.hashCode();
    }

    @Override
    public String toString() {
        return s.toString();
    }
}
// Wrapper class - uses composition in place of inheritance
import java.util.Collection;
import java.util.Set;

public class InstrumentedSet<E> extends ForwardingSet<E> {

    private int addCount = 0;

    public InstrumentedSet(Set<E> s) {
        super(s);
    }
    
    @Override public boolean add(E e) {
        addCount++;
        return super.add(e);
    }

    @Override public boolean addAll(Collection<? extends E> c) {
        addCount += c.size();
        return super.addAll(c);
    }

    public int getAddCount() {
        return addCount;
    }
}
```

InstrumentedSet类的设计是通过存在的Set接口来实现的，该接口包含HashSet类的功能特性。
除了功能强大，这个设计是非常灵活的。
InstrumentedSet类实现了Set接口，并有一个构造方法，其参数也是Set类型的。
本质上，这个类把Set转换为另一个类型Set， 同时添加了计数的功能。
与基于继承的方法不同，该方法仅适用于单个具体类，并且父类中每个需要支持构造方法，提供单独的构造方法，所以可以使用包装类来包装任何Set实现，并且可以与任何预先存在的构造方法结合使用：

```java
Set<Instant> times = new InstrumentedSet<>(new TreeSet<>(cmp));
Set<E> s = new InstrumentedSet<>(new HashSet<>(INIT_CAPACITY));
```

InstrumentedSet类甚至可以用于临时替换没有计数功能下使用的集合实例：

```java
static void walk(Set<Dog> dogs) {
    InstrumentedSet<Dog> iDogs = new InstrumentedSet<>(dogs);
    ... // Within this method use iDogs instead of dogs
}
```

InstrumentedSet类被称为包装类，因为每个InstrumentedSet实例都包含（“包装”）另一个Set实例。 
这也被称为装饰器模式[Gamma95]，因为InstrumentedSet类通过添加计数功能来“装饰”一个集合。 
有时组合和转发的结合被不精确地地称为委托（delegation）。 
从技术上讲，除非包装对象把自身传递给被包装对象，否则不是委托[Lieberman86;Gamma95]。

包装类的缺点很少。 
一个警告是包装类不适合在回调框架（callback frameworks）中使用，其中对象将自我引用传递给其他对象以用于后续调用（“回调”）。 
因为一个被包装的对象不知道它外面的包装对象，所以它传递一个指向自身的引用（this），回调时并不记得外面的包装对象。 
这被称为SELF问题[Lieberman86]。 
有些人担心转发方法调用的性能影响，以及包装对象对内存占用。 
两者在实践中都没有太大的影响。 
编写转发方法有些繁琐，但是只需为每个接口编写一次可重用的转发类，并且提供转发类。 
例如，Guava为所有的Collection接口提供转发类[Guava]。

只有在子类真的是父类的子类型的情况下，继承才是合适的。 
换句话说，只有在两个类之间存在“is-a”关系的情况下，B类才能继承A类。 
如果你试图让B类继承A类时，问自己这个问题：每个B都是A吗？ 如果你不能如实回答这个问题，那么B就不应该继承A。
如果答案是否定的，那么B通常包含一个A的私有实例，并且暴露一个不同的API：A不是B的重要部分 ，只是其实现细节。

在Java平台类库中有一些明显的违反这个原则的情况。 
例如，stacks实例并不是vector实例，所以Stack类不应该继承Vector类。 
同样，一个属性列表不是一个哈希表，所以Properties不应该继承Hashtable类。 
在这两种情况下，组合方式更可取。

如果在合适组合的地方使用继承，则会不必要地公开实现细节。
由此产生的API将与原始实现联系在一起，永远限制类的性能。
更严重的是，通过暴露其内部，客户端可以直接访问它们。
至少，它可能导致混淆语义。

例如，属性p指向Properties实例，那么 p.getProperty(key)和p.get(key)就有可能返回不同的结果：
前者考虑了默认的属性表，而后者是继承Hashtable的，它则没有考虑默认属性列表。
最严重的是，客户端可以通过直接修改超父类来破坏子类的不变性。
在Properties类，设计者希望只有字符串被允许作为键和值，但直接访问底层的Hashtable允许违反这个不变性。
一旦违反，就不能再使用属性API的其他部分（load和store方法）。
在发现这个问题的时候，纠正这个问题为时已晚，因为客户端依赖于使用非字符串键和值了。

# 总结

在决定使用继承来代替组合之前，你应该问自己最后一组问题。
对于试图继承的类，它的API有没有缺陷呢？ 
如果有，你是否愿意将这些缺陷传播到你的类的API中？
继承传播父类的API中的任何缺陷，而组合可以让你设计一个隐藏这些缺陷的新API。

总之，继承是强大的，但它是有问题的，因为它违反封装。 
只有在子类和父类之间存在真正的子类型关系时才适用。 
即使如此，如果子类与父类不在同一个包中，并且父类不是为继承而设计的，继承可能会导致脆弱性。 
为了避免这种脆弱性，使用合成和转发代替继承，特别是如果存在一个合适的接口来实现包装类。 
包装类不仅比子类更健壮，而且更强大。
