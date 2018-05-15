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

## 错误处理

### 1. 使用异常，而不是返回码
### 2. 先写Try Catch Finally 语句
### 3. 使用不可控异常

可控异常违反了开放/闭合原则

### 4. 给出异常发生的环境说明
### 5. 依调用者需要定义异常类
### 6. 定义常规流程
### 7. 别返回null
### 8. 别传递null

    java 8 中提供了Optional类来解决null这个问题，要善用它。
---

## 边界

该怎么将外来代码干净利落的整合进自己的代码中呢？

### 1. 使用第三方代码

要根据自己代码，进行适当封装。

### 2. 浏览和学习边界

善用测试

### 3. 学习log4j
### 4. 学习性测试的好处不仅是免费
### 5. 使用尚不存在的代码
### 6. 整洁的边界

改动底边界上常发生的事，所以在边界代码上，我们要清晰的分割和定义期望的测试，依靠能够确定的东西。


---

## 单元测试

### 1. TDD三定律

1. 除非这能让失败的单元测试通过，否则不允许去编写任何的产品代码。
2. 只允许编写刚好能够导致失败的单元测试。 （编译失败也属于一种失败）
3. 只允许编写刚好能够导致一个失败的单元测试通过的产品代码。

### 2. 保持测试整洁

脏测试等于没测试。

测试代码和生产代码一样重要。

### 3. 整洁的测试

三个要素：可读性，可读性和可读性。
```java
public void testGetPageHieratchyAsXml() throws Exception
{
    crawler.addPage(root, PathParser.parse("PageOne"));
    crawler.addPage(root, PathParser.parse("PageOne.ChildOne"));
    crawler.addPage(root, PathParser.parse("PageTwo"));
    request.setResource("root");
    request.addInput("type", "pages");
    Responder responder = new SerializedPageResponder();
    SimpleResponse response =
        (SimpleResponse) responder.makeResponse(
                            new FitNesseContext(root), request);
    String xml = response.getContent();
    assertEquals("text/xml", response.getContentType());
    assertSubString("<name>PageOne</name>", xml);
    assertSubString("<name>PageTwo</name>", xml);
    assertSubString("<name>ChildOne</name>", xml);
}

public void testGetPageHieratchyAsXmlDoesntContainSymbolicLinks()
throws Exception
{
    WikiPage pageOne = crawler.addPage(root, PathParser.parse("PageOne"));
    crawler.addPage(root, PathParser.parse("PageOne.ChildOne"));
    crawler.addPage(root, PathParser.parse("PageTwo"));
    PageData data = pageOne.getData();
    WikiPageProperties properties = data.getProperties();
    WikiPageProperty symLinks = properties.set(SymbolicPage.PROPERTY_NAME);
    symLinks.set("SymPage", "PageTwo");
    pageOne.commit(data);
    request.setResource("root");
    request.addInput("type", "pages");
    Responder responder = new SerializedPageResponder();
    SimpleResponse response =
        (SimpleResponse) responder.makeResponse(
                                new FitNesseContext(root), request);
    String xml = response.getContent();
    assertEquals("text/xml", response.getContentType());
    assertSubString("<name>PageOne</name>", xml);
    assertSubString("<name>PageTwo</name>", xml);
    assertSubString("<name>ChildOne</name>", xml);
    assertNotSubString("SymPage", xml);
}

public void testGetDataAsHtml() throws Exception
{
    crawler.addPage(root, PathParser.parse("TestPageOne"), "test page");
    request.setResource("TestPageOne");
    request.addInput("type", "data");
    Responder responder = new SerializedPageResponder();
    SimpleResponse response =
        (SimpleResponse) responder.makeResponse(
                            new FitNesseContext(root), request);
    String xml = response.getContent();
    assertEquals("text/xml", response.getContentType());
    assertSubString("test page", xml);
    assertSubString("<Test", xml);
}
```
重构后：
```java
public void testGetPageHierarchyAsXml() throws Exception {
    makePages("PageOne", "PageOne.ChildOne", "PageTwo");
    submitRequest("root", "type:pages");
    assertResponseIsXML();
    assertResponseContains(
        "<name>PageOne</name>", "<name>PageTwo</name>"，"<name>ChildOne</name>"
    );
}
public void testSymbolicLinksAreNotInXmlPageHierarchy() throws Exception {
    WikiPage page = makePage("PageOne");
    makePages("PageOne.ChildOne", "PageTwo");
    addLinkTo(page, "PageTwo", "SymPage");
    submitRequest("root", "type:pages");
    assertResponseIsXML();
    assertResponseContains(
        "<name>PageOne</name>", "<name>PageTwo</name>", "<name>ChildOne</name>"
    );
    assertResponseDoesNotContain("SymPage");
}
public void testGetDataAsXml() throws Exception {
    makePageWithContent("TestPageOne", "test page");
    submitRequest("TestPageOne", "type:data");
    assertResponseIsXML();
    assertResponseContains("test page", "<Test");
}
```
重构后，这些测试呈现了**构造-操作-检验**模式。每个测试都该这样子。

#### 1）面向特定领域的测试语言

可以构造一些清晰易懂的函数。

#### 2）双重标准

生产环境和测试环境不一样，所以可以依照双重标准来写测试用例。

### 4. 每个测试一个断言，每个测试一个概念

### 5. F.I.R.S.T

1. Fast 快速
2. Independent 独立
3. Repeatable 可重复
4. Self-Validating 自足验证
5. Timely 及时

---

## 类

### 1. 类的组织

对于java来讲，类应该由一组变量开始，公共静态变量第一，私有静态变量第二，私有实体变量第三，然后是很少的公共变量。

公共函数应该跟在变量之后，而公共函数调用的私有函数紧跟其后。

有时候为了使测试访问到，也可以把变量或者方法变为protect。

### 2. 类应该短小

对于函数，可以通过计算代码行数，对于类呢，可以通过计算**权责**。

想象一下，如果一个类有70多个公共方法，真是件可怕的事。

类的名称应该描述其权责。

所以我们大概需要25个词来描述一个类的功能，而且不用if、and、or、but等词汇。

#### 1）单一权责原则

单一权责（SRP）认为：**类或者模块应有且只一条修改它的理由**。

系统应该由许多短小的类而不是巨大的类组成，每个小类封装一个权责，只有一个修改的原因，并且与少数其他类一同协同达到期待的系统行为。

#### 2）内聚

类应该只有少量实体变量。类中的每个方法都应该操作一个或多个这种变量。

通常来讲，方法操作的变量越多，就越说它有内聚性。

一般来讲，创建这种极大化内聚性的类是不可取，也不可能的。另一方面，我们也希望内聚性保持在较高位置。

内聚性高，意味着类中方法和变量相互依赖，相互结合成一个逻辑整体。

#### 3）保持内聚性就会得到许多短小的类

将大函数拆分为小函数的过程，往往也是将类拆分成小类的时机。程序会更加有组织，也会拥有更为透明的结构。

### 3. 为了修改而组织

对于多数系统，修改一直在进行中。然而每处的修改，都会冒着影响系统其他部分无法工作的风险。

有效的分拆类，使类的功能更加内聚，对其他部分影响降为零。

类应该对拓展开放，对修改关闭。

#### 1）隔离修改

需求会改变，代码也会改变。

所以部件之间要解耦和隔离。

通过降低连接度，就引出了另一条需要遵守的原则：依赖倒置原则（Dependency Inversion Principle， DIP）。

它认为：类应依赖于抽象而不是依赖于具体细节。

---

## 系统

### 1. 如何构造一个城市

城市能够运转，是因为它演化出恰当的抽象等级和模块。

整洁的代码帮助我们在较低的抽象层达成有效运转这一目标。

这一章会讲述如何在更高抽象层“系统层级”上保持整洁。

### 2. 将系统的构造与使用分开

**构造**和**使用**，是两个非常不一样的过程。

> 软件系统应该将启始过程和启始过程后的运行时逻辑分离开，在启始过程中构建应用对象，也会存在互相纠缠的依赖关系。

将关注的方面分离开，是软件技艺中最古老也是最重要的设计技巧。

然而大多数程序，都没有做分离处理，如下:
```java
public Service getService() {
    if (service == null)
        service = new MyServiceImpl(...); // Good enough default for most cases?
    return service;
}
```

这段代码虽然是所谓的**延时构造、赋值**，有一些好处。但是我们也得到 MyServiceImpl 构造函数所需要的一切的硬编码依赖。

不分解这些依赖就无法编译，甚至在运行时永不使用这种类型的对象！

同时测试也是个难题，我们要在测试前，为service指定测试替身（test double）或者仿制对象（mock object）。

不过有了这些权责，说明方法违反了单一权责原则。

最重要的是，我们不知道 MyServiceImpl 是否在所有情形下都是正确的对象。

以下是分离的几个技巧。

#### 1）分解main

将全部构造过程迁到main或者称为main的模块中，设计系统的其余部分，假设所有对象都已正确构造和设置。

#### 2）工厂

有时候，应用程序也要负责确定何时创建对象，这时候可以使用工厂模式，构造细节隔离于应用系统程序代码之外。

#### 3）依赖注入

依赖注入（Dependency Injection， DI），控制反转（Inversion Of Control， IoC）。

### 3. 扩容

“一开始就做对系统”纯属神话。

> 软件系统和物理系统可以相互类比，它们都可以迭代式的增长，只要我们持续将关注面恰当地切分。

### 4. Java代理

Java代理适用于简单的情况。

### 5. 纯Java AOP 框架

AOP，面向方面编程。最著名的就是Spring。

### 6. AspectJ的方面

通过方面来实现关注面切分的功能最全的工具是AspectJ语言。

### 7. 测试驱动系统结构

最佳的系统结构由模块化的关注面领域组成，每个关注面均用纯java对象实现。不同的领域之间，用最不具有侵害性的方面或者类方面工具整合起来。
这种架构能测试驱动，就像代码一样。

### 8. 优化决策

拥有模块化关注面的POJO（Plain-Old Java Object）系统提供的敏捷能力，允许我们基于最新的知识做出优化的、时机刚好的决策，决策的复杂性也降低了。

### 9. 明智使用添加了可论证价值的标准

有了标准，就更易复用想法和组件、雇用拥有相关经验的人才、封装好点子，以及将组件封装起来。

### 10. 系统需要领域特定语言

领域特定语言允许所有抽象层级和应用程序中的所有领域，从高级策略到底层细节，使用POJO来表达。

---

## 迭进

### 1. 通过迭进设计达到整洁目的

四条简单的规则：
1. 运行所有测试
2. 不可重复
3. 表达了程序员的意图
4. 尽可能少的类和方法的数量
5. 以上按重要性排列

### 2. 简单设计规则1：运行所有测试

测试编写的越多，代码就越持续走向编写较易测试的代码。所以，确保系统完全可测试，可以帮助我们更好的设计代码。

### 3. 简单设计规则2~4：重构

测试消除了清理代码就会破坏代码的恐惧。

重构目的：提高内聚性，降低耦合度，切分关注面，模块化系统关注面，缩小函数和类的尺寸，选用更好的名称。

### 4. 不可重复

重复是良好系统的大敌。

### 5. 表达力

代码应该清晰地表达作者的意图。

做到表达力的最重要方式就是尝试。

### 6. 尽可能少的类和方法

即使是消除重复、提高表达力，也会被滥用。

我们的目标是保持函数和类短小的同时，保持整个系统的短小精悍。

不过要记住，这是四条规则中，优先级最低的一条。所以，尽管少的类和方法和重要，但是更重要的是测试、重构、不可重复。

---

## 并发编程

> 对象是过程的抽象，线程是调度的抽象。

编写整洁的并发程序很难。

### 1. 为什么要并发

并发是一种解耦策略，它帮助我们把做什么（目的）和何时做（时机）分解开。

并发也可以解决吞吐量问题，响应速度问题。

#### 迷思与误解

- 并发总能改进性能：其实只在多个线程或者处理器之间能分享大量等待时间时管用
- 编写并发程序无需修改设计：事实上，并发算法的设计和单线程系统设计极不相同
- 在采用Web或EJB容器的时候，理解并发问题并不重要：理解并发后，可以了解如果面对并发更新、死锁等问题

关于并发，比较中肯的说法：
- 并发会在性能和编写额外代码上增加一些开销
- 正确的并发是复杂的
- 并发的缺陷并非总能重现
- 并非常常需要对设计策略的根本性修改

### 2. 挑战

```java
public class X {
    private int lastIdUsed;
    public int getNextId() {
        return ++lastIdUsed;
    }
}
```
如果两个线程中共享这个实例对象，那么getNextId返回的结果就有可能不是我们想要的。

### 3. 并发防御原则

#### 1）单一权责原则

并发的实现细节常常直接嵌入到其他生产代码中。应考虑一些问题：
- 并发相关代码有自己的开发、修改和调优生命周期
- 开发相关代码有自己要对付的挑战，和非并发相关代码不同，而且往往更困难
- 写的不好的并发代码可能的出错方式数量也已经足具挑战性

**建议**：分离并发相关代码和其他代码

#### 2）推论：限制数据作用域

两个线程中修改共享对象的同一字段时，可能相互干扰，导致未预期的行为。

解决方案之一就是synchronized关键字，在代码中保护一块使用共享对象的临界区（critical section）。

临界区的数量很重要，数量越多，就越可能：
- 会忘记保护一个或多个临界区
- 需要多花力气保证一切都受到有效防护
- 很难找到错误原因

**建议**：谨记数据封装；严格限制对可能被共享的数据的访问。

#### 3）推论：使用数据复本

避免共享数据的好方法之一就是一开始就避免共享数据。

#### 4）推论：线程应尽可能地独立

**建议**：尝试将数据分解到可被独立线程操作的独立子集。

### 4. 了解java库

Java 5提供了许多并发开发方面的改进：
- 使用类库提供的线程安全群集
- 使用executor框架执行无关任务
- 尽可能使用非锁定解决方案
- 有几个类并不是线程安全的

支持高级并发设计的类：
1. ReentrantLock: 可以在一个方法中获取、在另一个方法中释放的锁
2. Semaphore： 经典的“信号”的一种实现，有计数器的锁
3. CountDownLatch： 在释放所有等待线程之前，等待指定数量事件发生的锁。这样子，所有线程都平等地几乎同时启动。

### 5. 了解执行模型

先了解一些基础定义：
1. 限定资源：并发环境中有这固定尺寸或者数量的资源
2. 互斥：每一时刻仅有一个线程能访问共享数据或资源
3. 线程饥饿：一个或一组线程在长时间或者永久被禁止
4. 死锁：两个线程互相持有对方所需资源，陷入无限等待中。
5. 活锁：执行次序一致的线程，但都发现其他线程已经“在路上”，由于竞争的原因，会持续尝试起步，但在很长时间甚至永远也不能启动。

#### 1）生产者-消费者模型

生产者和消费者之间的队列是一种限定资源。

#### 2）读者-作者模型

当存在一个主要为读者线程提供信息，偶尔被作者线程更新的共享资源，吞吐量是个问题。

增加吞吐量，会导致线程饥饿和过时信息的积累。作者线程的更新，要协调读者线程，影响吞吐量。

#### 3）宴席哲学家

一个圆桌，每个人左手边有个叉子，桌子中间有碗面，但是必须左右手同时拿叉子，才可以进餐。

大多数的并发问题都是以上三个问题的变种。

### 6. 警惕同步方法之间的依赖

建议：避免使用一个共享对象的多个方法。

如果必须有这种情况，有3种写对代码的手段：
1. 基于客户端的锁定：客户端代码在调用第一个方法前锁定服务端，确保锁的范围覆盖了调用最后一个方法的代码。
2. 基于服务端的锁定：在服务端内创建锁定服务端的方法，调用所有方法，然后解锁。让客户端代码调用新方法。
3. 适配服务端：创建执行锁定的中间层。

### 7. 保持同步区域微小

锁是昂贵的，它带来了延迟和额外开销。所有应该尽量减小同步区域。

### 8. 很难编写正确的关闭代码

编写运行一段时间后，平静关闭的系统代码，是件很难的事。

常见问题与死锁有关。

建议：尽早考虑关闭问题，尽早另其工作正常。

### 9. 测试线程代码

建议：编写有潜力暴露问题的测试。

#### 1）将伪失败看作可能的线程问题

不要将系统错误归咎于偶发事件。

#### 2）先使非线程代码可工作

不要同时追踪非线程缺陷和线程缺陷，确保代码在线程之外可工作。

#### 3）编写可插拔的线程代码

编写可以在数个配置环境下运行的线程代码。

#### 4）编写可调整的线程代码

可以在不同配置下试错。

#### 5）运行多于处理器数量的线程

系统在切换任务的时候回发生一些事，任务越多，切换越频繁，就越容易发现问题。

#### 6）在不同平台上运行

尽早并经常地在所有目标平台上运行线程代码。

#### 7）装置试错代码

增加对Objec.wait()、Object.sleep()、Object.yield()、Object.priority()等方法的调用，改变代码执行顺序。

有两种装置代码的方法
- 硬编码：手工向代码中插入wait、sleep、yield、priority的调用
- 自动化：使用ASM等工具来装置代码
