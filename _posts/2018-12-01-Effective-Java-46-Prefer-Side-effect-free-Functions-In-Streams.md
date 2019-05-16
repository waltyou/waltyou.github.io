---
layout: post
title: 《Effective Java》学习日志（六）46：在流中偏爱无副作用的函数
date: 2018-12-01 17:40:04
author: admin
comments: true
categories: [Java]
tags: [Java,Effective Java]
---

流看起来运行时很好用，但是要小心函数的副作用。

<!-- more -->

---

学习资料主要参考： 《Effective Java Third Edition》，作者：Joshua Bloch

---




* 目录
{:toc}

---

如果你是一个刚开始使用流的新手，那么很难掌握它们。
仅仅将计算表示为流管道是很困难的。
当你成功时，你的程序将运行，但对你来说可能没有意识到任何好处。
流不仅仅是一个API，它是基于函数式编程的范式（paradigm）。
为了获得流提供的可表达性、速度和某些情况下的并行性，你必须采用范式和API。

流范式中最重要的部分是将计算结构化为一系列转换，其中每个阶段的结果尽可能接近前一阶段结果的纯函数（ pure function）。 
**纯函数的结果仅取决于其输入：它不依赖于任何可变状态，也不更新任何状态。** 
为了实现这一点，你传递给流操作的任何函数对象（中间操作和终结操作）都应该没有副作用。

有时，可能会看到类似于此代码片段的流代码，该代码构建了文本文件中单词的频率表:

    // Uses the streams API but not the paradigm--Don't do this!
    Map<String, Long> freq = new HashMap<>();
    try (Stream<String> words = new Scanner(file).tokens()) {
        words.forEach(word -> {
            freq.merge(word.toLowerCase(), 1L, Long::sum);
        });
    }
    
这段代码出了什么问题？ 
毕竟，它使用了流，lambdas和方法引用，并得到正确的答案。 
简而言之，它根本不是流代码; 它是伪装成流代码的迭代代码。 
它没有从流API中获益，并且它比相应的迭代代码更长，更难读，并且更难于维护。 

问题源于这样一个事实：这个代码在一个终结操作forEach中完成所有工作，使用一个改变外部状态（频率表）的lambda。
forEach操作除了表示由一个流执行的计算结果外，什么都不做，这是“代码中的臭味”，就像一个改变状态的lambda一样。
那么这段代码应该是什么样的呢?

    // Proper use of streams to initialize a frequency table
    Map<String, Long> freq;
    try (Stream<String> words = new Scanner(file).tokens()) {
        freq = words
            .collect(groupingBy(String::toLowerCase, counting()));
    }
    
此代码段与前一代码相同，但正确使用了流API。 
它更短更清晰。 
那么为什么有人会用其他方式写呢？ 因为它使用了他们已经熟悉的工具。 
Java程序员知道如何使用for-each循环，而forEach终结操作是类似的。 
但forEach操作是终端操作中最不强大的操作之一，也是最不友好的流操作。 
它是明确的迭代，因此不适合并行化。 

**forEach操作应仅用于报告流计算的结果，而不是用于执行计算。**
有时，将forEach用于其他目的是有意义的，例如将流计算的结果添加到预先存在的集合中。

改进后的代码使用了收集器（collector），这是使用流必须学习的新概念。
Collectors的API令人生畏：它有39个方法，其中一些方法有多达5个类型参数。
好消息是，你可以从这个API中获得大部分好处，而不必深入研究它的全部复杂性。
对于初学者来说，可以忽略收集器接口，将收集器看作是封装缩减策略（ reduction strategy）的不透明对象。
在此上下文中，reduction意味着将流的元素组合为单个对象。 
收集器生成的对象通常是一个集合（它代表名称收集器）。

将流的元素收集到真正的集合中的收集器非常简单。
有三个这样的收集器:toList()、toSet()和toCollection(collectionFactory)。
它们分别返回集合、列表和程序员指定的集合类型。
有了这些知识，我们就可以编写一个流管道从我们的频率表中提取出现频率前10个单词的列表。

    // Pipeline to get a top-ten list of words from a frequency table
    List<String> topTen = freq.keySet().stream()
        .sorted(comparing(freq::get).reversed())
        .limit(10)
        .collect(toList());

注意，我们没有对toList方法的类收集器进行限定。
**静态导入收集器的所有成员是一种惯例和明智的做法，因为它使流管道更易于阅读。**

这段代码中唯一比较棘手的部分是我们把comparing(freq::get).reverse()传递给sort方法。
comparing是一种比较器构造方法(条目 14)，它具有一个key的提取方法。
该函数接受一个单词，而“提取”实际上是一个表查找：
绑定方法引用freq::get在frequency表中查找单词，并返回单词出现在文件中的次数。
最后，我们在比较器上调用reverse方法，因此我们将单词从最频繁到最不频繁进行排序。
然后，将流限制为10个单词并将它们收集到一个列表中就很简单了。

前面的代码片段使用Scanner的stream方法在scanner实例上获取流。
这个方法是在Java 9中添加的。
如果正在使用较早的版本，可以使用类似于条目 47中(streamOf(Iterable<E>))的适配器将实现了Iterator的scanner序转换为流。

那么收集器中的其他36种方法呢?它们中的大多数都是用于将流收集到map中的，这比将流收集到真正的集合中要复杂得多。
每个流元素都与一个键和一个值相关联，多个流元素可以与同一个键相关联。

最简单的映射收集器是toMap(keyMapper、valueMapper)，它接受两个函数，一个将流元素映射到键，另一个映射到值。
在条目34中的fromString实现中，我们使用这个收集器从enum的字符串形式映射到enum本身:

    // Using a toMap collector to make a map from string to enum
    private static final Map<String, Operation> stringToEnum =
        Stream.of(values()).collect(
            toMap(Object::toString, e -> e));

如果流中的每个元素都映射到唯一键，则这种简单的toMap形式是完美的。 
如果多个流元素映射到同一个键，则管道将以IllegalStateException终止。

toMap更复杂的形式，以及groupingBy方法，提供了处理此类冲突(collisions)的各种方法。
一种方法是向toMap方法提供除键和值映射器(mappers)之外的merge方法。
merge方法是一个BinaryOperator，其中V`是map的值类型。
与键关联的任何附加值都使用merge方法与现有值相结合，因此，例如，如果merge方法是乘法，那么最终得到的结果是是值mapper与键关联的所有值的乘积。

toMap的三个参数形式对于从键到与该键关联的选定元素的映射也很有用。
例如，假设我们有一系列不同艺术家（artists）的唱片集（albums），我们想要一张从唱片艺术家到最畅销专辑的map。
这个收集器将完成这项工作。

    // Collector to generate a map from key to chosen element for key
    Map<Artist, Album> topHits = albums.collect(
       toMap(Album::artist, a->a, maxBy(comparing(Album::sales))));

请注意，比较器使用静态工厂方法maxBy，它是从BinaryOperator静态导入的。 
此方法将Comparator <T>转换为BinaryOperator <T>，用于计算指定比较器隐含的最大值。 
在这种情况下，比较器由比较器构造方法comparing返回，它采用key提取器函数Album :: sales。 
这可能看起来有点复杂，但代码可读性很好。 
简而言之，它说，“将专辑（albums）流转换为地map，将每位艺术家（artist）映射到销售量最佳的专辑。”
这与问题陈述出奇得接近。

toMap的三个参数形式的另一个用途是产生一个收集器，当发生冲突时强制执行last-write-wins策略。 
对于许多流，结果是不确定的，但如果映射函数可能与键关联的所有值都相同，或者它们都是可接受的，则此收集器的行为可能正是您想要的：

    // Collector to impose last-write-wins policy
    toMap(keyMapper, valueMapper, (oldVal, newVal) ->newVal)

toMap的第三个也是最后一个版本采用第四个参数，它是一个map工厂，用于指定特定的map实现，例如EnumMap或TreeMap。

toMap的前三个版本也有变体形式，名为toConcurrentMap，它们并行高效运行并生成ConcurrentHashMap实例。

除了toMap方法之外，Collectors API还提供了groupingBy方法，该方法返回收集器以生成基于分类器函数(classifier function)将元素分组到类别中的map。 
分类器函数接受一个元素并返回它所属的类别。 
此类别来用作元素的map的键。 
groupingBy方法的最简单版本仅采用分类器并返回一个map，其值是每个类别中所有元素的列表。 
这是我们在条目 45中的Anagram程序中使用的收集器，用于生成从按字母顺序排列的单词到单词列表的map：

    Map<String, Long> freq = words
            .collect(groupingBy(String::toLowerCase, counting()));

groupingBy的第三个版本允许指定除downstream收集器之外的map工厂。 
请注意，这种方法违反了标准的可伸缩参数列表模式(standard telescoping argument list pattern)：mapFactory参数位于downStream参数之前，而不是之后。 
此版本的groupingBy可以控制包含的map以及包含的集合，因此，例如，可以指定一个收集器，它返回一个TreeMap，其值是TreeSet。

groupingByConcurrent方法提供了groupingBy的所有三个重载的变体。 
这些变体并行高效运行并生成ConcurrentHashMap实例。 
还有一个很少使用的grouping的亲戚称为partitioningBy。 
代替分类器方法，它接受predicate并返回其键为布尔值的map。 
此方法有两种重载，除了predicate之外，其中一种方法还需要downstream收集器。

通过counting方法返回的收集器仅用作下游收集器。 
Stream上可以通过count方法直接使用相同的功能，因此没有理由说collect(counting())。 
此属性还有十五种收集器方法。 
它们包括九个方法，其名称以summing，averaging和summarizing开头（其功能在相应的原始流类型上可用）。 
它们还包括reduce方法的所有重载，以及filter，mapping，flatMapping和collectingAndThen方法。 
大多数程序员可以安全地忽略大多数这些方法。 
从设计的角度来看，这些收集器代表了尝试在收集器中部分复制流的功能，以便下游收集器可以充当“迷你流(ministreams)”。

我们还有三种收集器方法尚未提及。 
虽然他们在收Collectors类中，但他们不涉及集合。 
前两个是minBy和maxBy，它们取比较器并返回比较器确定的流中的最小或最大元素。 
它们是Stream接口中min和max方法的次要总结，是BinaryOperator中类似命名方法返回的二元运算符的类似收集器。 
回想一下，我们在最畅销的专辑中使用了BinaryOperator.maxBy方法。

最后的Collectors中方法是join，它仅对CharSequence实例（如字符串）的流进行操作。 
在其无参数形式中，它返回一个简单地连接元素的收集器。 
它的一个参数形式采用名为delimiter的单个CharSequence参数，并返回一个连接流元素的收集器，在相邻元素之间插入分隔符。 
如果传入逗号作为分隔符，则收集器将返回逗号分隔值字符串（但请注意，如果流中的任何元素包含逗号，则字符串将不明确）。 
除了分隔符之外，三个参数形式还带有前缀和后缀。 
生成的收集器会生成类似于打印集合时获得的字符串，例如[came, saw, conquered]。

总之，编程流管道的本质是无副作用的函数对象。 
这适用于传递给流和相关对象的所有许多函数对象。 
终结操作orEach仅应用于报告流执行的计算结果，而不是用于执行计算。 
为了正确使用流，必须了解收集器。 
最重要的收集器工厂是toList，toSet，toMap，groupingBy和join。