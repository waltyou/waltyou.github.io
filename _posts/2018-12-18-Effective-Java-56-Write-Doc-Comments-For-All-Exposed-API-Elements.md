---
layout: post
title: 《Effective Java》学习日志（七）56：给所有对外暴露的API写文档
date: 2018-12-18 18:54:04
author: admin
comments: true
categories: [Java]
tags: [Java,Effective Java]
---


<!-- more -->

---

学习资料主要参考： 《Effective Java Third Edition》，作者：Joshua Bloch

---

## 目录
{:.no_toc}

* 目录
{:toc}
---

如果API可用，则它必须有文档。
传统上，API文档是手动生成的，并且与代码保持同步是件苦差事。 
Java编程环境使用Javadoc实用程序简化了此任务。 
Javadoc使用特殊格式的文档注释（通常称为doc注释）从源代码自动生成API文档。

虽然文档注释约定不是语言的正式部分，但它们构成了每个Java程序员都应该知道的事实上的API。 
“How to Write Doc Comments”网页[Javadoc-guide]中介绍了这些约定。
虽然自Java 4发布以来该页面尚未更新，但它仍然是一个非常宝贵的资源。 
Java 9中添加了一个重要的doc标签，{@ index}; Java 8中的一个，{@implSpec}; 
Java 5中有两个，{@literal}和{@code}。
上述网页中缺少这些标签，但在此Item中进行了讨论。

**要正确记录API，必须在每个导出的类，接口，构造函数，方法和字段声明之前加上doc注释。**
如果一个类是可序列化的，你还应该记录它的序列化形式（第87项）。
在没有文档注释的情况下，Javadoc可以做的最好的事情是将声明重现为受影响的API元素的唯一文档。
使用缺少文档注释的API令人沮丧且容易出错。
公共类不应使用默认构造函数，因为无法为它们提供doc注释。
要编写可维护的代码，您还应该为大多数未导出的类，接口，构造函数，方法和字段编写doc注释，尽管这些注释不需要像导出的API元素那样彻底。

**方法的doc注释应该简洁地描述方法与其客户之间的契约。**
除了为继承而设计的类中的方法（第19项）之外，合同应该说明方法的作用而不是它的工作方式。 
doc注释应该枚举所有方法的前提条件，这些条件是客户端调用它的必须为真的，以及它的后置条件，这些条件是在调用成功完成后将成立的事情。
通常，对于未经检查的异常，@throws标记会隐式描述前提条件;每个未经检查的异常对应于前提条件违规。
此外，可以在@param标记中指定前提条件以及受影响的参数。

除前提条件和后置条件外，方法还应记录任何副作用。
副作用是系统状态的可观察到的变化，为了实现后置条件，这显然不是必需的。
例如，如果方法启动后台线程，则文档应记录它。

要完全描述方法的合约，doc注释应该为每个参数都有一个@param标记，@return标记除非方法具有void返回类型，并且对于方法抛出的每个异常都有@throws标记，无论是选中还是未选中（第74项）。
如果@return标记中的文本与方法的描述相同，则可以允许省略它，具体取决于您遵循的编码标准。

按照惯例，@param标记或@return标记后面的文本应该是描述参数或返回值表示的值的名词短语。
很少使用算术表达式代替名词短语;请参阅BigInteger的示例。 
@throws标记后面的文本应包含单词“if”，后跟一个描述抛出异常的条件的子句。
按照惯例，@param，@return或@throws标记之后的短语或子句不会被句点终止。
以下文档评论说明了所有这些约定：

```java
/**
* Returns the element at the specified position in this list.
*
* <p>This method is <i>not</i> guaranteed to run in constant
* time. In some implementations it may run in time proportional
* to the element position.
*
* @param index index of element to return; must be
*
non-negative and less than the size of this list
* @return the element at the specified position in this list
* @throws IndexOutOfBoundsException if the index is out of range
*
({@code index < 0 || index >= this.size()})
*/
E get(int index);

```

请注意在此doc注释（<p>和<i>）中使用HTML标记。 
Javadoc实用程序将doc注释转换为HTML，文档注释中的任意HTML元素最终都会生成HTML文档。
有时，程序员甚至会在他们的文档评论中嵌入HTML表格，尽管这种情况很少见。

还要注意在@throws子句中围绕代码片段使用Javadoc{@code}标记。
此标记有两个用途：它使代码片段以代码字体呈现，并禁止在代码片段中处理HTML标记和嵌套的Javadoc标记。
后一个属性允许我们在代码片段中使用小于号（<），即使它是HTML元字符。
要在文档注释中包含多行代码示例，请使用包含在HTML<pre>标记内的Javadoc{@code}标记。
换句话说，在代码示例之前加上字符<pre>{@code并跟随它} </pre>。
这样可以保留代码中的换行符，并且无需转义HTML元字符，但不需要转义符号（@），如果代码示例使用注释，则必须对其进行转义。

最后，请注意在doc评论中使用“this list”。
按照惯例，“this”一词指的是在实例方法的doc注释中使用方法时调用方法的对象。

如第15项所述，当您设计一个继承类时，您必须记录其自用模式，因此程序员知道覆盖其方法的语义。
应使用在Java 8中添加的@implSpec标记来记录这些自用模式。
回想一下，普通的doc注释描述了方法与其客户之间的契约;相反，@ implSpec注释描述了方法及其子类之间的契约，允许子类继承方法或通过super调用它时依赖于实现行为。
以下是它在实践中的表现：

```java
/**
* Returns true if this collection is empty.
*
* @implSpec
* This implementation returns {@code this.size() == 0}.
*
* @return true if this collection is empty
*/
public boolean isEmpty() { ... }

```

从Java 9开始，除非传递命令行开关-tag“implSpec：a：Implementation Requirements：”，否则Javadoc实用程序仍会忽略@implSpec标记。 

希望这将在随后的版本中得到补救。 不要忘记，您必须采取特殊操作来生成包含HTML元字符的文档，例如小于号（<），大于号（>）和符号（＆）。 将这些字符放入文档中的最佳方法是用{@literal}标记将它们包围起来，这样就可以禁止处理HTML标记和嵌套的Javadoc标记。 它类似于{@code}标记，除了它不以代码字体呈现文本。 例如，这个Javadoc片段：

```java
* A geometric series converges if {@literal |r| < 1}.
```

会生成文档：“如果| r | < 1 ，几何系列会收敛.“ {@literal}标记可能只放在小于号的位置，而不是整个不等式，并且使用相同的结果文档，但文档注释在源代码中的可读性较差。**这说明了doc注释在源代码和生成的文档中都应该是可读的一般原则。**如果您无法实现这两者，则生成的文档的可读性将胜过源代码的可读性。

每个文档注释的第一个“句子”（如下定义）将成为注释所属元素的摘要描述。例如，第255页上的doc注释中的摘要描述是“返回此列表中指定位置的元素。”摘要描述必须单独用于描述其汇总的元素的功能。为避免混淆，**类或接口中的两个成员或构造函数不应具有相同的摘要描述**。特别注意过载，使用相同的第一句通常很自然（但在文档评论中是不可接受的）。

如果预期的摘要描述包含句点，请小心，因为句点可能会提前终止描述。例如，以“大学学位”开头的文档评论，例如B.S.，M.S。或博士“将导致摘要描述”大学学位，如BS，MS“问题是摘要描述在第一个句点结束，后面是空格，制表符或行终止符（或第一个块标记） [Javadoc的参考]。这里，缩写“M.S.”中的第二个句点后跟一个空格。最好的解决方案是使用{@literal}标记包围违规期和任何相关文本，因此源代码中的空格后面不再有空格：

```
/**
* A college degree, such as B.S., {@literal M.S.} or Ph.D.
*/
public class Degree { ... }
```

说摘要描述是文档评论中的第一句话有点误导。 公约规定它很少应该是一个完整的句子。 对于方法和构造函数，摘要描述应该是描述该方法执行的操作的动词短语（包括任何对象）。 例如：

- ArrayList(int initialCapacity) —Constructs an empty list with the specified initial capacity.
- Collection.size() —Returns the number of elements in this collection.

如这些示例所示，使用第三人称声明时而不是第二人命令。

对于类，接口和字段，摘要描述应该是描述由类或接口的实例或字段本身表示的事物的名词短语。 例如：

- Instant —An instantaneous point on the time-line.
- Math.PI —The double value that is closer than any other to pi, the ratio of the circumference of a circle to its diameter.

总而言之，文档注释是记录API的最佳，最有效的方法。 对于所有导出的API元素，它们的使用应被视为必需的。 采用符合标准惯例的一致风格。 请记住，文档注释中允许使用任意HTML，并且必须转义HTML元字符。





