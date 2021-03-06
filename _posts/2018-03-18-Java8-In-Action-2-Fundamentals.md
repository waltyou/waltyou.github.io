---
layout: post
title: 《Java 8 in Action》学习日志（二）：基础知识
date: 2018-3-18 22:15:04
author: admin
comments: true
categories: [Java]
tags: [Java,Java8]
---


这是《Java 8 In Action》书中的第一部分，着重介绍了Java8的改善点与优势，以及两个重点概念：行为参数化以及lambda。
行为参数化可以将函数作为参数传递，避免咯嗦。因此，匿名函数lambda顺应而生。

<!-- more -->

学习资料主要参考： 《Java 8 In Action》、《Java 8实战》，以及其源码：[Java8 In Action](https://github.com/java8/Java8InAction)

---

* 目录
{:toc}
---

# 为何关心java8

1. 新概念和新功能，有助于写出高效简约的代码
2. 现有的Java编程实践并不能很好地利用多核处理器
3. 借鉴函数式语言的其他优点: 处理null和模式匹配

---
# 行为参数化

## 1. Why

应对不断变化的需求，避免啰嗦，而且不打破DRY（Don’t Repeat Yourself）规则。

## 2. What

简单讲：把方法（你的代码）作为参数传递给另一个方法。

复杂讲： 让方法接受多种行为（或战略）作为参数，并在内部使用，来完成不同的行为。

## 3. How

Example 1：用一个Comparator排序Apple，使用Java 8中List默认的sort方法。

```java
// java.util.Comparator
public interface Comparator<T> {
    public int compare(T o1, T o2);
}
// 匿名类写法
inventory.sort(new Comparator<Apple>() {
    public int compare(Apple a1, Apple a2){
        return a1.getWeight().compareTo(a2.getWeight());
    }
});
// lambda写法
inventory.sort(
    (Apple a1, Apple a2) ->
    a1.getWeight().compareTo(a2.getWeight()));
```

Example 2：用Runnable执行代码块。

```java
// java.lang.Runnable
public interface Runnable{
    public void run();
}
// 匿名类写法
Thread t = new Thread(new Runnable() {
    public void run(){
        System.out.println("Hello world");
    }
});
// lambda写法
Thread t = new Thread(() -> System.out.println("Hello world"));
```

Example 3：GUI事件处理。

```java
Button button = new Button("Send");
// 匿名类写法
button.setOnAction(new EventHandler<ActionEvent>() {
    public void handle(ActionEvent event) {
        label.setText("Sent!!");
    }
});

// lambda写法
button.setOnAction((ActionEvent event) -> label.setText("Sent!!"));
```
---

# 匿名函数lambda

## 1. Why

匿名类太啰嗦

## 2. What

简单讲：匿名函数。

复杂讲：简洁地表示可传递的匿名函数的一种方式：它没有名称，但它有参数列表、函数主体、返回类型，可能还有一个可以抛出的异常列表。

特点：
- 匿名——不需要明确的名称
- 函数——不需要属于特定的类，但和方法一样，有参数列表、函数主体、返回类型，还可能有可以抛出的异常列表。
- 传递——作为参数传递或存储在变量中。
- 简洁——无需像匿名类那样写很多模板代码。

关键词：
参数列表 + 箭头 + 主体

基本写法
```java
(parameters) -> expression
```
当主体中出现控制流语句，如return等，要使此Lambda有效，需要使花括号
```java
(parameters) -> { statements; }
```

## 3. How

使用lambda之前，需要了解两个概念：**函数式接口** 和**函数描述符**。

### 函数式接口

只定义一个抽象方法的接口。如：Runable和Comparator。

lambda其实可以看作是函数式接口的实例。

**@FunctionalInterface**：标注用于表示该接口会设计成一个函数式接口。

Java 8 提供了一些新的函数式接口， 位置：java.util.function
1. Predicate.test: (T) -> boolean
2. Consumer.accept： (T) -> void
3. Function.apply： (T) -> R

### 函数描述符

函数式接口的抽象方法的签名基本上就是Lambda表达式的签名。如下：
```java
() -> void
(Apple) -> int
(Apple, Apple) -> boolean
```

## 4. 实现细节

###  类型检查

上下文中Lambda表达式需要的类型称为目标类型
1. 同样的Lambda，不同的函数式接口
2. 特殊的void兼容规则：如果一个Lambda的主体是一个语句表达式 它就和一个返回void的函数描述符兼容。

###  类型推断

编译器可以了解Lambda表达式的参数类型，这样就可以在Lambda语法中省去标注参数类型。

###  使用局部变量：
```java
int portNumber = 1337; 
Runnable r = () -> System.out.println(portNumber); 
```

注意：
Lambda可以没有限制地捕获（也就是在其主体中引用）实例变量和静态变量。但局部变量必须显式声明为final，或事实上是final。

原因：
1）局部变量保存在栈上，并且隐式表示它们仅限于其所在线程，如果允许捕获可改变的局部变量，就会引发造成线程不安全新的可能性；
2）不鼓励你使用改变外部变量的典型命令式编程模式

### 方法引用（method reference）

目标引用放在分隔符 :: 前, 方法的名称放在后面。
```java
inventory.sort(comparing(Apple::getWeight));
```

方法引用主要有三类:
1. 指向静态方法的方法引用: Integer::parseInt
2. 指向任意类型实例方法的方法引用: String::length
3. 指向现有对象的实例方法的方法引用: expensiveTransaction::getValue

```java
//改写
Function<String, Integer> stringToInteger = (String s) -> Integer.parseInt(s);
Function<String, Integer> stringToInteger = Integer::parseInt;

BiPredicate<List<String>, String> contains = (list, element) -> list.contains(element);
BiPredicate<List<String>, String> contains = List::contains;
```

构造函数引用： 
```java
Supplier<Apple> c1 = Apple::new;
Apple a1 = c1.get();

Function<Integer, Apple> c2 = Apple::new;
Apple a2 = c2.apply(110);
```

### 复合Lambda表达式 (因为引入了默认方法)

#### 1. 比较器复合

```java
Comparator<Apple> c = Comparator.comparing(Apple::getWeight);
// 逆序
inventory.sort(comparing(Apple::getWeight).reversed());
// 比较器链
inventory.sort(comparing(Apple::getWeight)
    .reversed()
    .thenComparing(Apple::getCountry))
```

#### 2. 谓词复合

negate、and和or

```java
//取非
Predicate<Apple> notRedApple = redApple.negate();
//and操作
Predicate<Apple> redAndHeavyApple = 
redApple.and(a -> a.getWeight() > 150);
//and + or操作
Predicate<Apple> redAndHeavyAppleOrGreen = 
redApple.and(a -> a.getWeight() > 150).or(a -> "green".equals(a.getColor()));
```

从左向右确定优先级.

#### 3. 函数复合
Function提供了andThen(), compose()

```java
Function<Integer, Integer> f = x -> x + 1; 
Function<Integer, Integer> g = x -> x * 2; 
// expect: (2 + 1) * 2 = 4
// f(g(x))
System.out.println(f.andThen(g).apply(1));
// expect: 1 * 2 + 1 = 3
// g(f(x))
System.out.println(f.compose(g).apply(1));
```
*复合Lambda表达式可以用来创建各种转型流水线。*
