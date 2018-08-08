---
layout: post
title: 《Effective Java》学习日志（一）：对象的创建与销毁
date: 2018-07-22 10:11:04
author: admin
comments: true
categories: [Java]
tags: [Java，Effective Java]
---

该如何编写有效的 Java 代码呢？来学习一下《Effective Java》第三版。

<!-- more -->
---

学习资料主要参考： 《Effective Java Third Edition》，作者：Joshua Bloch

---
## 目录
{:.no_toc}

* 目录
{:toc}

---

# 全书简介

这本书是为了帮助我们有效的使用 Java 语言和它的基础库（如java.lang , java.util , java.io） 和子包（如：java.util.concurrent and java.util.function）

共分为 11 章节和 90 个 item。 

每个 Item 表示一条规则，它们可以交叉阅读，因为它们都是独立的部分。

全书脑图如下：

[![](/images/posts/Effective+Java+3rd+Edition.png)](/images/posts/Effective+Java+3rd+Edition.png)

首先来看第一章：对象的创建与销毁。

--- 

# Item 1: 考虑使用静态工厂方法来代替构造方法

## 优点

### 1）静态工厂方法有名字

当构造函数的参数本身不能很好的描述函数返回的是什么样的对象时，一个有好名字的静态方法，会帮助客户端代码更好的理解。

举个例子就是：构造函数 BigInteger(int, int, Random)，它返回了一个可能是素数的 BigIntege， 但是如果使用静态工厂方法 BigInteger.probablePrime ，表达就会更加清晰。

另外，我们都知道对于给定的一个标识，一个类只能有一个对应的构造函数。但有时候，为了打破这个限制，程序员可能会使用两个仅仅参数顺序不一致的构造函数来解决这个问题。这是个很不好的行为。因为使用者很可能分不清哪个构造函数该被使用，从而导致错误发生。除非他们认真的阅读使用文档。

但是静态工厂方法的名字就解决了上述问题，只需要取两个定义清晰且不同的名字就可以了。
 
### 2）静态工厂方法不需要每次都创建新对象

这个特点允许不变类（immutable class）来使用提前构造好的实例，或来缓存他们构造的实例，又或可以重复分发已有实例来避免创建重复的对象。

举个例子就是 Boolean.valueOf(boolean)：

```java
public static Boolean valueOf(boolean b) {
    return b ? Boolean.TRUE : Boolean.FALSE;
}
```
这个方法从来不会创建一个对象，有点像是设计模式中的享元模式(Flyweight Pattern)。如果经常请求同样的对象，它可以极大地提高性能，特别是它们的创建代价很昂贵时。
 
静态工厂方法保证了在反复的调用中都能返回相同的对象，它的这种能力保证了类对存在的实例进行严格的控制。这种控制叫做“实例控制 instance-controlled”。

有以下几个原因来写实例控制的类：
- 保证类是单例或者不可实例化的
- 对于不变值的类，可以保证他们是相等的
- 这是享元模式的基础

### 3）静态工厂方法可以返回其返回类型的任何子类型对象

这个功能让使用者可以更加灵活地选择返回对象的类。

这个灵活性的一个应用就是 API 可以在不使返回对象类公开的情况下，返回一个对象。只需要返回对象的类，是静态工厂方法定义时规定的返回类型的子类即可。这项技术适用于基于接口的框架（interface-based frameworks），这里的接口提供对象的原生返回类型。

按照惯例，一个名字是“Type”的接口，它的静态工厂方法，通常都会放在一个名为“Types”的不可实例化的伴随类中。例如，Java Collections Framework的接口有45个实用程序实现，提供不可修改的集合，同步集合等。几乎所有这些实现都是通过静态工厂方法在一个不可实例化的类（java.util.Collections）中导出的。返回对象的类都是非公共的。

借助这种技术，Collections 类就变的小了很多。这不仅仅是API大部分的减少，也包括概念上的重量：程序员为使用API必须掌握概念的数量和难度。程序员知道返回的对象具有其接口指定的API，因此不需要为这个实现类而阅读额外的类文档。

此外，使用这种静态工厂方法，需要客户端通过**接口**而不是**实现类**来引用返回的对象，这通常是一种很好的做法。

从Java 8开始，消除了接口不能包含静态方法的限制，因此通常没有理由为接口提供不可实例化的伴随类。许多公共静态成员应该放在接口本身中。但请注意，可能仍有必要将大量实现代码放在这些静态方法后面的单独的包私有（package-private）类中。这是因为Java 8要求接口的所有静态成员都是公共的。Java 9允许私有静态方法，但静态字段和静态成员类仍然需要公开。

### 4）静态工厂方法可以根据输入参数而改变返回对象的类

返回对象的类型，只要是声明类型的子类型就可以。

EnumSet 类就没有公共的构造方法，只有静态工厂。在 OpenJDk 的实现上，它可以返回两个子类型中的其中一种：如果 enum type 数量小于等于64，静态工厂会返回 RegularEnumSet，否则，会返回 JumboEnumSet 。

这两种实现的子类，对于调用者是不可见的。所以，如果将来出于性能考虑，移除这个类，那对使用者也毫无影响。同样的，再添加一个新的子类，对调用者也无影响。

### 5）在写静态工厂方法时，方法返回对象的类不需要存在。

这种灵活的静态工厂方法构成了服务提供者框架（service provider frameworks）的基础，如Java数据库连接API（JDBC）。服务提供者框架是提供者负责实现服务的系统。系统使实现可用于客户端，将客户端与实现分离。

服务提供者框架中有三个基本组件：
- service interface，代表一个具体实现
- provider registration API，提供者用于注册一个实现
- service access API，客户端使用它来获取服务的实例

Service access API可以允许客户端指定用于选择实现的标准，如果没有这样的标准，API将返回默认实现的实例，或允许客户端循环遍历所有可用的实现。 Service access API是灵活的静态工厂，它构成了服务提供者框架的基础。

另外一个可选的组件是：service provider interface，它用来描述一个生产service interface实例的工厂对象。在缺少服务提供者接口的情况下，必须反射地实例化实现。在 JDBC 中, Connection 作为 service interface, DriverManager.registerDriver 作为 provider registration API, DriverManager.getConnection 作为 service access API, Driver 是 service provider interface.

服务提供者框架模式有许多变体。 例如，服务访问API可以向客户端返回比提供者提供的服务接口更丰富的服务接口。这就是桥接模式(Bridge Pattern)。 依赖注入框架也看做是强大的服务提供者。 Java 6 提供了通用目的的服务提供者框架：java.util.ServiceLoader，所以你无需自己实现。

## 局限性

### 1）没有 public 或 protected 构造函数的类不能被子类化

例如，我们不可能在Collections Framework中继承任何便捷的实现类。

可以说这可能是一种伪装的祝福，因为它鼓励程序员使用组合而不是继承（第18项），并且是不可变类型（第17项）所必需的。

### 2）静态工厂方法不能容易的被使用者找到

构造方法，我们不看 API 文档也知道，但是静态工厂方法不一样，所以我们最好约定一些命名规范，来减少问题的发生。如下：

- from: 一种类型转换方法，它接受一个参数并返回一个相应的这种类型的实例。
    ```
    Date d = Date.from(instant);
    ```
- of：一种聚合方法，它接受多个参数并返回实例包含它们的这种类型
    ```
    Set<Rank> faceCards = EnumSet.of(JACK, QUEEN, KING);
    ```
- valueOf：一个更详细的替代 from 和 of
    ```
    BigInteger prime = BigInteger.valueOf(Integer.MAX_VALUE);
    ```
- instance or getInstance：返回由其参数（如果有）描述的实例，但不能说具有相同的值
    ```
    StackWalker luke = StackWalker.getInstance(options);
    ```
- create or newInstance：像instance或getInstance，但是该方法保证每个调用返回一个新实例
    ```
    Object newArray = Array.newInstance(classObject, arrayLen);
    ```
- getType：与getInstance类似，但如果工厂方法位于不同的类中，则使用它。 Type是工厂方法返回的对象类型
    ```
    FileStore fs = Files.getFileStore(path);
    ```
- newType：与newInstance类似，但如果工厂方法在不同的类中，则使用。 Type是工厂方法返回的对象类型
    ```
    BufferedReader br = Files.newBufferedReader(path);
    ```
- type：getType和newType的简明替代方案
    ```
    List<Complaint> litany = Collections.list(legacyLitany);
    ```

---

# Item 2：当构造函数有许多参数的时，请考虑构建器（Builder）

静态工厂和构造函数共享一个限制：当有很多可选参，它们不能很好地扩展。

因为面对这种可选参数较多的情况，构造函数无论如何都需要传递一个值给它，即使这些参数我们不需要。

直观上，我们可以采用**伸缩构造模式**的方法（也就是函数复用），来一定程度上解决这个问题。但是当参数变得更多时，这个思路下代码就会臃肿起来。而且程序也变得更加难以阅读。

第二个思路是**JavaBeans**模式，也就是使用 get、set 方法。您可以在其中调用无参数构造函数来创建对象，然后调用setter方法来设置每个必需参数和每个感兴趣的可选参数。这个方法没有上一个方法的缺点。它很容易创建实例，并且易于阅读生成的代码。

不幸的是，JavaBeans模式本身就存在严重的缺点。因为想要构造出一个完整地对象，需要多次调用，而这些调用在多线程的情况下，可以会出现不一致的状态。当然我们可以使用锁来避免这类错误，但是程序就变得笨重了。

幸运的是，这里有第三种方式，就是生成器模式（Builder Pattern）。它先用必须的参数，构建一个builder对象，然后再设置那些可选参数（这一步有些类似setter函数），最后，通过调用 builder 方法，生成最后的对象。

```java
public class NutritionFacts {

    private final int servingSize;
    private final int servings;
    private final int calories;
    private final int fat;
    private final int sodium;
    private final int carbohydrate;
    
    public static class Builder {
        // Required parameters
        private final int servingSize;
        private final int servings;
        
        // Optional parameters - initialized to default values
        private int calories = 0;
        private int fat = 0;
        private int sodium = 0;
        private int carbohydrate = 0;
        
        public Builder(int servingSize, int servings) {
            this.servingSize = servingSize;
            this.servings = servings;
        }
        
        public Builder calories(int val)
        { calories = val; return this; }
        public Builder fat(int val)
        { fat = val; return this; }
        public Builder sodium(int val)
        { sodium = val; return this; }
        public Builder carbohydrate(int val)
        { carbohydrate = val; return this; }
        
        public NutritionFacts build() {
            return new NutritionFacts(this);
        }
    }
    
    private NutritionFacts(Builder builder) {
        servingSize = builder.servingSize;
        servings = builder.servings;
        calories = builder.calories;
        fat = builder.fat;
        sodium = builder.sodium;
        carbohydrate = builder.carbohydrate;
    }
}
```

客户端的调用程序是这样子的：

```java
NutritionFacts cocaCola = new NutritionFacts.Builder(240, 8)
.calories(100).sodium(35).carbohydrate(27).build();
```

Builder模式模拟Python和Scala中的命名可选参数。

另外，需要尽早在builder函数中检查参数的有效性，如果不满足，及时抛出 IllegalArgumentException，并指明具体的无效参数。

Builder模式非常适合类层次结构。使用并行的构建器层次结构，每个构建器都嵌套在相应的类中。 抽象类有抽象构建器; 具体的类有具体的建设者。

构建器相对于构造函数的一个小优点是构建器可以具有多个varargs参数，因为每个参数都在其自己的方法中指定。 或者，构建器可以将传递给方法的多个调用的参数聚合到单个字段中。

Builder模式非常灵活。 可以重复使用单个构建器来构建多个对象。 可以在构建方法的调用之间调整构建器的参数，以改变创建的对象。 构建器可以在创建对象时自动填充某些字段，例如每次创建对象时增加的序列号。

Builder模式也有缺点，就是要创建对象，必须先创建其构建器。虽然在实践中创建此构建器的成本不太可能明显，但在性能关键的情况下可能会出现问题。

此外，Builder模式比伸缩构造函数模式更冗长，因此只有在有足够的参数使其值得（例如四个或更多）时才应使用它。但请记住，参数可能在未来会变多。

但是如果一开始写的是构造函数或静态工厂，那么随着需求变化，在参数数量多到失控时，再切换到构建器，那么过时的构造函数或静态工厂就很冗余了。因此，首先从 builder 模式开始通常会更好。

总之，在设计构造函数或静态工厂具有多个参数的类时，Builder模式是一个不错的选择，特别是如果许多参数是可选的或类型相同的话。与使用伸缩式构造函数相比，客户端代码更易于使用构建器进行读写，与JavaBeans相比，则更安全。

---

# Item 3: 强制单例属性为私有构造函数或枚举类型

## 1. 什么是单例

单例只是一个实例化一次的类。单例通常代表无状态对象，例如函数或本质上唯一的系统组件。

通常有两种方式实现单例。这两种都是保证构造函数是私有的，然后提供公共静态成员唯一的获取方式。

## 2. 第一种方法

```java
// Singleton with public final field
public class Elvis {
    public static final Elvis INSTANCE = new Elvis();
    private Elvis() { ... }
    public void leaveTheBuilding() { ... }
}
```
客户端调用可以直接使用*Elvis.INSTANCE*来获取对象。这种方法创建的单例对象，在类加载时就会创建。不过要小心的时，可以通过反射来调用构造方法，所以当这种情况发生时，需要在构造函数中抛出异常。

这个方法的主要优点是API清楚地表明该类是单例：公共静态字段是final，因此它将始终包含相同的对象引用。 第二个优点是它更简单。

## 3. 第二种方法

```java
// Singleton with static factory
public class Elvis {
    private static final Elvis INSTANCE = new Elvis();
    private Elvis() { ... }
    public static Elvis getInstance() { return INSTANCE; }
    public void leaveTheBuilding() { ... }
}
```
客户端使用 *Elvis.getInstance* 来获取对象。

静态工厂方法的一个优点是，它能在你不更改API的情况下，灵活地控制类是否为单例。工厂方法返回唯一的实例，但可以修改它，例如，为每个调用它的线程返回一个单独的实例。

第二个优点是，如果您的应用需要，您可以编写通用的单例工厂。 

使用静态工厂的最后一个优点是方法引用可以用作供应商，例如Elvis :: instance是Supplier <Elvis>。 

除非是为了其中一个优点，否则第一种方法更可取。

## 4. 序列化

要注意对一个拥有单例属性的类来讲，仅仅实现 *Serializable* 接口是不够的。而是要将单例属性前加上 *transient* 关键字，否则每一次的反序列化，都会创建出一个的新的对象。在反序列化后，如果需要获取单例属性，需要添加 *readResolve* 方法。

## 5. 第三种方法

第三种实现单例的方式就是声明一个单元素的枚举类型。

```java
// Enum singleton - the preferred approach
public enum Elvis {
    INSTANCE;
    public void leaveTheBuilding() { ... }
}
```

这种方法类似于公共领域方法，但它更简洁，免费提供序列化机制，并提供了对多次实例化的铁定保证，即使面对复杂的序列化或反射攻击。

这种方法可能会有点不自然，但单元素枚举类型通常是实现单例的最佳方法。 请注意，如果您的单例必须扩展Enum以外的超类，则不能使用此方法（尽管您可以声明枚举来实现接口）。

---

# Item 5：使用依赖注入取代硬连接资源

许多类依赖于一个或多个底层资源。例如，拼写检查器依赖于字典。

比如将其实现为静态实用工具类：
```java
// Inappropriate use of static utility - inflexible & untestable!
public class SpellChecker {
    private static final Lexicon dictionary = ...;

    private SpellChecker() {} // Noninstantiable

    public static boolean isValid(String word) { ... }
    public static List<String> suggestions(String typo) { ... }
}
```
又或者，将它们实现为单例：
```java
// Inappropriate use of singleton - inflexible & untestable!
public class SpellChecker {
    private final Lexicon dictionary = ...;

    private SpellChecker(...) {}
    public static INSTANCE = new SpellChecker(...);

    public boolean isValid(String word) { ... }
    public List<String> suggestions(String typo) { ... }
}
```

然而这两种方法都不令人满意，因为他们都假设只有一本字典值得使用。在实际中，每种语言都有自己的字典，特殊的字典被用于特殊的词汇表。另外，使用专门的字典来进行测试也是可取的。想当然地认为一本字典就足够了，这是一厢情愿的想法。

可以通过使dictionary属性设置为非final，并添加一个方法来更改现有拼写检查器中的字典，从而让拼写检查器支持多个字典，但是在并发环境中，这是笨拙的、容易出错的和不可行的。**静态实用类和单例对于那些行为被底层资源参数化的类来说是不合适的**。

所需要的是能够支持类的多个实例(在我们的示例中，即SpellChecker)，每个实例都使用客户端所期望的资源(在我们的例子中是dictionary)。满足这一需求的简单模式是在**创建新实例时将资源传递到构造方法**中。这是**依赖项注入（dependency injection）**的一种形式：字典是拼写检查器的一个依赖项，当它创建时被注入到拼写检查器中。

```java
// Dependency injection provides flexibility and testability
public class SpellChecker {
    private final Lexicon dictionary;

    public SpellChecker(Lexicon dictionary) {
        this.dictionary = Objects.requireNonNull(dictionary);
    }

    public boolean isValid(String word) { ... }
    public List<String> suggestions(String typo) { ... }
}
```

依赖注入模式非常简单。虽然我们的拼写检查器的例子只有一个资源（这里是字典），但是依赖项注入可以使用任意数量的资源和任意依赖图。它保持了不变性，因此多个客户端可以共享依赖对象（假设客户需要相同的底层资源）。 依赖注入同样适用于构造方法，静态工厂和 builder模式。

该模式的一个有用的变体是将资源工厂传递给构造方法。工厂是可以重复调用以创建类型实例的对象。 这种工厂体现了工厂方法模式（Factory Method pattern ）。 Java 8中引入的*Supplier<T>*接口非常适合代表工厂。 在输入上采用Supplier<T>的方法通常应该使用有界的通配符类型( bounded wildcard type)约束工厂的类型参数，以允许客户端传入工厂，创建指定类型的任何子类型。 

例如，下面是一个使用客户端提供的工厂生成tile的方法：

```java
Mosaic create(Supplier<? extends Tile> tileFactory) { ... }
```

尽管依赖注入极大地提高了灵活性和可测试性，但它可能使大型项目变得混乱，这些项目通常包含数千个依赖项。使用依赖注入框架(如Dagger[Dagger]、Guice[Guice]或Spring[Spring])可以消除这些混乱。这些框架的使用超出了本书的范围，但是请注意，为手动依赖注入而设计的API非常适合使用这些框架。

总之，当类依赖于一个或多个底层资源，不要使用单例或静态的实用类来实现一个类，这些资源的行为会影响类的行为，并且不要让类直接创建这些资源。相反，将资源或工厂传递给构造方法(或静态工厂或builder模式)。这种称为依赖注入的实践将极大地增强类的灵活性、可重用性和可测试性。

---

# Item 6: 避免创建不必要的对象

通常重用单个对象，比起每次需要时创建一个新的功能等效对象，要更加合适。 重复使用可以更快，更优雅。如果一个对象是不可变的，那么它总是可以被重用。

以下的这个例子就是不合适的。
```java
String s = new String("bikini"); // DON'T DO THIS!
```
它每次都会创建一个新的string对象，而这些创建都是无意义的。因为String的构造函数参数就是一个String对象。如果这是在一个大的循环中，那么更加浪费。

改善的版本应该如下：
```java
String s = "bikini";
```
这保证了在同一个虚拟机中，只有一个相同内容的 String 实例。

通过使用静态工厂方法，可以避免创建不需要的对象。例如，工厂方法*Boolean.valueOf(String)*比构造方法*Boolean(String)*更可取，后者在Java 9中被弃用。构造方法每次调用时都必须创建一个新对象，而工厂方法永远不需要这样做，在实践中也不需要。除了重用不可变对象，如果知道它们不会被修改，还可以重用可变对象。

一些对象的创建会比其他的昂贵的多。如果你需要重复使用这些创建昂贵的对象，把它缓存并复用它，将是个明智的选择。不幸的是，创建这种昂贵对象的动作，并不是总是明显可见的。

比如你想写一个正则来判断一个String是否为罗马数字。

```java
// Performance can be greatly improved!
static boolean isRomanNumeral(String s) {
    return s.matches("^(?=.)M*(C[MD]|D?C{0,3})"
            + "(X[CL]|L?X{0,3})(I[XV]|V?I{0,3})$");
}
```
上面的这个实现，主要问题在于它依赖于 *String.matches* 方法。虽然 String.matches 方法是对一个 string 进行正则匹配的最简单的方式，但是它不适合在高性能要求的情况下重复使用。问题是它在内部为正则表达式创建一个Pattern实例，并且只使用它一次，之后它就有资格进行垃圾收集。而创建Pattern实例是昂贵的，因为它需要将正则表达式编译成有限状态机（finite state machine）。

为了改善性能，可以将正则表达式显式编译为一个不可变的Pattern实例，作为类初始化的一部分，来缓存它，并在isRomanNumeral方法的每个调用中重复使用相同的实例：

```java
// Reusing expensive object for improved performance
public class RomanNumerals {
    private static final Pattern ROMAN = Pattern.compile(
            "^(?=.)M*(C[MD]|D?C{0,3})"
            + "(X[CL]|L?X{0,3})(I[XV]|V?I{0,3})$");
    static boolean isRomanNumeral(String s) {
        return ROMAN.matcher(s).matches();
    }
}
```
如果经常调用，isRomanNumeral的改进版本的性能会显著提升。

如果包含isRomanNumeral方法的改进版本的类被初始化，但该方法从未被调用，则ROMAN属性则没必要初始化。在第一次调用isRomanNumeral方法时，可以通过延迟初始化（ lazily initializing）属性来排除初始化，但一般不建议这样做。延迟初始化常常会导致实现复杂化，而性能没有可衡量的改进。

当一个对象是不可变的时，很明显它可以被安全地重用，但是在其他情况下，它远没有那么明显，甚至是违反直觉的。考虑适配器（adapters）的情况，也称为视图（views）。一个适配器是一个对象，它委托一个支持对象（backing object），提供一个可替代的接口。由于适配器没有超出其支持对象的状态，因此不需要为给定对象创建多个给定适配器的实例。

例如，Map接口的keySet方法返回Map对象的Set视图，包含Map中的所有key。 天真地说，似乎每次调用keySet都必须创建一个新的Set实例，但是对给定Map对象的keySet的每次调用都返回相同的Set实例。
尽管返回的Set实例通常是可变的，但是所有返回的对象在功能上都是相同的：当其中一个返回的对象发生变化时，所有其他对象也都变化，因为它们全部由相同的Map实例支持。
虽然创建keySet视图对象的多个实例基本上是无害的，但这是没有必要的，也没有任何好处。

另一种创建不必要的对象的方法是自动装箱（autoboxing），它允许程序员混用基本类型和包装的基本类型，根据需要自动装箱和拆箱。自动装箱模糊不清，但不会消除基本类型和装箱基本类型之间的区别。有微妙的语义区别和不那么细微的性能差异。 考虑下面的方法，它计算所有int正整数的总和。

要做到这一点，程序必须使用long类型，因为int类型不足以保存所有正整数的总和：
```
// 非常慢！ 你能发现对象的创建吗？
private static long sum() {
    Long sum = 0L;
    for (long i = 0; i <= Integer.MAX_VALUE; i++)
        sum += i;
    return sum;
}
```
这个程序的结果是正确的，但由于写错了一个字符，运行的结果要比实际慢很多。变量sum被声明成了Long而不是long，这意味着程序构造了大约2^31不必要的Long实例（大约每次往Long类型的 sum变量中增加一个long类型构造的实例）。

把sum变量的类型由Long改为long，在我的机器上运行时间从6.3秒降低到0.59秒。这个教训很明显：优先使用基本类型而不是装箱的基本类型，也要注意无意识的自动装箱。

这个条目不应该被误解为暗示对象创建是昂贵的，应该避免创建对象。 相反，使用构造方法创建和回收小的对象是非常廉价，构造方法只会做很少的显示工作，，尤其是在现代JVM实现上。创建额外的对象以增强程序的清晰度，简单性或功能性通常是件好事。

相反，除非池中的对象非常重量级，否则通过维护自己的对象池来避免对象创建是一个坏主意。对象池的典型例子就是数据库连接。建立连接的成本非常高，因此重用这些对象是有意义的。但是，一般来说，维护自己的对象池会使代码混乱，增加内存占用，并损害性能。现代JVM实现具有高度优化的垃圾收集器，它们在轻量级对象上轻松胜过此类对象池。

这个条目的对应点是针对 Item 50 的防御性复制（defensive copying）。 目前的条目说：“当你应该重用一个现有的对象时，不要创建一个新的对象”，而Item 50说：“不要重复使用现有的对象，当你应该创建一个新的对象时。”
请注意，重用防御性复制所要求的对象所付出的代价，要远远大于不必要地创建重复的对象。未能在需要的情况下防御性复制会导致潜在的错误和安全漏洞；而不必要地创建对象只会影响程序的风格和性能。

---

# Item 7：消除过时的对象引用

如果你是从 C++ 之类的语言过渡到 Java 来的，你一定会觉得编程简单了许多，因为 Java 自带垃圾回收机制。这个过程看起来有些很神奇，而且很容易给你造成一个错觉，那就是不需要再关心内存的使用情况了。

```java
// Can you spot the "memory leak"?
public class Stack {
    private Object[] elements;
    private int size = 0;
    private static final int DEFAULT_INITIAL_CAPACITY = 16;
    
    public Stack() {
        elements = new Object[DEFAULT_INITIAL_CAPACITY];
    }
    
    public void push(Object e) {
        ensureCapacity();
        elements[size++] = e;
    }
    
    public Object pop() {
        if (size == 0)
            throw new EmptyStackException();
        return elements[--size];
    }
    
    /**
    * Ensure space for at least one more element, roughly
    * doubling the capacity each time the array needs to grow.
    */
    private void ensureCapacity() {
        if (elements.length == size)
            elements = Arrays.copyOf(elements, 2 * size + 1);
    }
}
```
上面这个程序，乍看没有问题，但是当你仔细观察，就会发现，它有一个潜在的问题，那就是 pop 方法，stack pop 出一个对象后，elements 仍然持有该对象的引用。也就是说这些pop的对象不会被垃圾回收，因为stack维护了对这些对象的过期引用（obsolete references）。

垃圾收集语言中的内存泄漏（更适当地称为无意的对象保留 unintentional object retentions）是隐蔽的。如果无意中保留了对象引用，那么不仅这个对象排除在垃圾回收之外，而且该对象引用的任何对象也是如此。即使只有少数对象引用被无意地保留下来，也可以阻止垃圾回收机制对许多对象的回收，这对性能产生很大的影响。

这类问题的解决方法很简单：一旦对象引用过期，将它们设置为null。代码如下：

```java
public Object pop() {
    if (size == 0)
        throw new EmptyStackException();
    Object result = elements[--size];
    elements[size] = null; // Eliminate obsolete reference
    return result;
}
```

取消过期引用的另一个好处是，如果它们随后被错误地引用，程序立即抛出NullPointerException异常，而不是悄悄地做继续做错误的事情。尽可能快地发现程序中的错误是有好处的。

那么什么时候应该清空一个引用呢？Stack类的哪个方面使它容易受到内存泄漏的影响？简单地说，**它管理自己的内存**。存储池（storage pool）由elements数组的元素组成(对象引用单元，而不是对象本身)。数组中活动部分的元素(如前面定义的)被分配，其余的元素都是空闲的。垃圾收集器没有办法知道这些；对于垃圾收集器来说，elements数组中的所有对象引用都同样有效。只有程序员知道数组的非活动部分不重要。程序员可以向垃圾收集器传达这样一个事实，一旦数组中的元素变成非活动的一部分，就可以手动清空这些元素的引用。

一般来说，当一个类自己管理内存时，程序员应该警惕内存泄漏问题。每当一个元素被释放时，元素中包含的任何对象引用都应该被清除

另一个常见的内存泄漏来源是**缓存**。一旦将对象引用放入缓存中，很容易忘记它的存在，并且在它变得无关紧要之后，仍然保留在缓存中。对于这个问题有几种解决方案。如果你正好想实现了一个缓存：只要在缓存之外存在对某个项（entry）的键（key）引用，那么这项就是明确有关联的，就可以用WeakHashMap来表示缓存；这些项在过期之后自动删除。记住，只有当缓存中某个项的生命周期是由外部引用到键（key）而不是值（value）决定时，WeakHashMap才有用。

更常见的情况是，缓存项有用的生命周期不太明确，随着时间的推移一些项变得越来越没有价值。在这种情况下，缓存应该偶尔清理掉已经废弃的项。这可以通过一个后台线程(也许是ScheduledThreadPoolExecutor)或将新的项添加到缓存时顺便清理。LinkedHashMap类使用它的removeEldestEntry方法实现了后一种方案。对于更复杂的缓存，可能直接需要使用java.lang.ref。

第三个常见的内存泄漏来源是监听器和其他回调。如果你实现了一个API，其客户端注册回调，但是没有显式地撤销注册回调，除非采取一些操作，否则它们将会累积。确保回调是垃圾收集的一种方法是只存储弱引用（weak references），例如，仅将它们保存在WeakHashMap的键（key）中。

因为内存泄漏通常不会表现为明显的故障，所以它们可能会在系统中保持多年。 通常仅在仔细的代码检查或借助堆分析器（ heap profiler）的调试工具才会被发现。 因此，学习如何预见这些问题，并防止这些问题发生，是非常值得的。

---

# Item 8：避免使用 finalizers 和 cleaners

Finalizers 不可预见它的行为，经常也是危险的，同时也是没必要的。它们的使用会导致不稳定的行为、低下的性能和移植性的问题。虽然它有一些合适的场景，但是总而言之，还是应该避免使用它。

Java 9 中， finalizer 已经被弃用了，虽然它还在被 Java 库使用。Java 9 使用 Cleaners 来替代 finalizer。虽然 cleaner 比起 finalizer 少了一些危险，但是它仍然是不可预知的、慢的和没必要的。

提醒C++程序员不要把Java中的Finalizer或Cleaner机制当成的C ++析构函数（destructors）的等价物。在C++中，析构函数是回收对象相关资源的正常方式，是与构造方法相对应的。在Java中，当一个对象变得不可达时，垃圾收集器回收与对象相关联的存储空间，不需要开发人员做额外的工作。 C ++析构函数也被用来回收其他非内存资源。在Java中，try-with-resources或try-finally块用于此目的。

Finalizer和Cleaner机制的一个缺点是**不能保证他们能够及时执行**。 在一个对象变得无法访问时，到Finalizer和Cleaner机制开始运行时，这期间的时间是任意长的。 这意味着你永远不应该Finalizer和Cleaner机制做任何时间敏感（time-critical）的事情。例如，依赖于Finalizer和Cleaner机制来关闭文件是严重的错误，因为打开的文件描述符是有限的资源。 如果由于系统迟迟没有运行Finalizer和Cleaner机制而导致许多文件被打开，程序可能会失败，因为它不能再打开文件了。

及时执行Finalizer和 Cleaner机制是垃圾收集算法的一个功能，这种算法在不同的实现中有很大的不同。程序的行为依赖于Finalizer和Cleaner机制的及时执行，其行为也可能大不不同。 这样的程序完全可以在你测试的JVM上完美运行，然而在你最重要的客户的机器上可能运行就会失败。

延迟终结（finalization）不只是一个理论问题。为一个类提供一个Finalizer机制可以任意拖延它的实例的回收。一位同事调试了一个长时间运行的GUI应用程序，这个应用程序正在被一个神秘的 OutOfMemoryError 错误而死掉。分析显示，在它死亡的时候，应用程序的Finalizer机制队列上有成千上万的图形对象正在等待被终结和回收。不幸的是，Finalizer机制线程的运行优先级低于其他应用程序线程，所以对象被回收的速度低于进入队列的速度。语言规范并不保证哪个线程执行Finalizer机制，因此除了避免使用Finalizer机制之外，没有轻便的方法来防止这类问题。在这方面，Cleaner 机制比 Finalizer 机制要好一些，因为 Java 类的创建者可以控制自己 cleaner 机制的线程，但 cleaner 机制仍然在后台运行，在垃圾回收器的控制下运行，但不能保证及时清理。

Java规范不能保证Finalizer和Cleaner机制能及时运行；**它甚至不能能保证它们是否会运行**。当一个程序结束后，一些不可达对象上的Finalizer和Cleaner机制仍然没有运行。因此，不应该依赖于Finalizer和Cleaner机制来更新持久化状态。例如，依赖于Finalizer和Cleaner机制来释放对共享资源(如数据库)的持久锁，这是一个使整个分布式系统陷入停滞的好方法。

**不要相信System.gc和System.runFinalization方法**。 他们可能会增加Finalizer和Cleaner机制被执行的几率，但不能保证一定会执行。曾经声称做出这种保证的两个方法：System.runFinalizersOnExit 和它的孪生兄弟 Runtime.runFinalizersOnExit ，包含致命的缺陷，并已被弃用了几十年。

Finalizer机制的另一个问题是在执行Finalizer机制过程中，**未捕获的异常会被忽略，并且该对象的Finalizer机制也会终止**。未捕获的异常会使其他对象陷入一种损坏的状态（corrupt state）。如果另一个线程试图使用这样一个损坏的对象，可能会导致任意不确定的行为。通常情况下，未捕获的异常将终止线程并打印堆栈跟踪（ stacktrace），但如果发生在Finalizer机制中，则不会发出警告。Cleaner机制没有这个问题，因为使用Cleaner机制的类库可以控制其线程。

使用finalizer和cleaner机制会**导致严重的性能损失**。在我的机器上，创建一个简单的AutoCloseable对象，使用try-with-resources关闭它，并让垃圾回收器回收它的时间大约是12纳秒。 使用finalizer机制，而时间增加到550纳秒。 换句话说，使用finalizer机制创建和销毁对象的速度要慢50倍。 这主要是因为*finalizer机制会阻碍有效的垃圾收集*。 如果使用它们来清理类的所有实例(在我的机器上的每个实例大约是500纳秒)，那么cleaner机制的速度与finalizer机制的速度相当，但是如果仅将它们用作安全网（ safety net），则cleaner机制要快得多，如下所述。在这种环境下，创建，清理和销毁一个对象在我的机器上需要大约66纳秒，这意味着如果你不使用安全网的话，需要支付5倍(而不是50倍)的保险。

finalizer机制有一个严重的安全问题：**它们会打开你的类来进行finalizer机制攻击**。finalizer机制攻击的想法很简单：如果一个异常是从构造方法或它的序列化中抛出的——readObject和readResolve方法(第12章)——恶意子类的finalizer机制可以运行在本应该“中途夭折（died on the vine）”的部分构造对象上。finalizer机制可以在静态字属性记录对对象的引用，防止其被垃圾收集。一旦记录了有缺陷的对象，就可以简单地调用该对象上的任意方法，而这些方法本来就不应该允许存在。从构造方法中抛出异常应该足以防止对象出现；而在finalizer机制存在下，则不是。这样的攻击会带来可怕的后果。Final类不受finalizer机制攻击的影响，因为没有人可以编写一个final类的恶意子类。为了保护非final类不受finalizer机制攻击，编写一个final的finalize方法，它什么都不做。

那么，你应该怎样做呢？为对象封装需要结束的资源(如文件或线程)，而不是为该类编写Finalizer和Cleaner机制？让你的类实现AutoCloseable接口即可，并要求客户在在不再需要时调用每个实例close方法，通常使用try-with-resources确保终止，即使面对有异常抛出情况。一个值得一提的细节是实例必须跟踪是否已经关闭：close方法必须记录在对象里不再有效的属性，其他方法必须检查该属性，如果在对象关闭后调用它们，则抛出IllegalStateException异常。

那么，Finalizer和Cleaner机制有什么好处呢？它们可能有两个合法用途。一个是作为一个**安全网**（safety net），以防资源的拥有者忽略了它的close方法。虽然不能保证Finalizer和Cleaner机制会迅速运行(或者根本就没有运行)，最好是把资源释放晚点出来，也要好过客户端没有这样做。如果你正在考虑编写这样的安全网Finalizer机制，请仔细考虑一下这样保护是否值得付出对应的代价。一些Java库类，如FileInputStream、FileOutputStream、ThreadPoolExecutor和java.sql.Connection，都有作为安全网的Finalizer机制。

第二种合理使用Cleaner机制的方法与**本地对等类**（native peers）有关。本地对等类是一个由普通对象委托的本地(非Java)对象。由于本地对等类不是普通的 Java对象，所以垃圾收集器并不知道它，当它的Java对等对象被回收时，本地对等类也不会回收。假设性能是可以接受的，并且本地对等类没有关键的资源，那么Finalizer和Cleaner机制可能是这项任务的合适的工具。但如果性能是不可接受的，或者本地对等类持有必须迅速回收的资源，那么类应该有一个close方法，正如前面所述。

Cleaner机制使用起来有点棘手。下面是演示该功能的一个简单的Room类。假设Room对象必须在被回收前清理干净。Room类实现AutoCloseable接口；它的自动清理安全网使用的是一个Cleaner机制，这仅仅是一个实现细节。与Finalizer机制不同，Cleaner机制不污染一个类的公共API：

```
// An autocloseable class using a cleaner as a safety net
public class Room implements AutoCloseable {
    private static final Cleaner cleaner = Cleaner.create();

    // Resource that requires cleaning. Must not refer to Room!
    private static class State implements Runnable {
        int numJunkPiles; // Number of junk piles in this room

        State(int numJunkPiles) {
            this.numJunkPiles = numJunkPiles;
        }

        // Invoked by close method or cleaner
        @Override
        public void run() {
            System.out.println("Cleaning room");
            numJunkPiles = 0;
        }
    }

    // The state of this room, shared with our cleanable
    private final State state;

    // Our cleanable. Cleans the room when it’s eligible for gc
    private final Cleaner.Cleanable cleanable;

    public Room(int numJunkPiles) {
        state = new State(numJunkPiles);
        cleanable = cleaner.register(this, state);
    }

    @Override
    public void close() {
        cleanable.clean();
    }
}
```

静态内部State类拥有Cleaner机制清理房间所需的资源。在这里，它仅仅包含numJunkPiles属性，它代表混乱房间的数量。更实际地说，它可能是一个final修饰的long类型的指向本地对等类的指针。 State类实现了Runnable接口，其run方法最多只能调用一次，只能被我们在Room构造方法中用Cleaner机制注册State实例时得到的Cleanable调用。

对run方法的调用通过以下两种方法触发：通常，通过调用Room的close方法内调用Cleanable的clean方法来触发。如果在Room实例有资格进行垃圾回收的时候客户端没有调用close方法，那么Cleaner机制将（希望）调用State的run方法。

一个State实例不引用它的Room实例是非常重要的。如果它引用了，则创建了一个循环，阻止了Room实例成为垃圾收集的资格(以及自动清除)。因此，State必须是静态的嵌内部类，因为非静态内部类包含对其宿主类的实例的引用。同样，使用lambda表达式也是不明智的，因为它们很容易获取对宿主类对象的引用。

就像我们之前说的，Room的Cleaner机制仅仅被用作一个安全网。如果客户将所有Room的实例放在try-with-resource块中，则永远不需要自动清理。行为良好的客户端如下所示：


```
public class Adult {
    public static void main(String[] args) {
        try (Room myRoom = new Room(7)) {
            System.out.println("Goodbye");
        }
    }
}
```

正如你所预料的，运行Adult程序会打印Goodbye字符串，随后打印Cleaning room字符串。但是如果时不合规矩的程序，它从来不清理它的房间会是什么样的?


```
public class Teenager {
    public static void main(String[] args) {
        new Room(99);
        System.out.println("Peace out");
    }
}
```

你可能期望它打印出Peace out，然后打印Cleaning room字符串，但在我的机器上，它从不打印Cleaning room字符串；仅仅是程序退出了。 这是我们之前谈到的不可预见性。 Cleaner机制的规范说：“System.exit方法期间的清理行为是特定于实现的。 不保证清理行为是否被调用。”虽然规范没有说明，但对于正常的程序退出也是如此。 在我的机器上，将System.gc()方法添加到Teenager类的main方法足以让程序退出之前打印Cleaning room，但不能保证在你的机器上会看到相同的行为。

总之，除了作为一个安全网或者终止非关键的本地资源，不要使用Cleaner机制，或者是在Java 9发布之前的finalizers机制。即使是这样，也要当心不确定性和性能影响。


---

# 未完待续.....
