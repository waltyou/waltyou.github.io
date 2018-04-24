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

## 未完待续......