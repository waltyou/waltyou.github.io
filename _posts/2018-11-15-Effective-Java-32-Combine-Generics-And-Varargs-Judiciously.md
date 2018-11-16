---
layout: post
title: 《Effective Java》学习日志（四）32：合理地结合泛型和可变参数
date: 2018-11-15 17:50:04
author: admin
comments: true
categories: [Java]
tags: [Java,Effective Java]
---

在Java 5中，可变参数方法（条目 53）和泛型都被添加到平台中，所以你可能希望它们能够正常交互; 可悲的是，他们并没有。 

<!-- more -->

---

学习资料主要参考： 《Effective Java Third Edition》，作者：Joshua Bloch

---

## 目录
{:.no_toc}

* 目录
{:toc}

---

# 可变参数与泛型失败的结合

可变参数的目的是允许客户端将一个可变数量的参数传递给一个方法，但这是一个脆弱的抽象（ leaky abstraction）：
当你调用一个可变参数方法时，会创建一个数组来保存可变参数；那个应该是实现细节的数组是可见的。 

因此，当可变参数具有泛型或参数化类型时，会导致编译器警告混淆。

回顾条目 28，非具体化（non-reifiable）的类型是其运行时表示比其编译时表示具有更少信息的类型，并且几乎所有泛型和参数化类型都是不可具体化的。 

如果某个方法声明其可变参数为非具体化的类型，则编译器将在该声明上生成警告。 
如果在推断类型不可确定的可变参数参数上调用该方法，那么编译器也会在调用中生成警告。 

警告看起来像这样：

    warning: [unchecked] Possible heap pollution from
        parameterized vararg type List<String>
    
当参数化类型的变量引用不属于该类型的对象时会发生堆污染（Heap pollution）[JLS，4.12.2]。 
它会导致编译器的自动生成的强制转换失败，违反了泛型类型系统的基本保证。

例如，请考虑以下方法，该方法是第127页上的代码片段的一个不太明显的变体：

    // Mixing generics and varargs can violate type safety!
    static void dangerous(List<String>... stringLists) {
        List<Integer> intList = List.of(42);
        Object[] objects = stringLists;
        objects[0] = intList;             // Heap pollution
        String s = stringLists[0].get(0); // ClassCastException
    }
    
此方法没有可见的强制转换，但在调用一个或多个参数时抛出ClassCastException异常。 
它的最后一行有一个由编译器生成的隐形转换。 
这种转换失败，表明类型安全性已经被破坏，并且将值保存在泛型可变参数数组参数中是不安全的。

这个例子引发了一个有趣的问题：为什么声明一个带有泛型可变参数的方法是合法的，当明确创建一个泛型数组是非法的时候呢？ 
换句话说，为什么前面显示的方法只生成一个警告，而127页上的代码片段会生成一个错误？ 

答案是，具有泛型或参数化类型的可变参数参数的方法在实践中可能非常有用，因此语言设计人员选择忍受这种不一致。 

事实上，Java类库导出了几个这样的方法，包括Arrays.asList(T... a)，Collections.addAll(Collection<? super T> c, T... elements)，EnumSet.of(E first, E... rest)。 
与前面显示的危险方法不同，这些类库方法是类型安全的。

# SafeVararg

在Java 7中，SafeVarargs注解已添加到平台，以允许具有泛型可变参数的方法的作者自动禁止客户端警告。 
实质上，SafeVarargs注解构成了作者对类型安全的方法的承诺。 为了交换这个承诺，编译器同意不要警告用户调用可能不安全的方法。

除非它实际上是安全的，否则注意不要使用@SafeVarargs注解标注一个方法。 
那么需要做些什么来确保这一点呢？ 

回想一下，调用方法时会创建一个泛型数组，以容纳可变参数。 
如果方法没有在数组中存储任何东西（它会覆盖参数）并且不允许对数组的引用进行转义（这会使不受信任的代码访问数组），那么它是安全的。 

换句话说，如果可变参数数组仅用于从调用者向方法传递可变数量的参数——毕竟这是可变参数的目的——那么该方法是安全的。

## 不安全示例 

值得注意的是，你可以违反类型安全性，即使不会在可变参数数组中存储任何内容。 
考虑下面的泛型可变参数方法，它返回一个包含参数的数组。 乍一看，它可能看起来像一个方便的小工具：

    // UNSAFE - Exposes a reference to its generic parameter array!
    static <T> T[] toArray(T... args) {
        return args;
    }
    
这个方法只是返回它的可变参数数组。 该方法可能看起来并不危险，但它是！ 
该数组的类型由传递给方法的参数的编译时类型决定，编译器可能没有足够的信息来做出正确的判断。 
由于此方法返回其可变参数数组，它可以将堆污染传播到调用栈上。

为了具体说明，请考虑下面的泛型方法，它接受三个类型T的参数，并返回一个包含两个参数的数组，随机选择：

    static <T> T[] pickTwo(T a, T b, T c) {
        switch(ThreadLocalRandom.current().nextInt(3)) {
          case 0: return toArray(a, b);
          case 1: return toArray(a, c);
          case 2: return toArray(b, c);
        }
        throw new AssertionError(); // Can't get here
    }
    
这个方法本身不是危险的，除了调用具有泛型可变参数的toArray方法之外，不会产生警告。

编译此方法时，编译器会生成代码以创建一个将两个T实例传递给toArray的可变参数数组。 
这段代码分配了一个Object []类型的数组，它是保证保存这些实例的最具体的类型，而不管在调用位置传递给pickTwo的对象是什么类型。 
toArray方法只是简单地将这个数组返回给pickTwo，然后pickTwo将它返回给调用者，所以pickTwo总是返回一个Object []类型的数组。

现在考虑这个测试pickTw的main方法：

    public static void main(String[] args) {
        String[] attributes = pickTwo("Good", "Fast", "Cheap");
    }
    
这种方法没有任何问题，因此它编译时不会产生任何警告。 
但是当运行它时，抛出一个ClassCastException异常，尽管不包含可见的转换。 

你没有看到的是，编译器已经生成了一个隐藏的强制转换为由pickTwo返回的值的String []类型，以便它可以存储在属性中。 
转换失败，因为Object []不是String []的子类型。 

这种故障相当令人不安，因为它从实际导致堆污染（toArray）的方法中移除了两个级别，并且在实际参数存储在其中之后，可变参数数组未被修改。

这个例子是为了让人们认识到给另一个方法访问一个泛型的可变参数数组是不安全的，
除了两个例外：将数组传递给另一个可变参数方法是安全的，这个方法是用@SafeVarargs正确标注的， 将数组传递给一个非可变参数的方法是安全的，该方法仅计算数组内容的一些方法。

## 安全示例 

这里是安全使用泛型可变参数的典型示例。 

此方法将任意数量的列表作为参数，并按顺序返回包含所有输入列表元素的单个列表。 
由于该方法使用@SafeVarargs进行标注，因此在声明或其调用站位置上不会生成任何警告：

    // Safe method with a generic varargs parameter
    @SafeVarargs
    static <T> List<T> flatten(List<? extends T>... lists) {
        List<T> result = new ArrayList<>();
        for (List<? extends T> list : lists)
            result.addAll(list);
        return result;
    }
    
决定何时使用SafeVarargs注解的规则很简单：在每种方法上使用@SafeVarargs，并使用泛型或参数化类型的可变参数，
这样用户就不会因不必要的和令人困惑的编译器警告而担忧。 
这意味着你不应该写危险或者toArray等不安全的可变参数方法。 
每次编译器警告你可能会受到来自你控制的方法中泛型可变参数的堆污染时，请检查该方法是否安全。 

提醒一下，在下列情况下，泛型可变参数方法是安全的：
1. 它不会在可变参数数组中存储任何东西
2. 它不会使数组（或克隆）对不可信代码可见。 

如果违反这些禁令中的任何一项，请修复。

请注意，SafeVarargs注解只对不能被重写的方法是合法的，因为不可能保证每个可能的重写方法都是安全的。 
在Java 8中，注解仅在静态方法和final实例方法上合法; 在Java 9中，它在私有实例方法中也变为合法。

使用SafeVarargs注解的替代方法是采用条目 28的建议，并用List参数替换可变参数（这是一个变相的数组）。 
下面是应用于我们的flatten方法时，这种方法的样子。 请注意，只有参数声明被更改了：

    // List as a typesafe alternative to a generic varargs parameter
    static <T> List<T> flatten(List<List<? extends T>> lists) {
        List<T> result = new ArrayList<>();
        for (List<? extends T> list : lists)
            result.addAll(list);
        return result;
    }
    
然后可以将此方法与静态工厂方法List.of结合使用，以允许可变数量的参数。 请注意，这种方法依赖于List.of声明使用@SafeVarargs注解：

    audience = flatten(List.of(friends, romans, countrymen));

这种方法的优点是编译器可以证明这种方法是类型安全的。 
不必使用SafeVarargs注解来证明其安全性，也不用担心在确定安全性时可能会犯错。 主要缺点是客户端代码有点冗长，运行可能会慢一些。

这个技巧也可以用在不可能写一个安全的可变参数方法的情况下，就像第147页的toArray方法那样。

它的列表模拟是List.of方法，所以我们甚至不必编写它; Java类库作者已经为我们完成了这项工作。 pickTwo方法然后变成这样：

    static <T> List<T> pickTwo(T a, T b, T c) {
        switch(rnd.nextInt(3)) {
          case 0: return List.of(a, b);
          case 1: return List.of(a, c);
          case 2: return List.of(b, c);
        }
        throw new AssertionError();
    }
    
main方变成这样：

    public static void main(String[] args) {
        List<String> attributes = pickTwo("Good", "Fast", "Cheap");
    }
    
生成的代码是类型安全的，因为它只使用泛型，不是数组。

总而言之，可变参数和泛型不能很好地交互，因为可变参数机制是在数组上面构建的脆弱的抽象，并且数组具有与泛型不同的类型规则。 

虽然泛型可变参数不是类型安全的，但它们是合法的。 

如果选择使用泛型（或参数化）可变参数编写方法，请首先确保该方法是类型安全的，然后使用@SafeVarargs注解对其进行标注，以免造成使用不愉快。
