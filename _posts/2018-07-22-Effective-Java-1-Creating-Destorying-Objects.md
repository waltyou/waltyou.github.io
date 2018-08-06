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

# 未完待续.....
