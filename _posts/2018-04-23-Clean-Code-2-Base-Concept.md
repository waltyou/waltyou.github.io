---
layout: post
title: 《代码整洁之道 Clean Code》学习日志（二）：基本概念
date: 2018-4-23 13:16:04
author: admin
comments: true
categories: [Java]
tags: [Clean Code]
---

整洁的代码，都有哪些标准呢？来看看

<!-- more -->

学习资料主要参考： 《代码整洁之道》，作者：(美国)马丁(Robert C. Martin) 译者：韩磊

---
## 目录
{:.no_toc}

* 目录
{:toc}
---

## 有意义的命名

### 1. 介绍

命名无处不在

### 2. 名副其实

通过阅读名字，就可以知道它为什么存在，它是什么，它做什么，它怎么用。



```java
// before
int d; // elapsed time in days
// after
int elapsedTimeInDays;
```

```java
// before
public List<int[]> getThem() {
    List<int[]> list1 = new ArrayList<int[]>();
    for (int[] x : theList){
        if (x[0] == 4)
        list1.add(x);
    }
    return list1;
}
// after
public List<Cell> getFlaggedCells() {
    List<Cell> flaggedCells = new ArrayList<Cell>();
    for (Cell cell : gameBoard){
        if (cell.isFlagged())
            flaggedCells.add(cell);
    }
    return flaggedCells;
}
```
### 3. 避免误导

- 必须避免留下掩藏代码本意的错误线索
- 提防使用不同之处较小的名称
- 以同样的方式拼写出同样的概念，而不是前后不一致
- 避免阅读上的混淆，如“0”与“O”，“1”与“I”

### 4. 做有意义的区分

1. 如果名称相异，那么意义也应不一样。

2. 尽量不以数字系列命名，如下：

    ```java
    public static void copyChars(char a1[], char a2[]) {
        for (int i = 0; i < a1.length; i++) {
            a2[i] = a1[i];
        }
    }
    ```
    可以修改如下:
    
    ```java
    public static void copyChars(char source[], char destination[]) {
        for (int i = 0; i < source.length; i++) {
            destination[i] = source[i];
        }
    }
    ```
3. 废话是来一种没意义的区分。如a/the，info/data
4. 废话都是冗余。如在表名中加入table，nameString为什么不用name，CustomerObject为什么不用Customer？

### 5. 使用读的出的名字

编程是中社会活动。一个可以读的名字，能使人的记忆和交流更方便。

```java
// before
class DtaRcrd102 {
    private Date genymdhms;
    private Date modymdhms;
    private final String pszqint = "102";
    /* ... */
};

// after
class Customer {
    private Date generationTimestamp;
    private Date modificationTimestamp;;
    private final String recordId = "102";
    /* ... */
};
```

### 6. 使用可搜索的名字

- 尽量不要用单字母和数字作为名字，很难被搜索。如英文字母e，是最常用的字母。
- 单字母可以用于局部变量。换而言之，名字的长短最好与它作用范围的大小成正比。
- 如果变量在代码的多处被使用，应该取一个好搜索的名字。

```java
// before
for (int j=0; j<34; j++) {
    s += (t[j]*4)/5;
}
// after
int realDaysPerIdealDay = 4;
const int WORK_DAYS_PER_WEEK = 5;
int sum = 0;
for (int j=0; j < NUMBER_OF_TASKS; j++) {
    int realTaskDays = taskEstimate[j] * realDaysPerIdealDay;
    int realTaskWeeks = (realdays / WORK_DAYS_PER_WEEK);
    sum += realTaskWeeks;
}
```

### 7. 避免使用编码

编码太多了，无需自找麻烦。

- 应当把类和函数做的足够小，从而不必使用成员变量的前缀
- 接口的实现不必要非要在名称前加“I”，如抽象工厂类的名称前可以不加“I”，因为它很明确就是一个工厂，它需要被实现，“ShapeFactory”要比“IShapeFactory”好。

### 8. 避免思维映射

避免让读者将你代码中的名字，映射到另外一个名字上。

要明白：**明确才是王道**。

### 9. 类名

类名应是名词或名词短语，而不是个动词。

如：*Customer, WikiPage, Account, AddressParser*

而不是：*Manager, Processor, Data, Info*

### 10. 方法名

方法名应该是个动词。

对属性访问器、修改器和断言，应该加上get、set、is前缀。

```java
string name = employee.getName();
customer.setName("mike");
if (paycheck.isPosted())...
```
重载构造器时，使用描述参数的静态工厂方法。

```java
// before
Complex fulcrumPoint = new Complex(23.0);
// after, this is better
Complex fulcrumPoint = Complex.FromRealNumber(23.0);
```

### 11. 别扮可爱

避免使用笑话或是方言

### 12. 每个概念对应一个词

给每个抽象概念选一个词，并一以贯之。

坏例子: 如果controller , manager, driver同时出现在一个代码中，肯定会让人崩溃。

### 13. 别用双关语

不能一味的“保持一致”，而将同一个词使用在不同场景下。应该避免这种情况。

### 14. 使用解决方案领域名称

使用程序员领域的名称，因为只有程序员才会看你的代码。

### 15. 使用来自所涉问题领域的名称

如果不能使用技术领域的名称，就要采用所涉问题领域的名称。至少程序员可以问产品经理。

### 16. 添加有意义的语境

需要用良好命名的类、函数或者名称空间，来为读者提供语境。如果没有，就只能加前缀了。

没语境的代码：
```java
private void printGuessStatistics(char candidate, int count) {
    String number;
    String verb;
    String pluralModifier;
    if (count == 0) {
        number = "no";
        verb = "are";
        pluralModifier = "s";
    } else if (count == 1) {
        number = "1";
        verb = "is";
        pluralModifier = "";
    } else {
        number = Integer.toString(count);
        verb = "are";
        pluralModifier = "s";
    }
    String guessMessage = String.format(
        "There %s %s %s%s", verb, number, candidate, pluralModifier
        );
    print(guessMessage);
}
```
有语境的代码
```java
public class GuessStatisticsMessage {
    private String number;
    private String verb;
    private String pluralModifier;
    
    public String make(char candidate, int count) {
        createPluralDependentMessageParts(count);
        return String.format(
                "There %s %s %s%s",
                verb, number, candidate, pluralModifier );
    }
    
    private void createPluralDependentMessageParts(int count) {
        if (count == 0) {
            thereAreNoLetters();
        } else if (count == 1) {
            thereIsOneLetter();
        } else {
            thereAreManyLetters(count);
        }
    }
    
    private void thereAreManyLetters(int count) {
        number = Integer.toString(count);
        verb = "are";
        pluralModifier = "s";
    }
    
    private void thereIsOneLetter() {
        number = "1";
        verb = "is";
        pluralModifier = "";
    }
    
    private void thereAreNoLetters() {
        number = "no";
        verb = "are";
        pluralModifier = "s";
    }
}
```

### 17. 不要添加没意义的语境

只要短名清晰，就不要用长名。

如对于Address来说，它就是一个好的类名，然而accountAddress和customerAddress更像是一个实例名。另外如果需要与MAC 地址，端口地址或者URL地址表示区别，那就使用PostalAddress，MAC，URL。

精确才是要点。

--- 

## 函数

### 1. 短小
```java
public static String testableHtml(
        PageData pageData,
        boolean includeSuiteSetup
    ) throws Exception {
        WikiPage wikiPage = pageData.getWikiPage();
        StringBuffer buffer = new StringBuffer();
        if (pageData.hasAttribute("Test")) {
            if (includeSuiteSetup) {
                WikiPage suiteSetup =
                    PageCrawlerImpl.getInheritedPage(
                    SuiteResponder.SUITE_SETUP_NAME, wikiPage
                );
                if (suiteSetup != null) {
                    WikiPagePath pagePath =
                        suiteSetup.getPageCrawler().getFullPath(suiteSetup);
                    String pagePathName = PathParser.render(pagePath);
                    buffer.append("!include -setup .")
                        .append(pagePathName)
                        .append("\n");
                }
            }
            WikiPage setup =
            PageCrawlerImpl.getInheritedPage("SetUp", wikiPage);
            if (setup != null) {
                WikiPagePath setupPath =
                    wikiPage.getPageCrawler().getFullPath(setup);
                String setupPathName = PathParser.render(setupPath);
                buffer.append("!include -setup .")
                    .append(setupPathName)
                    .append("\n");
            }
        }
        buffer.append(pageData.getContent());
        if (pageData.hasAttribute("Test")) {
            WikiPage teardown =
            PageCrawlerImpl.getInheritedPage("TearDown", wikiPage);
            if (teardown != null) {
                WikiPagePath tearDownPath =
                    wikiPage.getPageCrawler().getFullPath(teardown);
                String tearDownPathName = PathParser.render(tearDownPath);
                buffer.append("\n")
                    .append("!include -teardown .")
                    .append(tearDownPathName)
                    .append("\n");
            }
            if (includeSuiteSetup) {
                WikiPage suiteTeardown =
                    PageCrawlerImpl.getInheritedPage(
                    SuiteResponder.SUITE_TEARDOWN_NAME,
                    wikiPage
                );
                if (suiteTeardown != null) {
                    WikiPagePath pagePath =
                        suiteTeardown.getPageCrawler().getFullPath (suiteTeardown);
                    String pagePathName = PathParser.render(pagePath);
                    buffer.append("!include -teardown .")
                        .append(pagePathName)
                        .append("\n");
                }
            }
        }
        pageData.setContent(buffer.toString());
        return pageData.getHtml();
}
```
这容易看懂吗？来改善一次

```java
public static String renderPageWithSetupsAndTeardowns(
        PageData pageData, boolean isSuite
    ) throws Exception {
        boolean isTestPage = pageData.hasAttribute("Test");
        if (isTestPage) {
            WikiPage testPage = pageData.getWikiPage();
            StringBuffer newPageContent = new StringBuffer();
            includeSetupPages(testPage, newPageContent, isSuite);
            newPageContent.append(pageData.getContent());
            includeTeardownPages(testPage, newPageContent, isSuite);
            pageData.setContent(newPageContent.toString());
        }
        return pageData.getHtml();
}
```
这个好点了。但还是太长了，可以再改善一下。

```java
public static String renderPageWithSetupsAndTeardowns(
    PageData pageData, boolean isSuite) throws Exception {
    if (isTestPage(pageData))
        includeSetupAndTeardownPages(pageData, isSuite);
    return pageData.getHtml();
}
```

#### 代码块和缩进

if，else，while语句中，其中代码块应该只有一行。

这一行应该是个函数调用语句。

这个函数应该具有较强说明性的名称。

函数不应该大到足够容纳嵌套结构，所以最多一到两层。

### 2. 只做一件事

怎么判断函数只做了一件事？

如果函数只是做了函数名下同一抽象层的步骤，那么就是在做一件事。

另外一个判断方法就是，看一个函数能不能再拆出一个函数，而且这个新函数并不是单纯的重新诠释其实现。

### 3. 每个函数一个抽象层级

函数中混杂不同抽象层级，往往让人迷惑。而且还会产生“破窗效应”。

#### 自顶向下读代码：向下规则

每个函数后面都跟着位于下一抽象层级的函数。

### 4. switch语句

switch天生要做许多事。

```java
public Money calculatePay(Employee e)
    throws InvalidEmployeeType {
        switch (e.type) {
        case COMMISSIONED:
            return calculateCommissionedPay(e);
        case HOURLY:
            return calculateHourlyPay(e);
        case SALARIED:
            return calculateSalariedPay(e);
        default:
            throw new InvalidEmployeeType(e.type);
    }
}
```
以上代码有四个问题：
1. 太长，而且出现新雇员类型时，它会变更长
2. 它做了很多事
3. 它违背了单一权责原则（Single Responsibility Principle，SRP）
4. 它违反了开放闭合原则（Open Closed Principle，OCP），每次添加新类型时，就要修改它。

这个问题的解决方案，就是将switch埋到抽象工厂下，不让任何人看到。


```java
public abstract class Employee {
    public abstract boolean isPayday();
    public abstract Money calculatePay();
    public abstract void deliverPay(Money pay);
}
-----------------
public interface EmployeeFactory {
    public Employee makeEmployee(EmployeeRecord r) throws InvalidEmployeeType;
}
-----------------
public class EmployeeFactoryImpl implements EmployeeFactory {
    public Employee makeEmployee(EmployeeRecord r) throws InvalidEmployeeType {
        switch (r.type) {
        case COMMISSIONED:
            return new CommissionedEmployee(r) ;
        case HOURLY:
            return new HourlyEmployee(r);
        case SALARIED:
            return new SalariedEmploye(r);
        default:
            throw new InvalidEmployeeType(r.type);
        }
    }
}
```
### 5. 使用描述性名称

- 描述性强
- 不要怕长
- 别怕花时间
- 保持一致

### 6. 函数参数

#### 1）一元参数
大致分为两种。

一种是对参数进行转换，函数具有返回值；一种类似处理事件，没有返回值。

#### 2）标识参数（布尔类型参数）

非常不推荐，应该它表明了函数不只做一件事。

#### 3）二元参数

二元参数要比一元难懂。

然而有时候两个参数正好，如构建一个二维坐标系中的一个点，天生需要x，y坐标。

要学会利用适当机制，将二元转换为一元。

#### 4）三元参数

更加难懂，需要慎重

#### 5）参数对象

如果有两个，三个及以上的参数时，要考虑封装成类了。

#### 6）参数列表

有时候参数列表中存在可变参数。

#### 7）动词与关键字

给函数取个好名字，能更好的解释函数的意图，以及参数的顺序和意图。

### 7. 无副作用

尽量不要在函数中，更改参数的值。

```java
public class UserValidator {
    private Cryptographer cryptographer;
    public boolean checkPassword(String userName, String password) {
        User user = UserGateway.findByName(userName);
        if (user != User.NULL) {
            String codedPhrase = user.getPhraseEncodedByPassword();
            String phrase = cryptographer.decrypt(codedPhrase, password);
            if ("Valid Password".equals(phrase)) {
                Session.initialize();
                return true;
            }
        }
        return false;
    }
}
```
*Session.initialize();* 就是这一行产生的副作用。

应避免使用输出参数。如：
```java
appendFooter(s);

public void appendFooter(StringBuffer report)
```
会让我们花时间去阅读函数体。

### 8. 分割指令与询问

函数要么做什么事情，要么回答什么问题。不要混淆。

```java
// 如果有这样子一个函数
public boolean set(String attribute, String value);
// 被这样子调用，是很让人疑惑的
if (set("username", "unclebob"))...
// 应该这样子写：
if (attributeExists("username")) {
    setAttribute("username", "unclebob");
    ...
}
```

### 9. 使用异常代替返回码

返回错误码时，就是在要求调用者立即处理，而用异常则可稍后处理。

如果使用异常，错误的处理逻辑就可以从主路径代码中分离开。

#### 1）抽离Try、catch

Try/catch代码块丑陋不堪，要尽量把try和catch代码块分离。

```java
public void delete(Page page) {
    try {
        deletePageAndAllReferences(page);
    }
    catch (Exception e) {
        logError(e);
    }
}
private void deletePageAndAllReferences(Page page) throws Exception {
    deletePage(page);
    registry.deleteReference(page.name);
    configKeys.deleteKey(page.name.makeKey());
}
private void logError(Exception e) {
    logger.log(e.getMessage());
}
```

#### 2）错误处理就是一件事

如果try在一个函数中出现，那么这个函数的catch、finally代码块后也不该有其他内容。

#### 3）Error.java依赖磁铁

返回错误码，意味着有个类或者是枚举，定义了所有的错误码。

这样子的类就是**依赖磁铁**。

当这样子的类发生变动时，所有使用它的类都需要重新编译或部署。

使用异常的话，新异常可以从异常类中派生出来，无需重新编译或重新部署。

### 10. 别重复自己

重复是软件中一切邪恶的源头。

### 11. 结构化编程

在大函数中，不要出现break和continue语句，更不能有goto语句。

如果函数足够小，其实无所谓。

### 12. 如何写出这样的函数

先写，再打磨。没有人能一开始就写的优美。

### 13. 把写代码当做讲故事

真正的目标，是用代码在讲述系统的故事。

---
## 注释

注释最多算是一种必要的恶。若我们代码足够有表达力，则不需要注释。

注释的恰当用法，可以弥补我们在用代码表达意图的“失败”。

代码总在变动，而注释很难保持一致。

### 1. 注释不能美化那些槽糕的代码

有时间写注释，不如花时间把糟糕的代码变好

### 2. 用代码来阐述

学会用代码解释行为，而不是注释

### 3. 好注释

#### 1）法律信息
#### 2）提供信息的注释
#### 3）对意图的解释
解释为什么要这样子做
#### 4）阐释
把某种难懂的参数或者返回值，翻译为可读形式。
#### 5）警示
#### 6）TODO注释
#### 7）放大
注释可以用来放大某种看来不合理之物的重要性。
#### 8）公共API中的JavaDoc

### 4. 坏注释

#### 1）喃喃自语
写给自己看的
#### 2）多余的注释
#### 3）误导性注释
#### 4）循规式注释
要求所有的函数都需要有JavaDoc，或者每个变量都要有注释，是可笑的。
#### 5）日志式注释
每次更改就记录一次的注释，完全可以删除。
#### 6）废话注释
对显然的事，进行注释。
#### 7）可怕的废话
#### 8）能用函数和变量时就别用注释
#### 9）位置标志
有时候为了标识位置，加上一行注释。无用。
#### 10）括号后面的注释
如果函数足够小，不需要。
#### 11）归属与署名
现在有git
#### 12）注释掉的代码
#### 13）html注释
#### 14）非本地信息
不是描述本地上下文的注释信息。
#### 15）信息过多
很多无用的信息
#### 16）不明显的联系
注释和代码关系不大
#### 17）函数头
取个好名字吧
#### 18）非公共代码中的JavaDoc
如果不公用，代码写成JavaDoc的注释就很无趣了。

---

## 格式

### 1. 格式的目的

代码格式关乎沟通，而沟通是开发者的头等大事。

### 2. 垂直格式
单个文件，大多数为200行，最长为500行，就可以构造一个出色的系统。

#### 1）向报纸学习

头条告诉你是否要读下去，第一段是个故事大纲，粗线条概述，后面逐步讲述细节。

#### 2）概念间垂直方向的区隔

在不同思路间，加入空行。

#### 3）垂直方向上的靠近

如果说空白行隔开了概念，那么靠近的代码意味着它们之间联系更加紧密。

#### 4）垂直距离

我们都有过这样子的经历：在一个类中上下求索，想搞清楚它们之间是如何操作，如何相互相关的，那些变量是怎么赋值，怎么传递的。

这是个很令人沮丧的事，因为我们想要理解系统**做什么**，却花时间和精力在找到和记住那些代码碎片**在哪里**。

关系密切的概念应该相互靠近。

1. 变量声明：尽量靠近使用它的地方
2. 实体变量：java放在类的顶部
3. 相关函数：调用者在被调用者上面
4. 概念相关：相关性越强，放的越近

#### 5）垂直顺序

调用者在被调用者上面

### 3. 横向格式

一行代码要多宽？

现在IDE都支持format功能，此章就记个标题吧。

#### 1）水平方向上的区隔与靠近
#### 2）水平对其
#### 3）缩进
#### 4）空范围

### 4. 团队规则

一个团队的代码格式一定要一致。

---

## 对象与数据结构

将变量设为private的一个理由是：我们不想其他人依赖这个变量。

那我们为什么还要加get，set方法，将它们变的如公共变量一样呢？

### 1. 数据抽象

具象点
```java
public class Point {
    public double x;
    public double y;
}
```
抽象点
```java
public interface Point {
    double getX();
    double getY();
    void setCartesian(double x, double y);
    double getR();
    double getTheta();
    void setPolar(double r, double theta);
}
```
抽象点的好处在于，它不仅仅是在变量之上放了一个函数层那么简单。

隐藏关乎抽象。

也就是为了让用户不需要了解数据的实现就能操作数据本体。

### 2. 数据、对象的反对称性

过程式代码，更便于在不改动数据的情况下，添加函数。

面向对象代码，更便于在不改动函数的情况下，添加新类。

### 3. 德墨忒尔律

定理认为，类C的方法f只应该调用以下对象的方法：
- C
- 由f创建的对象
- 作为参数传递给f的对象
- 由C的实体变量所持有的对象

下面的代码就违反了定律：
```java
final String outputDir = ctxt.getOptions().getScratchDir().getAbsolutePath();
```
#### 1）火车失事

上述代码就被称为火车失事。
#### 2）混杂
#### 3）隐藏结构

### 4. 数据传送对象

最为精炼的一种数据结构，是一个只有公共变量，没有函数的类。

这类数据结构，有时候被称为**数据传送对象 DTO**。

---

## 未完待续......