---
layout: post
title: 《Java 8 in Action》学习日志（四）：高效Java8编程
date: 2018-3-28 22:15:04
author: admin
comments: true
categories: [Java]
tags: [Java,Java8]
---

这一部分主要介绍了Java 8的一些其他新特性，如默认方法，Optional，CompletableFuture，新的时间日期API等。并且介绍了由于引入了lambda和stream后，我们该如何重构、测试和调试代码。

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

### 1）可读性

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
    
### 2）灵活性

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

### 1）策略模式

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

### 2）模板方法

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

### 3）观察者模式
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


### 4）责任链模式

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

### 5）工厂模式

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

### 1）查看栈跟踪

由于Lambda表达式没有名字, 编译器只能为它们指定一个名字，比如lambda$main$0。

即使你使用了方法引用,还是有可能出现栈无法显示你使用的方法名的情况。

如果方法引用指向的是同一个类中声明的方法,那么它的名称是可以在栈跟踪中显示的。

涉及Lambda表达式的栈跟踪可能非常难理解。这是Java编译器未来版本可以改进的一个方面。

### 2）输出日志

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

### 1）可选方法

类继承了接口，但是对某些接口方法的实现留白，所以代码中会存在很多无用的代码。那么有了默认方法后，就可以对这些方法提供一个默认实现，这样子实体类就无需写上一个空方法。

### 2）行为的多继承

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

### 1）解决问题的三条规则

1. 类中的方法优先级最高
2. 如果无法依据第一条进行判断，那么子接口的优先级更高：函数签名相同时，优先选择拥有最具体实现的默认方法的接口，即如果B继承了A，那么B就比A更加具体。
3. 最后，如果还是无法判断，继承了多个接口的类必须通过**显式覆盖**和**调用期望**的方法，显式地选择使用哪一个默认方法的实现。


### 2）应用前两条规则
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

### 3）冲突及如何显式地消除歧义

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

### 4）菱形继承问题
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

### 1）创建Optional对象

```java
// 声明一个空的Optional
Optional<Car> optCar = Optional.empty();
// 依据一个非空值创建Optional
Optional<Car> optCar = Optional.of(car);
// 可接受null的Optional
Optional<Car> optCar = Optional.ofNullable(car);
```
### 2）使用map从Optional对象中提取和转换值

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
### 3）使用flatMap链接Optional对象

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

### 4）默认行为及解引用Optional对象

以下是Optional中的一些方法：

1. get()：如果变量存在，它直接返回封装的变量值，否则就抛出一个NoSuchElementException异常。
2. orElse(T other)： 在Optional对象不包含值时提供一个默认值
3. orElseGet(Supplier<? extends T> other)是orElse方法的延迟调用版，Supplier方法只有在Optional对象不含值时才执行调用。
4. orElseThrow(Supplier<? extends X> exceptionSupplier)和get方法非常类似，它们遭遇Optional对象为空时都会抛出一个异常，但是使用orElseThrow你可以定制希望抛出的异常类型。
5. ifPresent(Consumer<? super T>)让你能在变量值存在时执行一个作为参数传入的方法，否则就不进行任何操作。

### 5）两个Optional对象的组合

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

### 6）使用filter剔除特定的值

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

### 1）用Optional封装可能为null的值

举个例子，当从map中获取某个key对应的value时，如果map中找不到这个key，就会返回一个null。如果为此，我们加上if/else，无疑会使代码变得臃肿。Optional此时就是一个好的选择。
```java
// before
Object value = map.get("key");

// after
Optional value = Optional.ofNullable(map.get("key"));
```

### 2）异常与Optional的对比

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

### 3）结合以上

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

---
# CompletableFuture：组合式异步编程


在进行软件设计时，有两大趋势：
1. 与硬件条件结合，即提高并行能力
2. 与其他应用交互，即提高并发能力


对于第一点，Java 7引入了fork/join框架，可以将任务拆分为子任务，Java 8中引入并行流。

对于第二点，问题的本质其实就是该如何处理线程阻塞情况。因为阻塞时，CPU资源是完全浪费的。那在java中是怎么处理呢？


## 1. Future接口


### 1）是什么？怎么用？

Java 5中引入了Futrue接口。

它是什么呢？

简单说，它建模了一种异步计算,返回一个执行运算结果的引用,当运算结束后,这个引用被返回给调用方。

用代码语言讲就是，将耗时的操作封装在一个Callable对象中,再将它提交给 ExecutorService，然后就无需干涉了。

```java
ExecutorService executor = Executors.newCachedThreadPool(); 
// submit callable 到ExecutorService中，获得future引用
Future<Double> future = executor.submit(new Callable<Double>() { 
    public Double call() { 
        return doSomeLongComputation(); 
    }});
// 在执行callable时，可以做其他事情
doSomethingElse();
// 获取异步操作的结果，如果阻塞，最长等待1s
try { 
    Double result = future.get(1, TimeUnit.SECONDS); 
} catch (ExecutionException ee) { 
    // 计算抛出一个异常
} catch (InterruptedException ie) { 
    // 当前线程在等待过程中被中断
} catch (TimeoutException te) { 
    // 在Future对象完成之前超过已过期
}
```

### 2）局限性

#### 两个方面
1. 不够简洁
2. 很难表述清楚各个future之间的依赖关系

#### 一些难以解决的场景
- 将两个异步计算合并为一个，这两个异步计算之间相互独立，同时第二个又依赖于第一个的结果。
- 等待Future集合中的所有任务都完成
- 仅等待Future集合中最快结束的任务完成（有可能因为它们试图通过不同的方式计算同一个值），并返回它的结果。
- 通过编程方式完成一个Future任务的执行（即以手工设定异步操作结果的方式）
- 应对Future的完成事件（即当Future的完成事件发生时会收到通知，并能使用Future计算的结果进行下一步的操作，不只是简单地阻塞等待操作的结果）。

然而，Java 8中CompletableFuture的引入可以使以上皆变为可能。

## 2. 使用CompletableFuture构建异步应用

### 1）同步API与异步API

同步API：
> 你调用了某个方法，调用方在被调用方运行的过程中会等待，被调用方运行结束返回，调用方取得被调用方的返回值并继续运行，即阻塞式调用

异步API：
> 直接返回，或者至少在被调用方计算完成之前，将它剩余的计算任务交给另一个线程去做，该线程和调用方是异步的，即非阻塞式调用

### 2）例子

一起来构建"最佳价格查询器”（best-price-finder）的应用。它会查询多个在线商店，依据给定的产品或服务找出最低的价格。

1. 定义API
    ```java
    public class Shop { 
        public double getPrice(String product) { 
            // 待实现
        } 
    }
    ```
2. 添加模拟延迟
    ```java
    public static void delay() { 
        try { 
            Thread.sleep(1000L); 
        } catch (InterruptedException e) { 
            throw new RuntimeException(e); 
        } 
    } 
    ```
3. getPrice方法会调用delay方法，并返回一个随机计算的值
    ```java
    public double getPrice(String product) { 
        return calculatePrice(product); 
    } 
    private double calculatePrice(String product) { 
        delay(); 
        return random.nextDouble() * product.charAt(0) + product.charAt(1); 
    }
    ```
    显然这一步的实现，就是一个同步式API的实现，所有的操作都会为等待同步事件完成而等待1秒钟，这是无法接受的。
4. 将同步方法转换为异步方法
    ```java
    public Future<Double> getPriceAsync(String product) {
        CompletableFuture<Double> futurePrice = new CompletableFuture<>();
        new Thread( () -> { 
            double price = calculatePrice(product); futurePrice.complete(price);
        }).start(); 
        return futurePrice; 
    }
    ```
    这段代码，创建了一个代表异步计算的CompletableFuture对象实例，它在计算完成时会包含计算的结果。
    
    代码创建了另一个线程去执行实际的价格计算工作，不等该耗时计算任务结束，直接返回一个Future实例。
    
    当请求的产品价格最终计算得出时，你可以使用它的complete方法，结束completableFuture对象的运行，并设置变量的值。
5. 使用异步API
    ```java
    Shop shop = new Shop("BestShop"); 
    long start = System.nanoTime(); 
    Future<Double> futurePrice = shop.getPriceAsync("my favorite product"); 
    long invocationTime = ((System.nanoTime() - start) / 1_000_000); 
    System.out.println("Invocation returned after " + invocationTime + " msecs"); 
    // 执行更多任务，比如查询其他商店
    doSomethingElse(); 
    // 在计算商品价格的同时
    try { 
        double price = futurePrice.get(); 
        System.out.printf("Price is %.2f%n", price); 
    } catch (Exception e) { 
        throw new RuntimeException(e); 
    } 
    long retrievalTime = ((System.nanoTime() - start) / 1_000_000); 
    System.out.println("Price returned after " + retrievalTime + " msecs");
    ```

### 3）错误处理

如果计算价格时，内部产生了错误，而这些异常呢，会被限制在当前线程范围，最终会导致杀死该线程。于是get方法将永远阻塞。

那该怎么办呢？

简单想法是使用get方法的重载版，设置超时参数，这样子就可以解决永远阻塞的问题。

但是内部到底发生了什么问题呢？还是无从查起。

所以引入了新的方法：completeExceptionally，它可以将导致CompletableFuture内发生问题的异常抛出。

```java
public Future<Double> getPriceAsync(String product) { 
    CompletableFuture<Double> futurePrice = new CompletableFuture<>(); 
    new Thread( () -> { 
        try { 
            double price = calculatePrice(product); 
            futurePrice.complete(price); 
        } catch (Exception ex) { 
            futurePrice.completeExceptionally(ex); 
        } 
    }).start(); 
    return futurePrice; 
}
```

当运行时报错时，客户端会收到一个 ExecutionException 异常,该异常接收了一个包含失败原因的Exception 参数。

这样子我们就可以看到报错信息了。

### 4）更进一步：使用工厂方法supplyAsync创建CompletableFuture

```java
public Future<Double> getPriceAsync(String product) { 
    return CompletableFuture.supplyAsync(() -> calculatePrice(product)); 
}
```
supplyAsync方法接受一个生产者（Supplier）作为参数，返回一个CompletableFuture对象，该对象完成异步执行后会读取调用生产者方法的返回值。


## 3. 让代码免受阻塞之苦


### 1）准备与尝试

1. 定义一个商家列表
    ```java
    List<Shop> shops = Arrays.asList(new Shop("BestPrice"),
        new Shop("LetsSaveBig"),
        new Shop("MyFavoriteShop"),
        new Shop("BuyItAll"));
    ```
    
2. 顺序通过产品名称获取价格
    ```java
    public List<String> findPrices(String product) {
        return shops.stream()
            .map(shop -> String.format("%s price is %.2f",
                    shop.getName(), shop.getPrice(product)))
            .collect(toList());
    }
    ```
    
3. 验证 findPrices 的正确性和执行性能
 
    ```java
    long start = System.nanoTime();
    System.out.println(findPrices("myPhone27S"));
    long duration = (System.nanoTime() - start) / 1_000_000;
    System.out.println("Done in " + duration + " msecs");
    ```
    
    输出：
    > [BestPrice price is 123.26, LetsSaveBig price is 169.47, MyFavoriteShop price
        is 214.13, BuyItAll price is 184.74]
    >
    > Done in 4032 msecs

    可以发现，执行时间为4s多一些，因为shop.getPrice(product)方法是顺序执行的，每个方法都sleep了1s。
    

那么该如何改善呢？

### 2）使用并行流对请求进行并行操作
回忆stream中，有一个现成的并行操作：parallelStream。试试看。

```java
public List<String> findPrices(String product) {
    return shops.parallelStream()
        .map(shop -> String.format("%s price is %.2f",
        shop.getName(), shop.getPrice(product)))
        .collect(toList());
}
```
输出：
> [BestPrice price is 123.26, LetsSaveBig price is 169.47, MyFavoriteShop price
is 214.13, BuyItAll price is 184.74]
>
> Done in 1180 msecs

效果很不错，只花了1s多一点。能不能更好呢？


### 3）使用 CompletableFuture 发起异步请求

```java
public List<String> findPrices(String product) {
    // 使用工厂方法 supplyAsync 创建 CompletableFuture 对象
    List<CompletableFuture<String>> priceFutures =
        shops.stream()
        .map(shop -> CompletableFuture.supplyAsync(
                () -> shop.getName() + " price is " +
                    shop.getPrice(product)))
        .collect(Collectors.toList());
    
    // 等待所有异步操作结束
    return priceFutures.stream()
        .map(CompletableFuture::join)
        .collect(toList());
}
```
以上的操作使用了两个不同的stream流水线，而不是把两个map合在一起。这是为什么呢？

因为流操作之间是有延时特性的。如果两个map连接在一起，新的 CompletableFuture对象只有在前一个操作完全结束之后,才能创建。这个方法就变成了顺序执行了。


输出：
> [BestPrice price is 123.26, LetsSaveBig price is 169.47, MyFavoriteShop price
is 214.13, BuyItAll price is 184.74]
> 
> Done in 2005 msecs

结果还没有并行流好！

继续改进

### 4）寻找更好的方案

并行流的版本工作得非常好,那是因为它能并行地执行四个任务,所以它几乎能为每个商家分配一个线程。

所以如果是五个商家呢？

顺序流处理时间为：5s+，并行流处理时间为2s+，CompletableFuture版本时间为：2s+。


随着商店数目增加，并行流和CompletableFuture之间的性能差别不大。

究其原因，是因为：它们内部采用的是同样的通用线程池,默认都使用固定数目的线程,具体线程数取决于 Runtime.getRuntime().availableProcessors() 的返回值。


然而，CompletableFuture更有优势，因为它可以定制化执行器( Executor )。

### 5）定制执行器


> **线程池大小与处理器的利用率之比**
>
> 线程池大小 = 处理器的核的数目 * 期望的CPU利用率(该值应该介于0和1之间) * ( 1 + 等待时间与计算时间的比率 )


为“最优价格查询器”应用定制的执行器
```java
private final Executor executor =
    Executors.newFixedThreadPool(Math.min(shops.size(), 100),
        new ThreadFactory() {
            public Thread newThread(Runnable r) {
                Thread t = new Thread(r);
                // 使用守护线程,不会阻止程序的关停
                t.setDaemon(true);
                return t;
            }
});
```
该怎么使用这个executor呢？只需要把它当做第二个参数传入supplayAsync中即可。
```java
CompletableFuture.supplyAsync(() -> shop.getName() + " price is " + shop.getPrice(product), executor); 
```

**使用流还是 CompletableFutures ?**
1. 计算密集型，推荐stream，实现简单,同时效率也可能是最高的。
2. 涉及I/O操作，使用CompletableFuture 灵活性更好,可以依据等待/计算,或者
W/C的比率设定需要使用的线程数。这种情况不使用并行流的另一个原因是,处理流的流水线中如果发生I/O等待,流的延迟特性会让我们很难判断到底什么时候触发了等待。


## 4. 对多个异步任务进行流水线操作


### 1）准备工作

如果所有商店都使用一个折扣服务，

```java
public class Discount {
    public enum Code {
        NONE(0), SILVER(5), GOLD(10), PLATINUM(15), DIAMOND(20);
        
        private final int percentage;
        
        Code(int percentage) {
            this.percentage = percentage;
        }
    }
    
    // Discount类的其他实现
}
```
修改getPrice方法，现在以 Shop-Name:price:DiscountCode 的格式返回一个 String 类型的值。
```java
public String getPrice(String product) {
    double price = calculatePrice(product);
    Discount.Code code = Discount.Code.values()[
        random.nextInt(Discount.Code.values().length)];
    return String.format("%s:%.2f:%s", name, price, code);
}

private double calculatePrice(String product) {
    delay();
    return random.nextDouble() * product.charAt(0) + product.charAt(1);
}
```

### 2）实现折扣服务

1. 创建Quote 类，对商店返回字符串的解析操作进行封装。
    ```java
    public class Quote {
        private final String shopName;
        private final double price;
        private final Discount.Code discountCode;
        
        public Quote(String shopName, double price, Discount.Code code) {
            this.shopName = shopName;
            this.price = price;
            this.discountCode = code;
        }
        
        public static Quote parse(String s) {
            String[] split = s.split(":");
            String shopName = split[0];
            double price = Double.parseDouble(split[1]);
            Discount.Code discountCode = Discount.Code.valueOf(split[2]);
            return new Quote(shopName, price, discountCode);
        }
        
        public String getShopName() { return shopName; }
        public double getPrice() { return price; }
        public Discount.Code getDiscountCode() { return discountCode; }
    }
    ```
2. 在Discount中实现applyDiscount 方法，它接收一个 Quote 对象,返回一个字符串,表示生成该 Quote 的 shop 中的折扣价格。

    ```java
    public class Discount {
        public enum Code {
            // 省略......
        }
        
        public static String applyDiscount(Quote quote) {
            return quote.getShopName() + " price is " +
                Discount.apply(quote.getPrice(),
                quote.getDiscountCode());
        }
        
        private static double apply(double price, Code code) {
            delay();
            return format(price * (100 - code.percentage) / 100);
        }
    }
    ```

3. 使用 Discount 服务

    ```java
    public List<String> findPrices(String product) {
        return shops.stream()
            .map(shop -> shop.getPrice(product))
            .map(Quote::parse)
            .map(Discount::applyDiscount)
            .collect(toList());
    }
    ```

4. 构造同步和异步操作

    使用 CompletableFuture 实现 findPrices 方法
    ```java
    public List<String> findPrices(String product) {
        List<CompletableFuture<String>> priceFutures =
            shops.stream().map(shop -> CompletableFuture.supplyAsync(
                () -> shop.getPrice(product), executor))
            .map(future -> future.thenApply(Quote::parse))
            .map(future -> future.thenCompose(quote -> 
                    CompletableFuture.supplyAsync(
                        () -> Discount.applyDiscount(quote), executor)))
            .collect(toList());
        
        return priceFutures.stream()
            .map(CompletableFuture::join)
            .collect(toList());
    }
    ```
    为了构建priceFutures，分了三步走。
    
    第一步异步调用getPrice方法；
    
    第二步同步调用Quote的parse方法，因为它本身不会产生I/O操作，所以它可以同步操作。用CompletableFuture对象的thenApply直接调用。
    
    第三步异步调用applyDiscount方法。为了以级联的方式串接起两个异步操作进行工作，Java 8的CompletableFuture API中的thenCompose的方法就是为了这个目的而生。
    
    同时API也提供了**thenComposeAsync**方法，它的意思是将后续的任务在新的线程上运行。

5. 将两个CompletableFuture对象整合起来，无论它们是否存在依赖

    上一步中使用了**thenCompose**方法，使用它的场景简单来说，是当第二个CompletableFuture需要上个CompletableFuture结果时。
    
    但是有时候，我们也需要将两个完全不相干的CompletableFuture对象的结果整合起来，而且你也不希望等到第一个任务完全结束才开始第二项任务。
    
    这个时候应该使用**thenCombine**方法。
    
    我们再添加一个异步操作，让它去查询汇率。
    
    ```java
    Future<Double> futurePriceInUSD = 
        CompletableFuture.supplyAsync(() -> shop.getPrice(product)) 
        .thenCombine( 
            CompletableFuture.supplyAsync( 
            () -> exchangeService.getRate(Money.EUR, Money.USD)), 
            (price, rate) -> price * rate 
    );
    ```
    它接收名为BiFunction的第二参数，这个参数定义了当两个CompletableFuture对象完成计算后，结果如何合并。
    
    API也提供了Async版本，它会导致BiFunction中定义的合并操作被提交到线程池中，由另一个任务以异步的方式执行。
    
## 5. 响应CompletableFuture的completion事件

现实世界中，请求的延时都是不固定的，写代码来模拟一个。

```java
private static final Random random = new Random(); 
public static void randomDelay() { 
    // 0.5s ~ 2.5s
    int delay = 500 + random.nextInt(2000); 
    try { 
        Thread.sleep(delay); 
    } catch (InterruptedException e) { 
        throw new RuntimeException(e); 
    } 
}
```

现有代码的实现是当所有商店返回结果时，才显示价格。但是如果希望的效果是：只要有商店返回商品价格就在第一时间显示返回值，不管其他商店。

### 1）尝试

重构findPrices方法返回一个由Future构成的流
```java
public Stream<CompletableFuture<String>> findPricesStream(String product) { 
    return shops.stream() 
        .map(shop -> CompletableFuture.supplyAsync( 
                () -> shop.getPrice(product), executor)) 
        .map(future -> future.thenApply(Quote::parse)) 
        .map(future -> future.thenCompose(quote -> 
            CompletableFuture.supplyAsync( 
            () -> Discount.applyDiscount(quote), executor))); 
}
```
Java 8的CompletableFuture通过**thenAccept**方法提供了：在每个CompletableFuture上注册一个操作，该操作会在CompletableFuture完成执行后使用它的返回值。

如下，我们想打印一下返回值。
```java
findPricesStream("myPhone").map(f -> f.thenAccept(System.out::println));
```

类似thenCombine，thenAccept 也存在异步版： **thenAcceptAsync**，可以异步对结果进行消费。

**thenAccept**方法会返回一个CompletableFuture<Void>，所以在map操作后，会得到一个Stream<CompletableFuture<Void>>。

```java
CompletableFuture[] futures = findPricesStream("myPhone") 
    .map(f -> f.thenAccept(System.out::println)) 
    .toArray(size -> new CompletableFuture[size]); 

CompletableFuture.allOf(futures).join();
```

allOf工厂方法接收一个由CompletableFuture构成的数组，数组中的所有CompletableFuture对象执行完成之后，它返回一个CompletableFuture<Void>对象。这意味着，如果你需要等待最初Stream中的所有CompletableFuture对象执行完毕，对allOf方法返回的CompletableFuture执行join操作是个不错的主意。


如果希望只要CompletableFuture对象数组中有任何一个执行完毕就不再等待，可以使用一个类似的工厂方法anyOf。

### 2）付诸实践

```java
long start = System.nanoTime(); 
CompletableFuture[] futures = findPricesStream("myPhone27S") 
    .map(f -> f.thenAccept( 
        s -> System.out.println(s + " (done in " + 
        ((System.nanoTime() - start) / 1_000_000) + " msecs)"))) 
    .toArray(size -> new CompletableFuture[size]); 
    
CompletableFuture.allOf(futures).join(); 

System.out.println("All shops have now responded in " 
    + ((System.nanoTime() - start) / 1_000_000) + " msecs");
```

---
# 新的日期和时间API

## 0. 为什么需要新的

之前Java 中时间API的有什么呢？

两个：java.util.Date类与java.util.Calendar类

- 易用性很差
- 两个类同时存在，很使人困惑
- 有的特性只在某一个类有提供
- 这两个类都是可以变的


## 1. LocalDate、LocalTime、Instant、Duration以及Period

### 1）使用LocalDate和LocalTime

#### 表示日期的LocalDate
```java
LocalDate date = LocalDate.of(2014, 3, 18); // 2014-03-18
int year = date.getYear();  // 2014
Month month = date.getMonth();  // MARCH
int day = date.getDayOfMonth();  // 18
DayOfWeek dow = date.getDayOfWeek();  // TUESDAY
int len = date.lengthOfMonth(); // 31
boolean leap = date.isLeapYear(); // false

//使用工厂方法从系统时钟中获取当前的日期
LocalDate today = LocalDate.now();
```
如要过去年份月份等信息，也可以通过get方法，只需传入一个TemporalField参数即可。ChronoField枚举实现了TemporalField接口，所以可以参考下列代码来调用。

```java
int year = date.get(ChronoField.YEAR); 
int month = date.get(ChronoField.MONTH_OF_YEAR); 
int day = date.get(ChronoField.DAY_OF_MONTH); 
```
#### 表示时间的LocalTime

```java
LocalTime time = LocalTime.of(13, 45, 20);  // 13:45:20
int hour = time.getHour(); // 13
int minute = time.getMinute(); // 45
int second = time.getSecond(); // 20

```

#### parse方法

LocalDate和LocalTime都可以通过解析代表它们的字符串创建

```java
LocalDate date = LocalDate.parse("2014-03-18"); 
LocalTime time = LocalTime.parse("13:45:20");
```
如果需要可以传入自定义的时间格式，只需再传入一个**DateTimeFormatter**即可。

一旦传递的字符串参数无法被解析为合法的LocalDate或LocalTime对象，这两个parse方法都会抛出一个继承自RuntimeException的**DateTimeParseException**异常。

### 2）合并日期和时间

这个复合类名叫LocalDateTime，是LocalDate和LocalTime的合体。

它同时表示了日期和时间，但**不带有时区信息**，你可以直接创建，也可以通过合并日期和时间对象构造。

```java
// 2014-03-18T13:45:20 
LocalDateTime dt1 = LocalDateTime.of(2014, Month.MARCH, 18, 13, 45, 20); 
LocalDateTime dt2 = LocalDateTime.of(date, time); 
LocalDateTime dt3 = date.atTime(13, 45, 20); 
LocalDateTime dt4 = date.atTime(time); 
LocalDateTime dt5 = time.atDate(date);
```
反向获取
```java
LocalDate date1 = dt1.toLocalDate(); 
LocalTime time1 = dt1.toLocalTime(); 
```

### 3）机器的日期和时间格式

使用新的java.time.Instant类，它使用一个单一大整型数来表示一个持续时间段上某个点，也就是一个包含秒及纳秒所构成的数字。

```java
Instant.ofEpochSecond(3); 
Instant.ofEpochSecond(3, 0); 
Instant.ofEpochSecond(2, 1_000_000_000); // 为2s加上100w纳秒（1s）
Instant.ofEpochSecond(4, -1_000_000_000); // 为4s减去100w纳秒（1s）
int day = Instant.now()； // 当前时刻的时间戳
```
**注意**：这个类无法处理那些我们非常容易理解的时间单位

### 4）定义Duration或Period

上面所提到的类，都实现了**Temporal**接口。这个接口定义了如何读取和操纵为时间建模的对象的值。

当我们需要获取两个Temporal对象之间的距离时，该怎么做呢？

1. Duration.between
    
    对与两个LocalTimes对象、两个LocalDateTimes对象，或者两个Instant对象之间的duration，可调用**Duration.between**方法。
    ```java
    Duration d1 = Duration.between(time1, time2); 
    Duration d1 = Duration.between(dateTime1, dateTime2); 
    Duration d2 = Duration.between(instant1, instant2); 
    ```
    注意：between函数传入的两个参数，类型应该一样。

2. Period.between

    那么对两个LocalDate对象呢？
    
    可以调用**Period.between**。
    
    ```java
    Period tenDays = Period.between(LocalDate.of(2014, 3, 8),
                                    LocalDate.of(2014, 3, 18));
    ```
3. Duration和Period类其他的工厂类
    
    ```java
    Duration threeMinutes = Duration.ofMinutes(3); 
    Duration threeMinutes = Duration.of(3, ChronoUnit.MINUTES); 
    Period tenDays = Period.ofDays(10); 
    Period threeWeeks = Period.ofWeeks(3); 
    Period twoYearsSixMonthsOneDay = Period.of(2, 6, 1);
    ```

## 2. 操纵、格式化以及解析日期

以上所提到的这些日期时间对象都是不可修改的，这是为了更好地支持函数式编
程，确保线程安全，保持领域模式一致性而做出的重大设计决定。

那当我们想要修改日期时，怎么办呢？

### 1）直观操作

使用withAttribute方法
```
LocalDate date1 = LocalDate.of(2014, 3, 18); 
LocalDate date2 = date1.withYear(2011); 
LocalDate date3 = date2.withDayOfMonth(25); 
LocalDate date4 = date3.with(ChronoField.MONTH_OF_YEAR, 9);
```
以相对方式修改LocalDate对象的属性
```java
LocalDate date1 = LocalDate.of(2014, 3, 18); 
LocalDate date2 = date1.plusWeeks(1); 
LocalDate date3 = date2.minusYears(3); 
LocalDate date4 = date3.plus(6, ChronoUnit.MONTHS);
```
### 2）使用TemporalAdjuster

有时候我们需要对时间进行一些复杂的操作，这个时候可以使用重载版本的with方法，向其传递一个提供了更多定制化选择的TemporalAdjuster对象。

```java
import static java.time.temporal.TemporalAdjusters.*; 

//2014-03-18
LocalDate date1 = LocalDate.of(2014, 3, 18); 
//2014-03-23
LocalDate date2 = date1.with(nextOrSame(DayOfWeek.SUNDAY)); 
//2014-03-31
LocalDate date3 = date2.with(lastDayOfMonth());
```

#### TemporalAdjuster类中的工厂方法

方法名 | 描述
---|---
dayOfWeekInMonth | 创建一个新的日期，它的值为同一个月中每一周的第几天
firstDayOfMonth | 创建一个新的日期，它的值为当月的第一天
firstDayOfNextMonth | 创建一个新的日期，它的值为下月的第一天
firstDayOfNextYear | 创建一个新的日期，它的值为明年的第一天
firstDayOfYear | 创建一个新的日期，它的值为当年的第一天
firstInMonth | 创建一个新的日期，它的值为同一个月中，第一个符合星期几要求的值
lastDayOfMonth | 创建一个新的日期，它的值为当月的最后一天
lastDayOfNextMonth | 创建一个新的日期，它的值为下月的最后一天
lastDayOfNextYear | 创建一个新的日期，它的值为明年的最后一天
lastDayOfYear | 创建一个新的日期，它的值为今年的最后一天
lastInMonth | 创建一个新的日期，它的值为同一个月中，最后一个符合星期几要求的值
next/previous | 创建一个新的日期，并将其值设定为日期调整后或者调整前，第一个符合指定星期几要求的日期
nextOrSame/previousOrSame | 创建一个新的日期，并将其值设定为日期调整后或者调整前，第一个符合指定星期几要求的日期，如果该日期已经符合要求，直接返回该对象

#### TemporalAdjuster接口

```java
@FunctionalInterface 
public interface TemporalAdjuster { 
    Temporal adjustInto(Temporal temporal); 
}
```

试着自定义一个TemporalAdjuster接口。

就叫做NextWorkingDay类吧，该类能够计算明天的日期，同时过滤掉周六和周日这些节假日。

```java
public class NextWorkingDay implements TemporalAdjuster { 
    @Override 
    public Temporal adjustInto(Temporal temporal) { 
        DayOfWeek dow = 
            DayOfWeek.of(temporal.get(ChronoField.DAY_OF_WEEK)); 
        int dayToAdd = 1; 
        if (dow == DayOfWeek.FRIDAY) dayToAdd = 3; 
        else if (dow == DayOfWeek.SATURDAY) dayToAdd = 2; 
        return temporal.plus(dayToAdd, ChronoUnit.DAYS); 
    } 
}
//调用
date = date.with(new NextWorkingDay());
```
换成用lambda实现一下
```java
date = date.with(temporal -> { 
    DayOfWeek dow = 
        DayOfWeek.of(temporal.get(ChronoField.DAY_OF_WEEK)); 
    int dayToAdd = 1; 
    if (dow == DayOfWeek.FRIDAY) dayToAdd = 3; 
    else if (dow == DayOfWeek.SATURDAY) dayToAdd = 2; 
    return temporal.plus(dayToAdd, ChronoUnit.DAYS); 
}); 
```
同时TemporalAdjusters.ofDateAdjuster可以接受一个UnaryOperator<LocalDate>类型的参数，同时返回一个TemporalAdjusters。
```java
TemporalAdjuster nextWorkingDay = TemporalAdjusters.ofDateAdjuster( 
    temporal -> { 
        DayOfWeek dow = 
            DayOfWeek.of(temporal.get(ChronoField.DAY_OF_WEEK)); 
        int dayToAdd = 1; 
        if (dow == DayOfWeek.FRIDAY) dayToAdd = 3; 
        if (dow == DayOfWeek.SATURDAY) dayToAdd = 2; 
        return temporal.plus(dayToAdd, ChronoUnit.DAYS); 
    }); 
date = date.with(nextWorkingDay);
```

### 3）打印输出及解析日期、时间对象

java.time.format包提供了格式化以及解析日期、时间对象的功能。

1. 使用 **BASIC_ISO_DATE** 和 **ISO_LOCAL_DATE**
    ```java
    LocalDate date = LocalDate.of(2014, 3, 18); 
    // 20140318
    String s1 = date.format(DateTimeFormatter.BASIC_ISO_DATE); 
    // 2014-03-18
    String s2 = date.format(DateTimeFormatter.ISO_LOCAL_DATE);
    ```
2. 使用工厂方法parse
    
    通过解析代表日期或时间的字符串重新创建该日期对象。
    ```java
    LocalDate date1 = LocalDate.parse("20140318", 
                                    DateTimeFormatter.BASIC_ISO_DATE); 
    LocalDate date2 = LocalDate.parse("2014-03-18", 
                                    DateTimeFormatter.ISO_LOCAL_DATE); 
    ```
    和老的java.util.DateFormat相比较，所有的DateTimeFormatter实例都是**线程安全**的。

3. 按照某个模式创建DateTimeFormatter

    DateTimeFormatter类还支持一个静态工厂方法**ofPattern**，它可以按照某个特定的模式创建格式器.
    
    ```java
    DateTimeFormatter formatter = DateTimeFormatter.ofPattern("dd/MM/yyyy"); 
    LocalDate date1 = LocalDate.of(2014, 3, 18); 
    String formattedDate = date1.format(formatter); 
    LocalDate date2 = LocalDate.parse(formattedDate, formatter);
    ```
4. 创建一个本地化的DateTimeFormatter
    
    ofPattern方法也提供了一个重载的版本，使用它你可以创建某个Locale的格式器
    ```
    DateTimeFormatter italianFormatter = 
        DateTimeFormatter.ofPattern("d. MMMM yyyy", Locale.ITALIAN); 
    LocalDate date1 = LocalDate.of(2014, 3, 18); 
    String formattedDate = date.format(italianFormatter); // 18. marzo 2014 
    LocalDate date2 = LocalDate.parse(formattedDate, italianFormatter);
    ```
5. 构造一个DateTimeFormatter

    DateTimeFormatterBuilder类还提供了更复杂的格式器，以及其他非常强大的解析功能，比如区分大小写的解析、柔性解析（允许解析器使用启发式的机制去解析输入，不精确地匹配指定的模式）、填充，以及在格式器中指定可选节。
    ```java
    DateTimeFormatter italianFormatter = new DateTimeFormatterBuilder() 
        .appendText(ChronoField.DAY_OF_MONTH) 
        .appendLiteral(". ") 
        .appendText(ChronoField.MONTH_OF_YEAR) 
        .appendLiteral(" ") 
        .appendText(ChronoField.YEAR) 
        .parseCaseInsensitive() 
        .toFormatter(Locale.ITALIAN); 
    ```

## 3. 处理不同的时区和历法

### 1）基本介绍

之前看到的日期和时间的种类都不包含时区信息。

新的java.time.ZoneId类是老版java.util.TimeZone的替代品。

它的设计目标就是要让你无需为时区处理的复杂和繁琐而操心。

ZoneId类也是无法修改的。

#### 如何创建
```java
// 创建一个ZoneId
ZoneId romeZone = ZoneId.of("Europe/Rome"); 
// 将一个老的时区对象转换为ZoneId
ZoneId zoneId = TimeZone.getDefault().toZoneId(); 
```

#### 为时间点添加时区信息
```java
LocalDate date = LocalDate.of(2014, Month.MARCH, 18); 
ZonedDateTime zdt1 = date.atStartOfDay(romeZone); 
LocalDateTime dateTime = LocalDateTime.of(2014, Month.MARCH, 18, 13, 45); 
ZonedDateTime zdt2 = dateTime.atZone(romeZone); 
Instant instant = Instant.now(); 
ZonedDateTime zdt3 = instant.atZone(romeZone);
```
#### 通过ZoneId，你还可以将LocalDateTime转换为Instant
```java
LocalDateTime dateTime = LocalDateTime.of(2014, Month.MARCH, 18, 13, 45);
Instant instantFromDateTime = dateTime.toInstant(romeZone);
```
#### 反向得到LocalDateTime对象
```java
Instant instant = Instant.now(); 
LocalDateTime timeFromInstant = LocalDateTime.ofInstant(instant, romeZone); 
```

### 2）利用和UTC/格林尼治时间的固定偏差计算时区

可以使用ZoneOffset类，它是ZoneId的一个子类，表示的是当前时间和伦敦格林尼治子午线时间的差异。
```java
ZoneOffset newYorkOffset = ZoneOffset.of("-05:00"); 
```
**注意**，使用这种方式定义的ZoneOffset并未考虑任何日光时的影响，所以在大多数情况下，不推荐使用。

使用它创建这样的OffsetDateTime：使用ISO-8601的历法系统，以相对于UTC/格林尼治时间的偏差方式表示日期时间。
```java
LocalDateTime dateTime = LocalDateTime.of(2014, Month.MARCH, 18, 13, 45);
OffsetDateTime dateTimeInNewYork = OffsetDateTime.of(date, newYorkOffset); 
```

### 3）使用别的日历系统

新版的日期和时间API还提供了另一个高级特性，即对非ISO历法系统（non-ISO calendaring）的支持。

Java 8中另外还提供了4种其他的日历系统：ThaiBuddhistDate、MinguoDate、JapaneseDate以及HijrahDate。

#### 创建这些类的实例
```java
LocalDate date = LocalDate.of(2014, Month.MARCH, 18); 
JapaneseDate japaneseDate = JapaneseDate.from(date);
```

#### 为某个Locale显式地创建日历系统，接着创建该Locale对应的日期的实例
```java
Chronology japaneseChronology = Chronology.ofLocale(Locale.JAPAN); 
ChronoLocalDate now = japaneseChronology.dateNow();
```



