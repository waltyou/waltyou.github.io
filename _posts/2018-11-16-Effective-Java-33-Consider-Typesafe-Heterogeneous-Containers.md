---
layout: post
title: 《Effective Java》学习日志（四）33：优先考虑类型安全的异构容器
date: 2018-11-16 10:50:04
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

泛型的常见用法包括集合，如Set<E>和Map<K，V>和单个元素容器，如ThreadLocal<T>和AtomicReference<T>。 
在所有这些用途中，它都是参数化的容器。 
这限制了每个容器只能有固定数量的类型参数。 
通常这正是你想要的。 

一个Set有单一的类型参数，表示它的元素类型; 一个Map有两个，代表它的键和值的类型；等等。

然而有时候，你需要更多的灵活性。 例如，数据库一行记录可以具有任意多列，并且能够以类型安全的方式访问它们是很好的。 
幸运的是，有一个简单的方法可以达到这个效果。 
这个想法是参数化键（key）而不是容器。 然后将参数化的键提交给容器以插入或检索值。 
泛型类型系统用于保证值的类型与其键一致。

# 简单示例

作为这种方法的一个简单示例，请考虑一个Favorites类，它允许其客户端保存和检索任意多种类型的favorite实例。 

该类型的Class对象将扮演参数化键的一部分。其原因是这Class类是泛型的。 
类的类型从字面上来说不是简单的Class，而是Class<T>。 
例如，String.class的类型为Class<String>，Integer.class的类型为Class<Integer>。 
当在方法中传递字面类传递编译时和运行时类型信息时，它被称为类型令牌（type token）[Bracha04]。

Favorites类的API很简单。 
它看起来就像一个简单Map类，除了该键是参数化的以外。 
客户端在设置和获取favorites实例时呈现一个Class对象。 这里是API：

    // Typesafe heterogeneous container pattern - API
    public class Favorites {
        public<T> void putFavorite(Class<T> type, T instance);
        public<T> T getFavorite(Class<T> type);
    }
    
下面是一个演示Favorites类，保存，检索和打印喜欢的String，Integer和Class实例：

    // Typesafe heterogeneous container pattern - client
    public static void main(String[] args) {
        Favorites f = new Favorites();
        f.putFavorite(String.class, "Java");
        f.putFavorite(Integer.class, 0xcafebabe);
        f.putFavorite(Class.class, Favorites.class);
        
        String favoriteString = f.getFavorite(String.class);
        int favoriteInteger = f.getFavorite(Integer.class);
        Class<?> favoriteClass = f.getFavorite(Class.class);
        System.out.printf("%s %x %s%n", favoriteString,
                    favoriteInteger, favoriteClass.getName());
    }
    
正如你所期望的，这个程序打印Java cafebabe Favorites。 

请注意，顺便说一下，Java的printf方法与C语言的不同之处在于，应该使用％n，而在C中使用\n。 
％n生成适用的特定于平台的行分隔符，该分隔符在很多但不是所有平台上都是\n。

Favorites实例是类型安全的：当你请求一个字符串时它永远不会返回一个整数。 
它也是异构的：与普通Map不同，所有的键都是不同的类型。 因此，我们将Favorites称为**类型安全异构容器**（typesafe heterogeneous container.）。

Favorites的实现非常小巧。 这是完整的代码：

    // Typesafe heterogeneous container pattern - implementation
    public class Favorites {
        private Map<Class<?>, Object> favorites = new HashMap<>();
    
        public<T> void putFavorite(Class<T> type, T instance) {
            favorites.put(Objects.requireNonNull(type), instance);
        }
    
        public<T> T getFavorite(Class<T> type) {
            return type.cast(favorites.get(type));
        }
    }
    
这里有一些微妙的事情发生。 每个Favorites实例都由一个名为favorites私有的Map<Class<?>, Object>来支持。 
你可能认为无法将任何内容放入此Map中，因为这是无限定的通配符类型，但事实恰恰相反。 
需要注意的是通配符类型是嵌套的：它不是通配符类型的Map类型，而是键的类型。 
这意味着每个键都可以有不同的参数化类型：一个可以是Class<String>，下一个Class<Integer>等等。 这就是异构的由来。

接下来要注意的是，favorites的Map的值类型只是Object。 
换句话说，Map不保证键和值之间的类型关系，即每个值都是由其键表示的类型。 
事实上，Java的类型系统并不足以表达这一点。 但是我们知道这是真的，并在检索一个favorite时利用了这点。

putFavorite实现很简单：只需将给定的Class对象映射到给定的favorites的实例即可。 
如上所述，这丢弃了键和值之间的“类型联系（type linkage）”；无法知道这个值是不是键的一个实例。 
但没关系，因为getFavorites方法可以并且确实重新建立这种关联。

getFavorite的实现比putFavorite更复杂。 
首先，它从favorites Map中获取与给定Class对象相对应的值。 
这是返回的正确对象引用，但它具有错误的编译时类型：它是Object（favorites map的值类型），我们需要返回类型T。

因此，getFavorite实现动态地将对象引用转换为Class对象表示的类型，使用Class的cast方法。

cast方法是Java的cast操作符的动态模拟。它只是检查它的参数是否由Class对象表示的类型的实例。
如果是，它返回参数；否则会抛出ClassCastException异常。

我们知道，假设客户端代码能够干净地编译，getFavorite中的强制转换不会抛出ClassCastException异常。 
也就是说，favorites map中的值始终与其键的类型相匹配。

那么这个cast方法为我们做了什么，因为它只是返回它的参数？ cast的签名充分利用了Class类是泛型的事实。 它的返回类型是Class对象的类型参数：

    public class Class<T> {
        T cast(Object obj);
    }
    
这正是getFavorite方法所需要的。 这正是确保Favorites类型安全，而不用求助一个未经检查的强制转换的T类型。


# 两个限制

Favorites类有两个限制值得注意。 

## 一

首先，恶意客户可以通过使用原始形式的Class对象，轻松破坏Favorites实例的类型安全。 
但生成的客户端代码在编译时会生成未经检查的警告。 
这与正常的集合实现（如HashSet和HashMap）没有什么不同。 
通过使用原始类型HashSet（条目 26），可以轻松地将字符串放入HashSet<Integer>中。 
也就是说，如果你愿意为此付出一点代价，就可以拥有运行时类型安全性。 
确保Favorites永远不违反类型不变的方法是，使putFavorite方法检查该实例是否由type表示类型的实例，并且我们已经知道如何执行此操作。
只需使用动态转换：

    // Achieving runtime type safety with a dynamic cast
    public<T> void putFavorite(Class<T> type, T instance) {
        favorites.put(type, type.cast(instance));
    }
    
java.util.Collections中有一些集合包装类，可以发挥相同的诀窍。 

它们被称为checkedSet，checkedList，checkedMap等等。 
他们的静态工厂除了一个集合（或Map）之外还有一个Class对象（或两个）。 
静态工厂是泛型方法，确保Class对象和集合的编译时类型匹配。 包装类为它们包装的集合添加了具体化。 
例如，如果有人试图将Coin放入你的Collection<Stamp>中，则包装类在运行时会抛出ClassCastException。 
这些包装类对于追踪在混合了泛型和原始类型的应用程序中添加不正确类型的元素到集合的客户端代码很有用。

## 二

Favorites类的第二个限制是它不能用于不可具体化的（non-reifiable）类型（条目 28）。 
换句话说，你可以保存你最喜欢的String或String []，但不能保存List<String>。 

如果你尝试保存你最喜欢的List<String>，程序将不能编译。 
原因是无法获取List<String>的Class对象。 List<String>.class是语法错误，也是一件好事。 
List<String>和List<Integer>共享一个Class对象，即List.class。 
如果“字面类型（type literals）”List<String> .class和List<Integer>.class合法并返回相同的对象引用，那么它会对Favorites对象的内部造成严重破坏。 

对于这种限制，没有完全令人满意的解决方法。

Favorites使用的类型令牌( type tokens)是无限制的：getFavorite和putFavorite接受任何Class对象。 
有时你可能需要限制可传递给方法的类型。 
这可以通过一个有限定的类型令牌来实现，该令牌只是一个类型令牌，它使用限定的类型参数（条目 30）或限定的通配符（条目 31）来放置可以表示的类型的边界。

注解API（条目 39）广泛使用限定类型的令牌。 例如，以下是在运行时读取注解的方法。 

此方法来自AnnotatedElement接口，该接口由表示类，方法，属性和其他程序元素的反射类型实现：

    public<T extends Annotation>
        T getAnnotation(Class<T> annotationType);
        
参数annotationType是表示注解类型的限定类型令牌。 
该方法返回该类型的元素的注解（如果它有一个）；如果没有，则返回null。 本质上，注解元素是一个类型安全的异构容器，其键是注解类型。

假设有一个Class<?>类型的对象，并且想要将它传递给需要限定类型令牌（如getAnnotation）的方法。 
可以将对象转换为Class<? extends Annotation>，但是这个转换没有被检查，所以它会产生一个编译时警告（条目 27）。 
幸运的是，Class类提供了一种安全（动态）执行这种类型转换的实例方法。 
该方法被称为asSubclass，并且它转换所调用的Class对象来表示由其参数表示的类的子类。 
如果转换成功，该方法返回它的参数；如果失败，则抛出ClassCastException异常。

以下是如何使用asSubclass方法在编译时读取类型未知的注解。 此方法编译时没有错误或警告：

    // Use of asSubclass to safely cast to a bounded type token
    static Annotation getAnnotation(AnnotatedElement element,
                                    String annotationTypeName) {
        Class<?> annotationType = null; // Unbounded type token
        try {
            annotationType = Class.forName(annotationTypeName);
        } catch (Exception ex) {
            throw new IllegalArgumentException(ex);
        }
        return element.getAnnotation(
            annotationType.asSubclass(Annotation.class));
    }
    
总之，泛型API的通常用法（以集合API为例）限制了每个容器的固定数量的类型参数。 
你可以通过将类型参数放在键上而不是容器上来解决此限制。 
可以使用Class对象作为此类型安全异构容器的键。 
以这种方式使用的Class对象称为类型令牌。 
也可以使用自定义键类型。 
例如，可以有一个表示数据库行（容器）的DatabaseRow类型和一个泛型类型Column<T>作为其键。

