---
layout: post
title: 《Java 8 in Action》学习日志（四）：高效Java8编程
date: 2018-3-28 22:15:04
author: admin
comments: true
categories: [Java]
tags: [Java,Java8]
---



<!-- more -->

学习资料主要参考： 《Java 8 In Action》、《Java 8实战》，以及其源码：[Java8 In Action](https://github.com/java8/Java8InAction)

---
# 目录
{:.no_toc}

* 目录
{:toc}
---

# 重构、测试和调试

## 1. 为改善可读性和灵活性重构代码

### 可读性

1. 从匿名类到Lambda表达式的转换

    匿名类是极其繁琐且容易出错的。
    
    例子：
    ```java
    Runnable r1 = new Runnable(){ 
        public void run(){ 
            System.out.println("Hello"); 
        } 
    }; 
    Runnable r2 = () -> System.out.println("Hello");
    ```
    注意点：

    1. 匿名类和Lambda表达式中的this和super的含义是不同的。在匿名类中，this代表的是类自身，但是在Lambda中，它代表的是包含类
    2. 匿名类可以屏蔽包含类的变量，而Lambda表达式不能
    3. 在涉及重载的上下文里，将匿名类转换为Lambda表达式可能导致最终的代码更加晦涩。（可使用显式的类型转换解决因重载而无法确定方法的问题）
    
2. 从Lambda表达式到方法引用的转换

    将Lambda表达式的内容抽取到一个单独的方法中，将其作为参数传递。既清晰，又可重用。
    ```java
    Map<CaloricLevel, List<Dish>> dishesByCaloricLevel = 
        menu.stream().collect(groupingBy(Dish::getCaloricLevel));
    ```
    还应该尽量考虑使用静态辅助方法（如comparing、maxBy），
    ```java
    //before
    inventory.sort( 
        (Apple a1, Apple a2) -> a1.getWeight().compareTo(a2.getWeight())); 
    //after
    inventory.sort(comparing(Apple::getWeight));
    ```
    通用的归约操作（如sum、maximum）。
    ```java
    // before
    int totalCalories = 
        menu.stream().map(Dish::getCalories) 
            .reduce(0, (c1, c2) -> c1 + c2);
    // after
    int totalCalories = menu.stream().collect(summingInt(Dish::getCalories));
    ```
    
3. 从命令式的数据处理切换到Stream
   
    将所有使用迭代器这种数据处理模式处理集合的代码都转换成Stream API的方
式。 Stream API能更清晰地表达数据处理管道的意图。
    
    例子：
    ```java
    // before
    List<String> dishNames = new ArrayList<>(); 
    for(Dish dish: menu){ 
        if(dish.getCalories() > 300){ 
            dishNames.add(dish.getName()); 
        } 
    }
    // after
    menu.parallelStream()
        .filter(dish -> dish.getCalories() > 300)
        .map(Dish::getName)
        .collect(tList());
    ```
    注意点：需要考虑控制流语句，比如break、continue、return，并选择使用恰当的流操作。
    
### 灵活性

要使用lambda，必然要采用函数接口。

引入函数接口的两种模式：

1. 有条件的延迟执行

    控制语句被混杂在业务逻辑代码之中，如：安全性检查以及日志输出。
    
    如果需要在代码中频繁的判断一个对象的状态，而只是为了**传递参数**、**调用该对象的某个方法**，可以实现一个新的方法，以lambda作为参数，**在新方法中进行判断**。这样子，代码会更易读，结构更清晰，封装性也更好。
    
2. 环绕执行
    
    定义：业务代码千差万别，但是准备与清理阶段一致。

    用法：将业务代码用lambda重写。
    
    优点：可以重用准备、清理阶段的代码。
    ```java
    String oneLine = processFile((BufferedReader b) -> b.readLine()); 
    String twoLines = processFile((BufferedReader b) -> b.readLine() + b.readLine()); 
    //相同的准备、清理阶段
    public static String processFile(BufferedReaderProcessor p) throws IOException { 
        try(BufferedReader br = new BufferedReader(new FileReader("java8inaction/chap8/data.txt"))){ 
            return p.process(br); 
        } 
    } 
    //定义接口
    public interface BufferedReaderProcessor{ 
        String process(BufferedReader b) throws IOException; 
    }
    ```

## 2. 使用Lambda重构面向对象的设计模式

### 策略模式

策略模式代表了解决一类算法的通用解决方案,你可以在运行时选择使用哪种方案。

直接将策略接口当作函数接口，使用lambda代替繁杂的策略接口实现类。

#### 例子：验证字符串是否合法

1. 准备工作

策略接口

```java
public interface ValidationStrategy {
    boolean execute(String s);
}
```
调用类
```java
public class Validator{
    private final ValidationStrategy strategy;

    public Validator(ValidationStrategy v){
        this.strategy = v;
    }
    
    public boolean validate(String s){
        return strategy.execute(s);
    }
}
```
2. 接口实现类的实现与调用
 
实现
```java
public class IsAllLowerCase implements ValidationStrategy {
    public boolean execute(String s){
        return s.matches("[a-z]+");
    }
}
public class IsNumeric implements ValidationStrategy {
    public boolean execute(String s){
        return s.matches("\\d+");
    }
}
```
调用
```java
Validator numericValidator = new Validator(new IsNumeric());
boolean b1 = numericValidator.validate("aaaa");
Validator lowerCaseValidator = new Validator(new IsAllLowerCase ());
boolean b2 = lowerCaseValidator.validate("bbbb");
```
3. lambda实现(无需创建接口实现类)
```java
Validator numericValidator =
    new Validator((String s) -> s.matches("[a-z]+"));
boolean b1 = numericValidator.validate("aaaa");

Validator lowerCaseValidator =
    new Validator((String s) -> s.matches("\\d+"));
boolean b2 = lowerCaseValidator.validate("bbbb");
```
### 模板方法
如果你需要采用某个算法的框架,同时又希望有一定的灵活度,能对它的某些部分进行改进,那么采用模板方法设计模式是比较通用的方案。

1.改写前
```java
abstract class OnlineBanking {
    public void processCustomer(int id){
        Customer c = Database.getCustomerWithId(id);
        makeCustomerHappy(c);
    }
    
    abstract void makeCustomerHappy(Customer c);
}
```
这个抽象类构建了一个模板，所有的子类都要实现makeCustomerHappy，来面对差异化的需求。
2. 改写后
首先添加一个重载的新方法，它多传入一个Consumer接口作为参数
```java
public void processCustomer(int id, Consumer<Customer> makeCustomerHappy){
    Customer c = Database.getCustomerWithId(id);
    makeCustomerHappy.accept(c);
}
```
传入lambda
```java
new OnlineBankingLambda().processCustomer(1337, (Customer c) ->
    System.out.println("Hello " + c.getName());
```
### 观察者模式
某些事件发生时(比如状态转变),如果一个对象(通常我们称之为主题Subject)需要自动地通知其他多个对象(称为观察者Observer)。

1. 准备
```java
interface Subject{
    void registerObserver(Observer o);
    void notifyObservers(String tweet);
}

class Feed implements Subject{
    private final List<Observer> observers = new ArrayList<>();
    public void registerObserver(Observer o) {
        this.observers.add(o);
    }
    public void notifyObservers(String tweet) {
        observers.forEach(o -> o.notify(tweet));
    }
}

interface Observer {
    void notify(String tweet);
}
```
2. 改写前

实现三个观察者 

```java
class NYTimes implements Observer{
    public void notify(String tweet) {
        if(tweet != null && tweet.contains("money")){
            System.out.println("Breaking news in NY! " + tweet);
        }
    }
}
class Guardian implements Observer{
    public void notify(String tweet) {
        if(tweet != null && tweet.contains("queen")){
            System.out.println("Yet another news in London... " + tweet);
        }
    }
}
class LeMonde implements Observer{
    public void notify(String tweet) {
        if(tweet != null && tweet.contains("wine")){
            System.out.println("Today cheese, wine and news! " + tweet);
        }
    }
}
```
调用
```java
Feed f = new Feed();
f.registerObserver(new NYTimes());
f.registerObserver(new Guardian());
f.registerObserver(new LeMonde());
f.notifyObservers("The queen said her favourite book is Java 8 in Action!");
```
3. 改写后

无需显式地实例化三个观察者对象,直接传递Lambda表达式表示需要执行的行为即可。
```java
f.registerObserver((String tweet) -> {
    if(tweet != null && tweet.contains("money")){
        System.out.println("Breaking news in NY! " + tweet);
    }
});
f.registerObserver((String tweet) -> {
    if(tweet != null && tweet.contains("queen")){
        System.out.println("Yet another news in London... " + tweet);
    }
});
```
**注意**：如果观察者的逻辑十分复杂,或者持有了状态,抑或定义了多个方法,诸如此
类。在这些情形下,还是应该继续使用类的方式。


### 责任链模式

责任链模式是一种创建处理对象序列(比如操作序列)的通用方案。

1. 改写前

构建抽象类
```java
public abstract class ProcessingObject<T> {
    protected ProcessingObject<T> successor;
    public void setSuccessor(ProcessingObject<T> successor){
        this.successor = successor;
    }
    public T handle(T input){
        T r = handleWork(input);
        if(successor != null){
            return successor.handle(r);
        }
        return r;
    }
    abstract protected T handleWork(T input);
}
```
实现类
```java
public class HeaderTextProcessing extends ProcessingObject<String> {
    public String handleWork(String text){
        return "From Raoul, Mario and Alan: " + text;
    }
}
public class SpellCheckerProcessing extends ProcessingObject<String> {
    public String handleWork(String text){
        return text.replaceAll("labda", "lambda");
    }
}
```
调用
```java
ProcessingObject<String> p1 = new HeaderTextProcessing();
ProcessingObject<String> p2 = new SpellCheckerProcessing();
p1.setSuccessor(p2);
String result = p1.handle("Aren't labdas really sexy?!!");
System.out.println(result);
```
2. 改写后

可以直接使用UnaryOperator
```java
UnaryOperator<String> headerProcessing =
    (String text) -> "From Raoul, Mario and Alan: " + text;
UnaryOperator<String> spellCheckerProcessing =
    (String text) -> text.replaceAll("labda", "lambda");
Function<String, String> pipeline =
    headerProcessing.andThen(spellCheckerProcessing);
String result = pipeline.apply("Aren't labdas really sexy?!!")
```

### 工厂模式

使用工厂模式,你无需向客户暴露实例化的逻辑就能完成对象的创建。

1. 改写前

```java
public class ProductFactory {
    public static Product createProduct(String name){
        switch(name){
            case "loan": return new Loan();
            case "stock": return new Stock();
            case "bond": return new Bond();
            default: throw new RuntimeException("No such product " + name);
        }
    }
}
```
2. 改写后
 
```java
public class ProductFactory {
    final static Map<String, Supplier<Product>> map = new HashMap<>();
    static {
        map.put("loan", Loan::new);
        map.put("stock", Stock::new);
        map.put("bond", Bond::new);
    }
    
    public static Product createProduct(String name){
        Supplier<Product> p = map.get(name);
        if(p != null) return p.get();
        throw new IllegalArgumentException("No such product " + name);
    }
}
```

## 3. 测试 Lambda 表达式

测试lambda表达式的正确性时，没必要对Lambda表达式本身进行测试，只需要关注其结果是否正确即可。

## 4. 调试

### 查看栈跟踪

由于Lambda表达式没有名字, 编译器只能为它们指定一个名字，比如lambda$main$0。

即使你使用了方法引用,还是有可能出现栈无法显示你使用的方法名的情况。

如果方法引用指向的是同一个类中声明的方法,那么它的名称是可以在栈跟踪中显示的。

涉及Lambda表达式的栈跟踪可能非常难理解。这是Java编译器未来版本可以改进的一个方面。

### 输出日志

流提供的 peek 方法在分析Stream流水线时,能将中间变量的值输出到日志中,是非常有用的工具。

peek 的设计初衷就是在流的每个元素恢复运行之前,插入执行一个动作。但是它不像 forEach 那样恢复整个流的运行,而是在一个元素上完成操作之后,它只会将操作顺承到流水线中的下一个操作。

例子
```java
List<Integer> numbers = Arrays.asList(2, 3, 4, 5);

List<Integer> result =
    numbers.stream()
    .peek(x -> System.out.println("from stream: " + x))
    .map(x -> x + 17)
    .peek(x -> System.out.println("after map: " + x))
    .filter(x -> x % 2 == 0)
    .peek(x -> System.out.println("after filter: " + x))
    .limit(3)
    .peek(x -> System.out.println("after limit: " + x))
    .collect(toList());
```
输出
```
from stream: 2
after map: 19
from stream: 3
after map: 20
after filter: 20
after limit: 20
from stream: 4
after map: 21
from stream: 5
after map: 22
after filter: 22
after limit: 22
```

---
# 默认方法

## 1. API的演进

当一个API接口发布后，它的后续演进会给用户带来一系列问题，比如在原先接口中添加一个新的方法，那么所有用户都需要在接口的实现类中添加这个方法的实现，这是大部分用户不想看到的。

## 2. 概述默认方法

默认方法由 default 修饰符修饰, 并像类中声明的其他方法一样包含方法体，它会作为接口的一部分由实现类继承。

由于引入了默认方法，就能够以兼容的方式演进库函数。

**Java8中抽象类与抽象接口区别**
1. 一个类只能继承单个抽象类，但是可以实现多个接口
2. 抽象类中可以有实例变量，接口中不能。

## 3. 使用

### 可选方法

类继承了接口，但是对某些接口方法的实现留白，所以代码中会存在很多无用的代码。那么有了默认方法后，就可以对这些方法提供一个默认实现，这样子实体类就无需写上一个空方法。

### 行为的多继承

我们都知道一个类是可以实现多接口的，那么在引入默认方法后，在不同接口中都可以存在一些默认方法。这样子，我们通过组合接口，就可以最大程度地实现代码复用和行为组合。

## 4. 如何解决冲突

如果一个类实现多个接口，而多个接口中有重名的默认方法时，会发生什么事呢？

看看下面这段代码会输出什么？
```java
public interface A { 
    default void hello() { 
        System.out.println("Hello from A"); 
    } 
} 
public interface B extends A { 
    default void hello() { 
        System.out.println("Hello from B"); 
    } 
} 
public class C implements B, A { 
    public static void main(String... args) { 
        new C().hello(); 
    } 
}
```

### 解决问题的三条规则

1. 类中的方法优先级最高
2. 如果无法依据第一条进行判断，那么子接口的优先级更高：函数签名相同时，优先选择拥有最具体实现的默认方法的接口，即如果B继承了A，那么B就比A更加具体。
3. 最后，如果还是无法判断，继承了多个接口的类必须通过**显式覆盖**和**调用期望**的方法，显式地选择使用哪一个默认方法的实现。


### 应用前两条规则
那么回过头来，上面那段代码输出结果应该是什么呢？

答案是： Hello from B。

因为按照规则(2)，应该选择的是提供了最具体实现的默认方法的接口。由于B比A更具体，所以应该选择B的hello方法。


继续看，如果是下列代码输出会是什么呢？
```java
public class D implements A{ } 

public class C extends D implements B, A { 
    public static void main(String... args) { 
        new C().hello(); 
    } 
}
```
答案依然是：Hello from B。

依据规则(1)，类中声明的方法具有更高的优先级。D并未覆盖hello方法，可是它实现了接口A。所以它就拥有了接口A的默认方法。规则(2)说如果类或者父类没有对应的方法，那么就应该选择提供了最具体实现的接口中的方法。因此，编译器会在接口A和接口B的hello方法之间做选择。由于B更加具体，所以程序会再次打印输出“Hello from B”。

### 冲突及如何显式地消除歧义

```java
public interface A { 
    void hello() { 
        System.out.println("Hello from A"); 
    } 
} 
public interface B { 
    void hello() { 
        System.out.println("Hello from B"); 
    } 
} 
public class C implements B, A { } 
```
对于以上代码，Java编译器会抛出一个编译错误，因为它无法判断哪一个方法更合适：“Error: class Cinherits unrelated defaults for hello() from types Band A.” 

#### 冲突的解决

Java 8中引入了一种新的语法X.super.m(…)，意思即为你希望调用的父接口X中的m方法。
```java
public class C implements B, A { 
    void hello(){ 
        B.super.hello(); 
    } 
}
```

### 菱形继承问题
```java
public interface A{ 
    default void hello(){ 
        System.out.println("Hello from A"); 
    } 
} 

public interface B extends A { } 

public interface C extends A { } 

public class D implements B, C { 
    public static void main(String... args) { 
        new D().hello(); 
    } 
}
```
B和C继承了A，也同时继承了A的默认方法，所以会打印：“Hello from A”。

想象以下情况：
1. 如果B中提供相同签名的默认方法hello
2. 如果B和C都使用相同的函数签名声明了hello方法
3. 如果在C接口中添加一个抽象的hello方法（这次添加的不是一个默认方法）

分析结果如下：
1. B中提供了更加具体的实现，所以会调用B中的方法
2. B和C优先级相同，需要显式调用
3. 需要在D中实现hello方法，否则无法通过编译


---
# 用Optional代替null

nullPointExcption 绝对是java成员常见的异常。所以为了程序的正常运行，我们不得不在使用某个值之前对它进行非null的check。但是这个样子的代码会变得臃肿，丧失易读性，而且难以维护。

## 1. null 带来的种种问题

1. 它是错误之源
2. 它会使你的代码膨胀
3. 它自身是毫无意义的
4. 它破坏了Java的哲学（避免让程序员意识到指针的存在，唯一的例外是: null 指针）
5. 它在Java的类型系统上开了个口子。 null并不属于任何类型。

## 2. 使用Optional
汲取Haskell和Scala的灵感，Java 8中引入了一个新的类java.util.Optional<T>。

```java
public class Person { 
    private Optional<Car> car; 
    public Optional<Car> getCar() { return car; } 
}
public class Car { 
    private Optional<Insurance> insurance; 
    public Optional<Insurance> getInsurance() { return insurance; } 
} 
public class Insurance { 
    private String name; 
    public String getName() { return name; } 
} 
```

注意：引入Optional类的意图并非要消除每一个null引用。与此相反，它的目标是帮助你更好地设计出普适的API，让程序员看到方法签名，就能了解它是否接受一个Optional的值。这种强制会让你更积极地将变量从Optional中解包出来，直面缺失的变量值。

## 3. 应用Optional的几种模式

### 创建Optional对象

```java
// 声明一个空的Optional
Optional<Car> optCar = Optional.empty();
// 依据一个非空值创建Optional
Optional<Car> optCar = Optional.of(car);
// 可接受null的Optional
Optional<Car> optCar = Optional.ofNullable(car);
```
### 使用map从Optional对象中提取和转换值

```java
// before
String name = null; 
if(insurance != null){ 
    name = insurance.getName(); 
} 
//After
Optional<Insurance> optInsurance = Optional.ofNullable(insurance); 
Optional<String> name = optInsurance.map(Insurance::getName); 
```
### 使用flatMap链接Optional对象

```java
// before
public String getCarInsuranceName(Person person) { 
    return person.getCar().getInsurance().getName(); 
} 
//After
public String getCarInsuranceName(Optional<Person> person) { 
    return person.flatMap(Person::getCar) 
        .flatMap(Car::getInsurance) 
        .map(Insurance::getName) 
        .orElse("Unknown"); 
}
```
**注意**： 由于Optional类设计时就没特别考虑将其作为类的字段使用，所以它也并未实现Serializable接口，所以它们无法序列化。

### 默认行为及解引用Optional对象

以下是Optional中的一些方法：

1. get()：如果变量存在，它直接返回封装的变量值，否则就抛出一个NoSuchElementException异常。
2. orElse(T other)： 在Optional对象不包含值时提供一个默认值
3. orElseGet(Supplier<? extends T> other)是orElse方法的延迟调用版，Supplier方法只有在Optional对象不含值时才执行调用。
4. orElseThrow(Supplier<? extends X> exceptionSupplier)和get方法非常类似，它们遭遇Optional对象为空时都会抛出一个异常，但是使用orElseThrow你可以定制希望抛出的异常类型。
5. ifPresent(Consumer<? super T>)让你能在变量值存在时执行一个作为参数传入的方法，否则就不进行任何操作。

### 两个Optional对象的组合

比如现在有一个方法，找出最便宜的保险公司。
```java
public Insurance findCheapestInsurance(Person person, Car car) { 
    // 不同的保险公司提供的查询服务
    // 对比所有数据
    return cheapestCompany; 
}
```
如果想要一个null-安全的版本的方法，它接受两个Optional对象作为参数，返回值是一个Optional<Insurance>对象，如果传入的任何一个参数值为空，它的返回值亦为空。

使用isPresent（）方法的实现：
```java
public Optional<Insurance> nullSafeFindCheapestInsurance( 
    Optional<Person> person, Optional<Car> car) { 
    if (person.isPresent() && car.isPresent()) { 
        return Optional.of(findCheapestInsurance(person.get(), car.get())); 
    } else { 
        return Optional.empty(); 
    } 
}
```
上述方法可以实现，但是还是在和以前一样，在做值的check。

更好的实现方式：
```java
public Optional<Insurance> nullSafeFindCheapestInsurance( 
    Optional<Person> person, Optional<Car> car) { 
    return person.flatMap(p -> car.map(c -> findCheapestInsurance(p, c)));
}
```

### 使用filter剔除特定的值

```java
// before
Insurance insurance = ...; 
if(insurance != null && "CambridgeInsurance".equals(insurance.getName())){     System.out.println("ok"); 
}
// after
Optional<Insurance> optInsurance = ...; 
optInsurance.filter(insurance -> 
    "CambridgeInsurance".equals(insurance.getName())) 
    .ifPresent(x -> System.out.println("ok"));
```

## 4. 使用Optional的实战示例

### 用Optional封装可能为null的值

举个例子，当从map中获取某个key对应的value时，如果map中找不到这个key，就会返回一个null。如果为此，我们加上if/else，无疑会使代码变得臃肿。Optional此时就是一个好的选择。
```java
// before
Object value = map.get("key");

// after
Optional value = Optional.ofNullable(map.get("key"));
```

### 异常与Optional的对比

有时候，Java API会以一个异常来代替返回null，这个时候我们就不得不在调用后，加上一个try/catch，这无疑增加了代码复杂度。然而为了向后兼容，Java API难以更改，所以可以自己构造一个方法，在异常时，返回Optional。

举个Integer.parseInt的例子，可以如此改写：
```java
public static Optional<Integer> stringToInt(String s) { 
    try { 
        return Optional.of(Integer.parseInt(s)); 
    } catch (NumberFormatException e) { 
        return Optional.empty(); 
    } 
}
```

**注意**：基础类型的Optional对象，应该避免使用它们，因为它们无法使用map、flatmap和filter方法。

### 结合以上

1. 创建一个类
```java
Properties props = new Properties(); 
props.setProperty("a", "5"); 
props.setProperty("b", "true"); 
props.setProperty("c", "-3");
```
2. 创建一个方法，它在value是正整数的字符串时，返回正整数的值，否则返回0
```java
public int readDuration(Properties props, String name)
```
3. 原始readDuration实现方式
```java
public int readDuration(Properties props, String name){
    String value = props.getProperty(name);
    if(value != null){
        try{
            int i = Integer.valueOf(value);
            if(i > 0){
                return i;
            }
        } catch (NumberFormatException nfe){}
    }
    return 0;
}
```
4. 使用Optional的readDuration实现方式
```java
public int readDuration(Properties props, String name){
    return Optional.ofNullable(props.getProperty(name))
            .flatMap(OptionalUtility::stringToInt)
            .filter(i -> i > 0)
            .orElse(0);
}
```

## 未完待续......