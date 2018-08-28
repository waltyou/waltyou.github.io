---
layout: post
title: 《Effective Java》学习日志（二）：Object的通用方法
date: 2018-08-12 21:11:04
author: admin
comments: true
categories: [Java]
tags: [Java,Effective Java]
---

这一篇主要谈一下 Object 类中的通用方法。

<!-- more -->
---

学习资料主要参考： 《Effective Java Third Edition》，作者：Joshua Bloch

---
## 目录
{:.no_toc}

* 目录
{:toc}

---

# Item 10：重写 equals 方法时遵守的通用约定

虽然Object是一个具体的类，但它主要是为继承而设计的。
它的所有非 final方法(equals、hashCode、toString、clone和finalize)都有清晰的通用约定（ general contracts），因为它们被设计为被子类重写。
任何类都有义务重写这些方法，以遵从他们的通用约定；如果不这样做，将会阻止其他依赖于约定的类(例如HashMap和HashSet)与此类一起正常工作。

## 1. 什么时候使用默认 equals 方法

重写equals方法看起来很简单，但是有很多方式会导致重写出错，同时其结果可能是可怕的。
避免此问题最简单的方法是不覆盖equals方法，但是在这种情况下，类的每个实例只与自身相等。

重写 equals 方法时，如果满足以下任一下条件，就是正确的：

- **每个类的实例都是固有唯一的**。对于像Thread这样代表活动实体而不是值的类来说，这是正确的。 Object提供的equals实现对这些类完全是正确的行为。
- **类不需要提供一个“逻辑相等（logical equality）”的测试功能**。
    例如java.util.regex.Pattern可以重写equals 方法检查两个是否代表完全相同的正则表达式Pattern实例，但是设计者并不认为客户需要或希望使用此功能。
    在这种情况下，从Object继承的equals实现是最合适的。
- **父类已经重写了equals方法，则父类行为完全适合于该子类**。例如，大多数Set从AbstractSet继承了equals实现、List从AbstractList继承了equals实现，Map从AbstractMap的Map继承了equals实现。
- **类是私有的或包级私有的，可以确定它的equals方法永远不会被调用**。如果你非常厌恶风险，可以重写equals方法，以确保不会被意外调用：
    ```java
    @Override public boolean equals(Object o) {
        throw new AssertionError(); // Method is never called
    }
    ```

## 2. 什么时候重写 equals 方法

那什么时候需要重写 equals 方法呢？如果一个类包含一个逻辑相等（ logical equality）的概念，此概念有别于对象标识（object identity），而且父类还没有重写过equals 方法。
这通常用在值类（ value classes）的情况。值类只是一个表示值的类，例如Integer或String类。
程序员使用equals方法比较值对象的引用，期望发现它们在逻辑上是否相等，而不是引用相同的对象。
重写 equals方法不仅可以满足程序员的期望，它还支持重写过equals 的实例作为Map 的键（key），或者 Set 里的元素，以满足预期和期望的行为。

一种不需要equals方法重写的值类是使用实例控制（instance control）（Item 1）的类，以确保每个值至多存在一个对象。 枚举类型（Itme 34）属于这个类别。对于这些类，逻辑相等与对象标识是一样的，所以Object的equals方法作用逻辑equals方法。

## 3. 重写 equals 方法的约定

当你重写equals方法时，必须遵守它的通用约定。Object的规范如下：equals方法实现了一个等价关系（equivalence relation）。它有以下这些属性:
- **自反性**：对于任何非空引用x，x.equals(x)必须返回true。
- **对称性**：对于任何非空引用x和y，如果且仅当y.equals(x)返回true时x.equals(y)必须返回true。
- **传递性**：对于任何非空引用x、y、z，如果x.equals(y)返回true，y.equals(z)返回true，则x.equals(z)必须返回true。
- **一致性**：对于任何非空引用x和y，如果在equals比较中使用的信息没有修改，则x.equals(y)的多次调用必须始终返回true或始终返回false。
- 对于任何非空引用x，x.equals(null)必须返回false。

如果一旦违反了它，很可能会发现你的程序运行异常或崩溃，并且很难确定失败的根源。套用约翰·多恩(John Donne)的说法，没有哪个类是孤立存在的。一个类的实例常常被传递给另一个类的实例。许多类，包括所有的集合类，都依赖于传递给它们遵守equals约定的对象。

接下来仔细理解一下上面提到的几个概念。

### 自反性（Reflexivity）

这一条是说一个对象必须是和它自己相等的。这个错误很难不知觉的发生。如果你违反了这个，很可能就重复添加了一个对象在一个集合中。

### 对称性（Symmetry）

这一条是说，任何两个对象对彼此的相等性判断应该是统一的。 这个错误很容易会发生，比如下面这个例子，来实现一个大小敏感的 string。

```java
// Broken - violates symmetry!
public final class CaseInsensitiveString {
    private final String s;
    
    public CaseInsensitiveString(String s) {
        this.s = Objects.requireNonNull(s);
    }
    // Broken - violates symmetry!
    @Override public boolean equals(Object o) {
        if (o instanceof CaseInsensitiveString)
            return s.equalsIgnoreCase(
                    ((CaseInsensitiveString) o).s);
        if (o instanceof String) // One-way interoperability!
            return s.equalsIgnoreCase((String) o);
        return false;
    }
    ... // Remainder omitted
}
```
然后我们在初始化两个 string 对象。
```java
CaseInsensitiveString cis = new CaseInsensitiveString("Polish");
String s = "polish";
```

可以很简单的看出来，_cis.equals(s)_ 返回 true， _s.equals(cis)_ 返回 false。

一旦违反了 equals 约定，你就不知道其他对象在面对你的对象时会如何表现。

为了修改这错误，可以这样写：
```java
@Override public boolean equals(Object o) {
    return o instanceof CaseInsensitiveString &&
            ((CaseInsensitiveString) o).s.equalsIgnoreCase(s);
}
```

### 传递性（Transitivity）

这个要求是说，如果第一个对象和第二个对象相等，第二个对象又和第三个对象相等，那么第一个和第三个对象也必然相等。

这个也很容易犯错。来看下面这个例子：

```java
// 父类
public class Point {
    private final int x;
    private final int y;
    
    public Point(int x, int y) {
        this.x = x;
        this.y = y;
    }
    
    @Override public boolean equals(Object o) {
        if (!(o instanceof Point))
            return false;
        Point p = (Point)o;
        return p.x == x && p.y == y;
    }
    ...    // Remainder omitted
}
// 子类
public class ColorPoint extends Point {
    private final Color color;
    
    public ColorPoint(int x, int y, Color color) {
        super(x, y);
        this.color = color;
    }
    ... // Remainder omitted
}
```

对于子类 ColorPoint 来讲， 它的 equals 方法应该怎么写呢？ 
如果完全省略，则实现继承自Point，并且在等比较中忽略颜色信息。 虽然这不违反平等合同，但显然是不可接受的。
假设你编写一个equals方法，只有当它的参数是另一个具有相同位置和颜色的颜色点时才返回true：
```java
// Broken - violates symmetry!
@Override public boolean equals(Object o) {
    if (!(o instanceof ColorPoint))
        return false;
    return super.equals(o) && ((ColorPoint) o).color == color;
}
```

我们来创建对象测试一下：
```java
Point p = new Point(1, 2);
ColorPoint cp = new ColorPoint(1, 2, Color.RED);
```

这个时候就会出现一个新的问题， _p.equals(cp)_ 返回 true， _cp.equals(p)_ 返回 false（因为类型不对）。
怎么保证一致呢？ 只能修改 ColorPoint 的 equals 方法， 让它在和 Point 对象比较时，忽略比较字段 color。
```java
// Broken - violates transitivity!
@Override public boolean equals(Object o) {
    if (!(o instanceof Point))
        return false;
    // If o is a normal Point, do a color-blind comparison
    if (!(o instanceof ColorPoint))
        return o.equals(this);
    // o is a ColorPoint; do a full comparison
    return super.equals(o) && ((ColorPoint) o).color == color;
}
```
然而上述解决方案，引入了新的问题。
```java
ColorPoint p1 = new ColorPoint(1, 2, Color.RED);
Point p2 = new Point(1, 2);
ColorPoint p3 = new ColorPoint(1, 2, Color.BLUE);
```

很明显的看出，_p1.equals(p2)_ 返回 true， _p2.equals(p3)_ 返回 true， 然而 _p1.equals(p3)_ 返回 false。

同时这个方法导致了一个内部的递归： 比如 Point 有两个子类，分别是 ColorPoint 和 SmellPoint， 它们都以类似上述的方式实现了自己的 equals 方法。
然后，当你调用 _myColorPoint.equals(mySmellPoint)_ 就会产生一个 StackOverflowError。

那么解决方案是什么呢？ 事实证明，这是面向对象语言中等价关系的基本问题。 
除非您愿意放弃面向对象抽象的好处，否则无法扩展可实例化的类并在保留 equals 约定的同时添加值组件。

你可能听说它可以扩展一个可实例化的类并添加一个值，同时通过在equals方法中使用 getClass 测试代替 instanceof 来实现 equals 约定。
```java
// Broken - violates Liskov substitution principle (page 43)
@Override public boolean equals(Object o) {
    if (o == null || o.getClass() != getClass())
        return false;
    Point p = (Point) o;
    return p.x == x && p.y == y;
}
```
这具有仅在对象具有相同实现类时才使对象等效的效果。
这可能看起来不是那么糟糕，但后果是不可接受的：Point的子类的实例仍然是一个Point，它仍然需要作为一个函数运行，但是如果采用这种方法它就不会这样做！

假设我们要写一个方法来判断一个Point 对象是否在unitCircle集合中。我们可以这样做：

```java
// Initialize unitCircle to contain all Points on the unit circle
private static final Set<Point> unitCircle = Set.of(
        new Point( 1, 0), new Point( 0, 1),
        new Point(-1, 0), new Point( 0, -1));

public static boolean onUnitCircle(Point p) {
    return unitCircle.contains(p);
}
```

虽然这可能不是实现功能的最快方法，但它可以正常工作。假设以一种不添加值组件的简单方式继承 Point 类，比如让它的构造方法跟踪记录创建了多少实例：

```java
public class CounterPoint extends Point {
    private static final AtomicInteger counter =
            new AtomicInteger();

    public CounterPoint(int x, int y) {
        super(x, y);
        counter.incrementAndGet();
    }

    public static int numberCreated() {
        return counter.get();
    }
}
```

里氏替代原则（ Liskov substitution principle）指出，任何类型的重要属性都应该适用于所有的子类型，因此任何为这种类型编写的方法都应该在其子类上同样适用。
这是我们之前声明的一个正式陈述，即Point的子类（如CounterPoint）仍然是一个Point，必须作为一个Point类来看待。 
但是，假设我们将一个CounterPoint对象传递给onUnitCircle方法。 
如果Point类使用基于getClass的equals方法，则无论CounterPoint实例的x和y坐标如何，onUnitCircle方法都将返回false。 
这是因为大多数集合（包括onUnitCircle方法使用的HashSet）都使用equals方法来测试是否包含元素，并且CounterPoint实例并不等于任何Point实例。 
但是，如果在Point上使用了适当的基于instanceof的equals方法，则在使用CounterPoint实例呈现时，同样的onUnitCircle方法可以正常工作。

虽然没有令人满意的方法来继承一个可实例化的类并添加一个值组件，但是有一个很好的变通方法：按照 Item 18的建议，“优先使用组合而不是继承”。
取代继承Point类的ColorPoint类，可以在ColorPoint类中定义一个私有Point属性，和一个公共的视图（view）（Item 6）方法，用来返回具有相同位置的ColorPoint对象。

```java
// Adds a value component without violating the equals contract
public class ColorPoint {
    private final Point point;
    private final Color color;

    public ColorPoint(int x, int y, Color color) {
        point = new Point(x, y);
        this.color = Objects.requireNonNull(color);
    }
    
    /**
     * Returns the point-view of this color point.
     */
    public Point asPoint() {
        return point;
    }

    @Override public boolean equals(Object o) {
        if (!(o instanceof ColorPoint))
            return false;
        ColorPoint cp = (ColorPoint) o;
        return cp.point.equals(point) && cp.color.equals(color);
    }
    
    ...    // Remainder omitted
}
```

Java平台类库中有一些类可以继承可实例化的类并添加一个值组件。 
例如，java.sql.Timestamp继承了java.util.Date并添加了一个nanoseconds字段。 
Timestamp的equals确实违反了对称性，并且如果Timestamp和Date对象在同一个集合中使用，或者以其他方式混合使用，则可能导致不稳定的行为。 
Timestamp类有一个免责声明，告诫程序员不要混用Timestamp和Date。 
虽然只要将它们分开使用就不会遇到麻烦，但没有什么可以阻止你将它们混合在一起，并且由此产生的错误可能很难调试。 
Timestamp类的这种行为是一个错误，不应该被仿效。

你可以将值组件添加到抽象类的子类中，而不会违反equals约定。
这对于通过遵循 Item 23 “优先考虑类层级（class hierarchies）来代替标记类（tagged classes）”中的建议而获得的类层级，是非常重要的。
例如，可以有一个没有值组件的抽象类Shape，子类Circle有一个radius属性，另一个子类Rectangle包含length和width属性。
只要不直接创建父类实例，就不会出现前面所示的问题。

### 一致性（Consistency） 

如果两个对象是相等的，除非一个（或两个）对象被修改了， 那么它们必须始终保持相等。 
换句话说，可变对象可以在不同时期可以与不同的对象相等，而不可变对象则不会。 
当你写一个类时，要认真思考它是否应该设计为不可变的。 
如果你认为应该这样做，那么确保你的equals方法强制执行这样的限制：相等的对象永远相等，不相等的对象永远都不会相等。

不管一个类是不是不可变的，都不要写一个依赖于不可靠资源的equals方法。 
如果违反这一禁令，满足一致性要求是非常困难的。 
例如，java.net.URL类中的equals方法依赖于与URL关联的主机的IP地址的比较。 
将主机名转换为IP地址可能需要访问网络，并且不能保证随着时间的推移会产生相同的结果。 
这可能会导致URL类的equals方法违反equals 约定，并在实践中造成问题。 
URL类的equals方法的行为是一个很大的错误，不应该被效仿。 
不幸的是，由于兼容性的要求，它不能改变。 为了避免这种问题，equals方法应该只对内存驻留对象执行确定性计算。

### Null无效（Non-nullity）

所有的对象都必须不等于 null。

许多类中的 equals方法都会明确阻止对象为null的情况：
```java
@Override public boolean equals(Object o) {
    if (o == null)
        return false;
    ...
}
``` 

这个判断是不必要的。 为了测试它的参数是否相等，equals方法必须首先将其参数转换为合适类型，以便调用访问器或允许访问的属性。 
在执行类型转换之前，该方法必须使用instanceof运算符来检查其参数是否是正确的类型：
```java
@Override public boolean equals(Object o) {
    if (!(o instanceof MyType))
        return false;
    MyType mt = (MyType) o;
    ...
}
```

如果此类型检查漏掉，并且equals方法传递了错误类型的参数，那么equals方法将抛出ClassCastException异常，这违反了equals约定。 
但是，如果第一个操作数为 null，则指定instanceof运算符返回false，而不管第二个操作数中出现何种类型[JLS，15.20.2]。 
因此，如果传入null，类型检查将返回false，因此不需要明确的 null检查。

## 4. 总结

综合起来，以下是编写高质量equals方法的配方（recipe）：
- 使用 == 运算符检查参数是否为该对象的引用。如果是，返回true。这只是一种性能优化，但是如果这种比较可能很昂贵的话，那就值得去做。
- 使用 instanceof 运算符来检查参数是否具有正确的类型。 如果不是，则返回false。 通常，正确的类型是equals方法所在的那个类。 
    有时候，该类实现了一些接口。 如果类实现了一个接口，该接口可以改进 equals约定以允许实现接口的类进行比较，那么使用接口。
    集合接口（如Set，List，Map和Map.Entry）具有此特性。
- 参数转换为正确的类型。因为转换操作在instanceof中已经处理过，所以它肯定会成功。
- 对于类中的每个“重要”的属性，请检查该参数属性是否与该对象对应的属性相匹配。
    如果所有这些测试成功，返回true，否则返回false。
    如果步骤2中的类型是一个接口，那么必须通过接口方法访问参数的属性;如果类型是类，则可以直接访问属性，这取决于属性的访问权限。

对于类型为非float或double的基本类型，使用 == 运算符进行比较；对于对象引用属性，递归地调用equals方法；
对于float 基本类型的属性，使用静态Float.compare(float, float)方法；对于double 基本类型的属性，使用Double.compare(double, double)方法。
由于存在Float.NaN，-0.0f和类似的double类型的值，所以需要对float和double属性进行特殊的处理；有关详细信息，请参阅JLS 15.21.1或Float.equals方法的详细文档。 
虽然你可以使用静态方法Float.equals和Double.equals方法对float和double基本类型的属性进行比较，这会导致每次比较时发生自动装箱，引发非常差的性能。 
对于数组属性，将这些准则应用于每个元素。 如果数组属性中的每个元素都很重要，请使用其中一个重载的Arrays.equals方法。

某些对象引用的属性可能合法地包含null。 为避免出现NullPointerException异常，请使用静态方法 Objects.equals(Object, Object)检查这些属性是否相等。

对于一些类，例如上的CaseInsensitiveString类，属性比较相对于简单的相等性测试要复杂得多。
在这种情况下，你想要保存属性的一个规范形式（ canonical form），这样 equals 方法就可以基于这个规范形式去做开销很小的精确比较，来取代开销很大的非标准比较。
这种方式其实最适合不可变类（条目 17）。一旦对象发生改变，一定要确保把对应的规范形式更新到最新。

equals方法的性能可能受到属性比较顺序的影响。 
为了获得最佳性能，你应该首先比较最可能不同的属性，开销比较小的属性，或者最好是两者都满足（derived fields）。 
你不要比较不属于对象逻辑状态的属性，例如用于同步操作的lock 属性。 
不需要比较可以从“重要属性”计算出来的派生属性，但是这样做可以提高equals方法的性能。 
如果派生属性相当于对整个对象的摘要描述，比较这个属性将节省在比较失败时再去比较实际数据的开销。 
例如，假设有一个Polygon类，并缓存该区域。 如果两个多边形的面积不相等，则不必费心比较它们的边和顶点。

当你完成编写完equals方法时，问你自己三个问题：它是对称的吗?它是传递吗?它是一致的吗?
除此而外，编写单元测试加以排查，除非使用AutoValue框架(第49页)来生成equals方法，在这种情况下可以安全地省略测试。
如果持有的属性失败，找出原因，并相应地修改equals方法。当然，equals方法也必须满足其他两个属性(自反性和非空性)，但这两个属性通常都会满足。

在下面这个简单的PhoneNumber类中展示了根据之前的配方构建的equals方法：

```java
// Class with a typical equals method
public final class PhoneNumber {

    private final short areaCode, prefix, lineNum;

    public PhoneNumber(int areaCode, int prefix, int lineNum) {
        this.areaCode = rangeCheck(areaCode, 999, "area code");
        this.prefix = rangeCheck(prefix, 999, "prefix");
        this.lineNum = rangeCheck(lineNum, 9999, "line num");
    }

    private static short rangeCheck(int val, int max, String arg) {
        if (val < 0 || val > max)
            throw new IllegalArgumentException(arg + ": " + val);
        
        return (short) val;
    }

    @Override
    public boolean equals(Object o) {
        if (o == this)
            return true;
        if (!(o instanceof PhoneNumber))
            return false;

        PhoneNumber pn = (PhoneNumber) o;

        return pn.lineNum == lineNum && pn.prefix == prefix
                && pn.areaCode == areaCode;
    }

    ... // Remainder omitted
}
```

以下是一些最后提醒：
- 当重写equals方法时，同时也要重写hashCode方法（Item 11）。
- 不要让equals方法试图太聪明。如果只是简单地测试用于相等的属性，那么要遵守equals约定并不困难。
    如果你在寻找相等方面过于激进，那么很容易陷入麻烦。
    一般来说，考虑到任何形式的别名通常是一个坏主意。
    例如，File类不应该试图将引用的符号链接等同于同一文件对象。幸好 File 类并没这么做。
- 在equal 时方法声明中，不要将参数Object替换成其他类型。
    对于程序员来说，编写一个看起来像这样的equals方法并不少见，然后花上几个小时苦苦思索为什么它不能正常工作：
    ```java
    // Broken - parameter type must be Object!
    public boolean equals(MyClass o) {
      ...
    }
    ```
    问题在于这个方法并没有重写Object.equals方法，它的参数是Object类型的，这样写只是重载了 equals 方法（Item 52）。 
    即使除了正常的方法之外，提供这种“强类型”的equals方法也是不可接受的，因为它可能会导致子类中的Override注解产生误报，提供不安全的错觉。
    在这里，使用Override注解会阻止你犯这个错误(Item 40)。这个equals方法不会编译，错误消息会告诉你到底错在哪里：
    ```java
    // Still broken, but won’t compile
    @Override 
    public boolean equals(MyClass o) {
      ...
    }
    ```

编写和测试equals(和hashCode)方法很繁琐，生的代码也很普通。
替代手动编写和测试这些方法的优雅的手段是，使用谷歌AutoValue开源框架，该框架自动为你生成这些方法，只需在类上添加一个注解即可。
在大多数情况下，AutoValue框架生成的方法与你自己编写的方法本质上是相同的。

很多 IDE（例如 Eclipse，NetBeans，IntelliJ IDEA 等）也有生成equals和hashCode方法的功能，但是生成的源代码比使用AutoValue框架的代码更冗长、可读性更差，不会自动跟踪类中的更改，因此需要进行测试。
这就是说，使用IDE工具生成equals(和hashCode)方法通常比手动编写它们更可取，因为IDE工具不会犯粗心大意的错误，而人类则会。

总之，除非必须：在很多情况下，不要重写equals方法，从Object继承的实现完全是你想要的。 
如果你确实重写了equals 方法，那么一定要比较这个类的所有重要属性，并且以保护前面equals约定里五个规定的方式去比较。

---

# Item 11：重写 equals 方法后，必须要重写 hashcode 方法

**在每个类中，在重写 equals 方法的时侯，一定要重写 hashcode 方法。**如果不这样做，你的类违反了hashCode的通用约定，这会阻止它在HashMap和HashSet这样的集合中正常工作。根据 Object 规范，以下时具体约定：

- 当在一个应用程序执行过程中，如果在equals方法比较中没有修改任何信息，在一个对象上重复调用hashCode方法时，它必须始终返回相同的值。从一个应用程序到另一个应用程序的每一次执行返回的值可以是不一致的。
- 如果两个对象根据equals(Object)方法比较是相等的，那么在两个对象上调用hashCode就必须产生的结果是相同的整数。
- 如果两个对象根据equals(Object)方法比较并不相等，则不要求在每个对象上调用hashCode都必须产生不同的结果。
    但是，程序员应该意识到，为不相等的对象生成不同的结果可能会提高散列表（hash tables）的性能。

当无法重写hashCode时，所违反第二个关键条款是：**相等的对象必须具有相等的哈希码（ hash codes）**。
根据类的equals方法，两个不同的实例可能在逻辑上是相同的，但是对于Object 类的hashCode方法，它们只是两个没有什么共同之处的对象。
因此，Object 类的hashCode方法返回两个看似随机的数字，而不是按约定要求的两个相等的数字。

举例说明，假设你使用 Item 10中的PhoneNumber类的实例做为HashMap的键（key）：

```java
Map<PhoneNumber, String> m = new HashMap<>();
m.put(new PhoneNumber(707, 867, 5309), "Jenny");
```

你可能期望m.get(new PhoneNumber(707, 867, 5309))方法返回Jenny字符串，但实际上，返回了 null。
注意，这里涉及到两个PhoneNumber实例：一个实例插入到 HashMap 中，另一个作为判断相等的实例用来检索。
PhoneNumber类没有重写 hashCode 方法导致两个相等的实例返回了不同的哈希码，违反了 hashCode 约定。
put 方法把PhoneNumber实例保存在了一个哈希桶（ hash bucket）中，
但get方法却是从不同的哈希桶中去查找，即使恰好两个实例放在同一个哈希桶中，get 方法几乎肯定也会返回 null。
因为HashMap 做了优化，缓存了与每一项（entry）相关的哈希码，如果哈希码不匹配，则不会检查对象是否相等了。

解决这个问题很简单，只需要为PhoneNumber类重写一个合适的 hashCode 方法。
hashCode方法是什么样的？写一个不规范的方法的是很简单的。
以下示例，虽然永远是合法的，但绝对不能这样使用：

```java
// The worst possible legal hashCode implementation - never use!
@Override public int hashCode() { return 42; }
```

这是合法的，因为它确保了相等的对象具有相同的哈希码。
这很糟糕，因为它确保了每个对象都有相同的哈希码。
因此，每个对象哈希到同一个桶中，哈希表退化为链表。
应该在线性时间内运行的程序，运行时间变成了平方级别。
对于数据很大的哈希表而言，会影响到能够正常工作。

一个好的 hash 方法趋向于为不相等的实例生成不相等的哈希码。
这也正是 hashCode 约定中第三条的表达。
理想情况下，hash 方法为集合中不相等的实例均匀地分配int 范围内的哈希码。
实现这种理想情况可能是困难的。
幸运的是，要获得一个合理的近似的方式并不难。 以下是一个简单的配方：

1. 声明一个 int 类型的变量result，并将其初始化为对象中第一个重要属性c的哈希码，如下面步骤2.a中所计算的那样。
    （回顾Item 10，重要的属性是影响比较相等的领域。）
2. 对于对象中剩余的重要属性f，请执行以下操作：
    1. 比较属性f与属性c的 int 类型的哈希码：
        1. 如果这个属性是基本类型的，使用Type.hashCode(f)方法计算，其中Type类是对应属性 f 基本类型的包装类。
        2. 如果该属性是一个对象引用，并且该类的equals方法通过递归调用equals来比较该属性，并递归地调用hashCode方法。 如果需要更复杂的比较，则计算此字段的“范式（“canonical representation）”，并在范式上调用hashCode。 如果该字段的值为空，则使用0（也可以使用其他常数，但通常来使用0表示）。
        3. 如果属性f是一个数组，把它看作每个重要的元素都是一个独立的属性。 也就是说，通过递归地应用这些规则计算每个重要元素的哈希码，并且将每个步骤2.b的值合并。 如果数组没有重要的元素，则使用一个常量，最好不要为0。如果所有元素都很重要，则使用Arrays.hashCode方法。
    2. 将步骤2.a中属性c计算出的哈希码合并为如下结果：result = 31 * result + c;
3. 返回 result 值。

当你写完hashCode方法后，问自己是否相等的实例有相同的哈希码。
编写单元测试来验证你的直觉（除非你使用AutoValue框架来生成你的equals和hashCode方法，在这种情况下，你可以放心地忽略这些测试）。
如果相同的实例有不相等的哈希码，找出原因并解决问题。

可以从哈希码计算中排除派生属性（derived fields）。
换句话说，如果一个属性的值可以根据参与计算的其他属性值计算出来，那么可以忽略这样的属性。
您必须排除在equals比较中没有使用的任何属性，否则可能会违反hashCode约定的第二条。

步骤 2.ii 中的乘法计算结果取决于属性的顺序，如果类中具有多个相似属性，则产生更好的散列函数。
例如，如果乘法计算从一个String散列函数中被省略，则所有的字符将具有相同的散列码。 
之所以选择31，因为它是一个奇数的素数。 
如果它是偶数，并且乘法溢出，信息将会丢失，因为乘以2相当于移位。 
使用素数的好处不太明显，但习惯上都是这么做的。 
31的一个很好的特性，是在一些体系结构中乘法可以被替换为移位和减法以获得更好的性能：31 * i ==（i << 5） - i。 
现代JVM可以自动进行这种优化。

让我们把上述办法应用到PhoneNumber类中：

```java
// Typical hashCode method

@Override public int hashCode() {

    int result = Short.hashCode(areaCode);

    result = 31 * result + Short.hashCode(prefix);

    result = 31 * result + Short.hashCode(lineNum);

    return result;

}
```

因为这个方法返回一个简单的确定性计算的结果，它的唯一的输入是PhoneNumber实例中的三个重要的属性，所以显然相等的PhoneNumber实例具有相同的哈希码。
实际上，这个方法是PhoneNumber的一个非常好的hashCode实现，与Java平台类库中的实现一样。 
它很简单，速度相当快，并且合理地将不相同的电话号码分散到不同的哈希桶中。

虽然在这个项目的方法产生相当好的哈希函数，但并不是最先进的。 
它们的质量与Java平台类库的值类型中找到的哈希函数相当，对于大多数用途来说都是足够的。
如果真的需要哈希函数而不太可能产生碰撞，请参阅Guava框架的的com.google.common.hash.Hashing [Guava]方法。

Objects类有一个静态方法，它接受任意数量的对象并为它们返回一个哈希码。
这个名为hash的方法可以让你编写一行hashCode方法，其质量与根据这个项目中的上面编写的方法相当。
不幸的是，它们的运行速度更慢，因为它们需要创建数组以传递可变数量的参数，以及如果任何参数是基本类型，则进行装箱和取消装箱。
这种哈希函数的风格建议仅在性能不重要的情况下使用。 以下是使用这种技术编写的PhoneNumber的哈希函数：

```java
// One-line hashCode method - mediocre performance

@Override public int hashCode() {

   return Objects.hash(lineNum, prefix, areaCode);

}
```

如果一个类是不可变的，并且计算哈希码的代价很大，那么可以考虑在对象中缓存哈希码，而不是在每次请求时重新计算哈希码。
如果你认为这种类型的大多数对象将被用作哈希键，那么应该在创建实例时计算哈希码。 
否则，可以选择在首次调用hashCode时延迟初始化（lazily initialize）哈希码。 
需要注意确保类在存在延迟初始化属性的情况下保持线程安全（Item 83）。 
PhoneNumber类不适合这种情况，但只是为了展示它是如何完成的。 
请注意，属性hashCode的初始值（在本例中为0）不应该是通常创建的实例的哈希码：

```java
// hashCode method with lazily initialized cached hash code

private int hashCode; // Automatically initialized to 0

@Override public int hashCode() {

    int result = hashCode;

    if (result == 0) {

        result = Short.hashCode(areaCode);

        result = 31 * result + Short.hashCode(prefix);

        result = 31 * result + Short.hashCode(lineNum);

        hashCode = result;

    }

    return result;

}
```

不要试图从哈希码计算中排除重要的属性来提高性能。 
由此产生的哈希函数可能运行得更快，但其质量较差可能会降低哈希表的性能，使其无法使用。
具体来说，哈希函数可能会遇到大量不同的实例，这些实例主要在你忽略的区域中有所不同。
如果发生这种情况，哈希函数将把所有这些实例映射到少许哈希码上，而应该以线性时间运行的程序将会运行平方级的时间。

这不仅仅是一个理论问题。 
在Java 2之前，String 类哈希函数在整个字符串中最多使用16个字符，从第一个字符开始，在整个字符串中均匀地选取。
对于大量的带有层次名称的集合（如URL），此功能正好显示了前面描述的病态行为。

不要为hashCode返回的值提供详细的规范，因此客户端不能合理地依赖它； 你可以改变它的灵活性。
Java类库中的许多类（例如String和Integer）都将hashCode方法返回的确切值指定为实例值的函数。
这不是一个好主意，而是一个我们不得不忍受的错误：它妨碍了在未来版本中改进哈希函数的能力。
如果未指定细节并在散列函数中发现缺陷，或者发现了更好的哈希函数，则可以在后续版本中对其进行更改。

总之，每次重写equals方法时都必须重写hashCode方法，否则程序将无法正常运行。
你的hashCode方法必须遵从Object类指定的常规约定，并且必须执行合理的工作，将不相等的哈希码分配给不相等的实例。
如果使用上述的配方，这很容易实现。
如Item 10所述，AutoValue框架为手动编写equals和hashCode方法提供了一个很好的选择，IDE也提供了一些这样的功能。

---

# Item 12: 始终重写 toString 方法

虽然Object类提供了toString方法的实现，但它返回的字符串通常不是你的类的用户想要看到的。 
它由类名后跟一个“at”符号（@）和哈希码的无符号十六进制表示组成，例如PhoneNumber@163b91。 
toString 的通用约定要求，返回的字符串应该是“一个简洁但内容丰富的表示，对人们来说是很容易阅读的”。
虽然可以认为PhoneNumber@163b91简洁易读，但相比于707-867-5309，但并不是很丰富 。 
toString 通用约定“建议所有的子类重写这个方法”。

虽然它并不像遵守 equals 和 hashCode 约定那样重要(Item 10和11)，但是提供一个良好的 toString 实现使你的类更易于使用，并对使用此类的系统更易于调试。
当对象被传递到println、printf、字符串连接操作符或断言，或者由调试器打印时，toString方法会自动被调用。
即使你从不调用对象上的toString，其他人也可以。
例如，对对象有引用的组件可能包含在日志错误消息中对象的字符串表示。如果未能重写toString，则消息可能是无用的。

如果为PhoneNumber提供了一个很好的toString方法，那么生成一个有用的诊断消息就像下面这样简单：

```java
System.out.println("Failed to connect to " + phoneNumber);
```

程序员将以这种方式生成诊断消息，不管你是否重写toString，但是除非你这样做，否则这些消息将不会有用。 
提供一个很好的toString方法的好处不仅包括类的实例，同样有益于包含实例引用的对象，特别是集合。 
打印map 对象时你会看到哪一个，{Jenny=PhoneNumber@163b91}还是{Jenny=707-867-5309}?

实际上，toString方法应该返回对象中包含的所有需要关注的信息，如电话号码示例中所示。 
如果对象很大或者包含不利于字符串表示的状态，这是不切实际的。 
在这种情况下，toString应该返回一个摘要，如 Manhattan residential phone directory (1487536 listings)或线程[main，5，main]。 
理想情况下，字符串应该是不言自明的（线程示例并没有遵守这点）。 
如果未能将所有对象的值得关注的信息包含在字符串表示中，则会导致一个特别烦人的处罚，测试失败报告如下所示：
```text
Assertion failure: expected {abc, 123}, but was {abc, 123}.
```

实现toString方法时，必须做出的一个重要决定是：在文档中指定返回值的格式。 
建议你对值类进行此操作，例如电话号码或矩阵类。 指定格式的好处是它可以作为标准的，明确的，可读的对象表示。 
这种表示形式可以用于输入、输出以及持久化可读性的数据对象，如CSV文件。 
如果指定了格式，通常提供一个匹配的静态工厂或构造方法，是个好主意，所以程序员可以轻松地在对象和字符串表示之间来回转换。 
Java平台类库中的许多值类都采用了这种方法，包括BigInteger，BigDecimal和大部分基本类型包装类。

指定toString返回值的格式的缺点是，假设你的类被广泛使用，一旦指定了格式，就会终身使用。
程序员将编写代码来解析表达式，生成它，并将其嵌入到持久数据中。
如果在将来的版本中更改了格式的表示，那么会破坏他们的代码和数据，并且还会抱怨。
但通过选择不指定格式，就可以保留在后续版本中添加信息或改进格式的灵活性。

无论是否决定指定格式，你都应该清楚地在文档中表明你的意图。如果指定了格式，则应该这样做。例如，这里有一个toString方法，该方法在条目 11中使用PhoneNumber类：

```java
/**
 * Returns the string representation of this phone number.
 * The string consists of twelve characters whose format is
 * "XXX-YYY-ZZZZ", where XXX is the area code, YYY is the
 * prefix, and ZZZZ is the line number. Each of the capital
 * letters represents a single decimal digit.
 *
 * If any of the three parts of this phone number is too small
 * to fill up its field, the field is padded with leading zeros.
 * For example, if the value of the line number is 123, the last
 * four characters of the string representation will be "0123".
 */
@Override public String toString() {
    return String.format("%03d-%03d-%04d",
            areaCode, prefix, lineNum);
}
```

如果你决定不指定格式，那么文档注释应该是这样的：

```java
/**
 * Returns a brief description of this potion. The exact details
 * of the representation are unspecified and subject to change,
 * but the following may be regarded as typical:
 *
 * "[Potion #9: type=love, smell=turpentine, look=india ink]"
 */
@Override public String toString() { ... }

```

在阅读了这条注释之后，那些生成依赖于格式细节的代码或持久化数据的程序员，在这种格式发生改变的时候，只能怪他们自己。

无论是否指定格式，都可以通过编程方式访问toString返回的值中包含的信息。 
例如，PhoneNumber类应该包含 areaCode, prefix, lineNum这三个属性。 如果不这样做，就会强迫程序员需要这些信息来解析字符串。 
除了降低性能和程序员做不必要的工作之外，这个过程很容易出错，如果改变格式就会中断，并导致脆弱的系统。 
由于未能提供访问器，即使已指定格式可能会更改，也可以将字符串格式转换为事实上的API。

在静态工具类（条目 4）中编写toString方法是没有意义的。 
你也不应该在大多数枚举类型（条目 34）中写一个toString方法，因为Java为你提供了一个非常好的方法。 
但是，你应该在任何抽象类中定义toString方法，该类的子类共享一个公共字符串表示形式。 
例如，大多数集合实现上的toString方法都是从抽象集合类继承的。

Google 的开放源代码 AutoValue 工具在条目 10中讨论过，它为你生成一个toString方法，就像大多数IDE工具一样。 
这些方法非常适合告诉你每个属性的内容，但并不是专门针对类的含义。 
因此，例如，为我们的PhoneNumber类使用自动生成的toString方法是不合适的（因为电话号码具有标准的字符串表示形式），但是对于我们的Potion类来说，这是完全可以接受的。 
也就是说，自动生成的toString方法比从Object继承的方法要好得多，它不会告诉你对象的值。

回顾一下，除非父类已经这样做了，否则在每个实例化的类中重写Object的toString实现。 
它使得类更加舒适地使用和协助调试。 toString方法应该以一种美观的格式返回对象的简明有用的描述。

---

# Item 13: 谨慎的重写 clone

Cloneable 接口的目的是作为一个 mixin 接口（Item 20），来标识那些宣称自己可以被克隆的类。
不幸的是，它没有做的这个目的。它最主要的缺陷是缺少了 clone 方法，而且对象的 clone 方法是 protected。

如果不使用反射（Item 65），就不能仅仅因为它实现了 Cloneable 接口而在对象上调用克隆。
即使是反射调用也可能失败，因为无法保证对象具有可访问的克隆方法。 
尽管存在这个缺陷和许多其他问题，但该设施的使用范围相当广泛，因此理解它是值得的。 
这个 Item 会告诉您如何实现良好的克隆方法，讨论何时适合这样做，并提供替代方法。

既然它不包含任何方法，那么 Cloneable 做了什么？ 
它确定了 Object 的受保护克隆实现的行为：如果一个类实现Cloneable，则Object的clone方法返回该对象的逐个字段副本; 否则会抛出CloneNotSupportedException。 
这是一个非常非典型的接口使用，而不是一个被模拟的接口。 
通常，实现接口会说明类可以为其客户做些什么。 
在这种情况下，它会修改超类上受保护方法的行为。

虽然规范没有说明，但实际上，实现 Cloneable 的类应该提供一个功能正常的公共克隆方法。 
为了实现这一目标，该类及其所有超类必须遵守复杂的，不可执行的，精简的协议。 
由此产生的机制是脆弱的，危险的和超语言的：它创建对象而不调用构造函数。

克隆方法的一般合同很弱。 这是从对象规范中复制的：
> 创建并返回此对象的副本。 “复制”的确切含义可能取决于对象的类别。 
一般意图是，对于任何对象x，表达式: _x.clone() != x_ 会返回 true，
表达式：_x.clone().getClass() == x.getClass()_ 也会返回 ture。但这些并非绝对要求。
特别的是 _x.clone().equals(x)_ 也会返回true， 单这个没有特别要求。
按照惯例，此方法返回的对象应该通过调用 super.clone 来获取。 
如果一个类及其所有超类（Object除外）都遵循这个约定，那就是这种情况： _x.clone().getClass() == x.getClass()_
按照惯例，返回的对象应该独立于被克隆的对象。 要实现此独立性，可能需要在返回之前修改super.clone返回的对象的一个或多个字段。 

这个机制与构造函数链接类似，只是它没有强制执行：
如果一个类的clone方法返回一个不是通过调用super.clone而是通过调用构造函数获得的实例，编译器不会抱怨，
但如果是该类的子类调用super.clone，生成的对象将具有错误的类，从而阻止克隆方法的子类正常工作。 
如果覆盖clone的类是final，则可以安全地忽略此约定，因为没有子类需要担心。 
但是如果final类有一个不调用super.clone的clone方法，那么该类没有理由实现Cloneable，因为它不依赖于Object的clone实现的行为。

假设您要在一个类中实现Cloneable，该类的超类提供了良好的克隆方法。 
先调用super.clone。 
您获得的对象将是原始的完整功能的副本。 
在您的类中声明的任何字段将具有与原始值相同的值。 
如果每个字段都包含原始值或对不可变对象的引用，则返回的对象可能正是您所需要的，在这种情况下，不需要进一步处理。 
例如，对于Item 11中的PhoneNumber类就是这种情况，但请注意，不可变类应该永远不提供克隆方法，因为它只会鼓励浪费复制。 
有了这个警告，PhoneNumber的克隆方法看起来应该是这个样子的：
```java
// Clone method for class with no references to mutable state
@Override 
public PhoneNumber clone() {
    try {
        return (PhoneNumber) super.clone();
    } catch (CloneNotSupportedException e) {
        throw new AssertionError(); // Can't happen
    }
}
```
为了使此方法起作用，必须修改PhoneNumber的类声明以指示它实现Cloneable。
通过Object clone方法返回Object，此clone方法返回Phone Number。
这是合法并且期望的做法，因为 Java 支持协变返回类型（covariant return types）。
换句话说，就是一个重写的方法的返回类型，可以是方法定义时返回类型的子类。
这消除了在客户端进行 cast 的需要。
我们必须在返回之前将super.clone的结果从Object转换为PhoneNumber，但这个转换是保证成功的。

调用 super.clone 包含在一个 try-catch 块。这是因为 Object 声明了它的 clone 方法要抛出一个受检异常 CloneNotSupportException。
因为 PhoneNumber 实现了 Cloneable 接口，我们知道了调用 super.clone 一定会成功。
对此样板文件的需求表明应该取消选中 CloneNotSupportedException（Item 71）。

如果对象包含引用可变对象的字段，则前面显示的简单克隆实现可能是灾难性的。
举个例子：
```java
public class Stack {

    private Object[] elements;
    private int size = 0;
    private static final int DEFAULT_INITIAL_CAPACITY = 16;

    public Stack() {
        this.elements = new Object[DEFAULT_INITIAL_CAPACITY];
    }

    public void push(Object e) {
        ensureCapacity();
        elements[size++] = e;
    }

    public Object pop() {
        if (size == 0)
            throw new EmptyStackException();
        Object result = elements[--size];

        elements[size] = null; // Eliminate obsolete reference
        return result;
    }

    // Ensure space for at least one more element.
    private void ensureCapacity() {
        if (elements.length == size)
            elements = Arrays.copyOf(elements, 2 * size + 1);
    }
}
```
假设你想让这个类可以克隆。 
如果clone方法仅返回super.clone()调用的对象，那么生成的Stack实例在其size 属性中具有正确的值，但elements属性引用与原始Stack实例相同的数组。 
修改原始实例将破坏克隆中的不变量，反之亦然。 
你会很快发现你的程序产生了无意义的结果，或者抛出NullPointerException异常。

如果你只是调用 Stack 单一的构造方法的话，上述错误永远也不会发生。
实际上，clone方法作为另一种构造方法; 必须确保它不会损坏原始对象，并且可以在克隆上正确建立不变量。 
为了使Stack上的clone方法正常工作，它必须复制stack 对象的内部。 最简单的方法是对元素数组递归调用clone方法：
```java
// Clone method for class with references to mutable state
@Override public Stack clone() {
    try {
        Stack result = (Stack) super.clone();
        result.elements = elements.clone();
        return result;
    } catch (CloneNotSupportedException e) {
        throw new AssertionError();
    }
}
```
请注意，我们不必将elements.clone的结果转换为Object[]数组。
在数组上调用clone会返回一个数组，其运行时和编译时类型与被克隆的数组相同。 
这是复制数组的首选习语。 
事实上，数组是clone 机制的唯一有力的用途。

还要注意，如果elements属性是final的，则以前的解决方案将不起作用，因为克隆将被禁止向该属性分配新的值。
这是一个基本的问题：像序列化一样，Cloneable体系结构与引用可变对象的final 属性的正常使用不兼容，除非可变对象可以在对象和其克隆之间安全地共享。 
为了使一个类可以克隆，可能需要从一些属性中移除 final修饰符。

仅仅递归地调用clone方法并不总是足够的。 
例如，假设您正在为哈希表编写一个clone方法，其内部包含一个哈希桶数组，每个哈希桶都指向“键-值”对链表的第一项。 
为了提高性能，该类实现了自己的轻量级单链表，而没有使用java内部提供的java.util.LinkedList：
```java
public class HashTable implements Cloneable {
    private Entry[] buckets = ...;
    private static class Entry {
        final Object key;
        Object value;
        Entry  next;

        Entry(Object key, Object value, Entry next) {
            this.key   = key;
            this.value = value;
            this.next  = next;  
        }
    }
    ... // Remainder omitted
}
```
假设你只是递归地克隆哈希桶数组，就像我们为Stack所做的那样：
```java
// Broken clone method - results in shared mutable state!
@Override public HashTable clone() {
    try {
        HashTable result = (HashTable) super.clone();
        result.buckets = buckets.clone();
        return result;
    } catch (CloneNotSupportedException e) {
        throw new AssertionError();
    }
}
```
虽然被克隆的对象有自己的哈希桶数组，但是这个数组引用与原始数组相同的链表，这很容易导致克隆对象和原始对象中的不确定性行为。 
要解决这个问题，你必须复制包含每个桶的链表。 下面是一种常见的方法：
```java
// Recursive clone method for class with complex mutable state
public class HashTable implements Cloneable {
    private Entry[] buckets = ...;

    private static class Entry {
        final Object key;
        Object value;
        Entry  next;

        Entry(Object key, Object value, Entry next) {
            this.key   = key;
            this.value = value;
            this.next  = next;  
        }

        // Recursively copy the linked list headed by this Entry
        Entry deepCopy() {
            return new Entry(key, value,
                next == null ? null : next.deepCopy());
        }
    }

    @Override public HashTable clone() {
        try {
            HashTable result = (HashTable) super.clone();
            result.buckets = new Entry[buckets.length];
            for (int i = 0; i < buckets.length; i++)
                if (buckets[i] != null)
                    result.buckets[i] = buckets[i].deepCopy();
            return result;
        } catch (CloneNotSupportedException e) {
            throw new AssertionError();
        }
    }
    ... // Remainder omitted
}
```
私有类HashTable.Entry已被扩充以支持“深度复制”方法。 
HashTable上的clone方法分配一个合适大小的新哈希桶数组，迭代原来哈希桶数组，深度复制每个非空的哈希桶。 
Entry上的deepCopy方法递归地调用它自己以复制由头节点开始的整个链表。 
如果哈希桶不是太长，这种技术很聪明并且工作正常。
但是，克隆链表不是一个好方法，因为它为列表中的每个元素消耗一个栈帧（stack frame）。 
如果列表很长，这很容易导致堆栈溢出。 
为了防止这种情况发生，可以用迭代来替换deepCopy中的递归：
```java
// Iteratively copy the linked list headed by this Entry
Entry deepCopy() {
   Entry result = new Entry(key, value, next);
   for (Entry p = result; p.next != null; p = p.next)
      p.next = new Entry(p.next.key, p.next.value, p.next.next);
   return result;
}
```
克隆复杂可变对象的最后一种方法是调用super.clone，将结果对象中的所有属性设置为其初始状态，然后调用更高级别的方法来重新生成原始对象的状态。 
以HashTable为例，bucket属性将被初始化为一个新的bucket数组，并且 put(key, value)方法（未示出）被调用用于被克隆的哈希表中的键值映射。 
这种方法通常产生一个简单，合理的优雅clone方法，其运行速度不如直接操纵克隆内部的方法快。 
虽然这种方法是干净的，但它与整个Cloneable体系结构是对立的，因为它会盲目地重写构成体系结构基础的逐个属性对象复制。

与构造方法一样，clone 方法绝对不可以在构建过程中，调用一个可以重写的方法（Item 19）。
如果 clone 方法调用一个在子类中重写的方法，则在子类有机会在克隆中修复它的状态之前执行该方法，很可能导致克隆和原始对象的损坏。
因此，我们在前面讨论的 put(key, value)方法应该时 final 或 private 修饰的。
（如果时 private 修饰，那么大概是一个非 final 公共方法的辅助方法）。

Object 类的 clone方法被声明为抛出CloneNotSupportedException异常，但重写方法时不需要。 
公共clone方法应该省略throws子句，因为不抛出检查时异常的方法更容易使用（Item 71）。

在为继承设计一个类时（Item 19），通常有两种选择，但无论选择哪一种，都不应该实现 Clonable 接口。
你可以选择通过实现正确运行的受保护的 clone方法来模仿Object的行为，该方法声明为抛出CloneNotSupportedException异常。 
这给了子类实现Cloneable接口的自由，就像直接继承Object一样。 
或者，可以选择不实现工作的 clone方法，并通过提供以下简并clone实现来阻止子类实现它：
```java
// clone method for extendable class not supporting Cloneable
@Override
protected final Object clone() throws CloneNotSupportedException {
    throw new CloneNotSupportedException();
}
```
还有一个值得注意的细节。 
如果你编写一个实现了Cloneable的线程安全的类，记得它的clone方法必须和其他方法一样（条目 78）需要正确的同步。 
Object 类的clone方法是不同步的，所以即使它的实现是令人满意的，也可能需要编写一个返回super.clone()的同步clone方法。

回顾一下，实现Cloneable的所有类应该重写公共clone方法，而这个方法的返回类型是类本身。 
这个方法应该首先调用super.clone，然后修复任何需要修复的属性。 
通常，这意味着复制任何包含内部“深层结构”的可变对象，并用指向新对象的引用来代替原来指向这些对象的引用。
虽然这些内部拷贝通常可以通过递归调用clone来实现，但这并不总是最好的方法。 
如果类只包含基本类型或对不可变对象的引用，那么很可能是没有属性需要修复的情况。 
这个规则也有例外。 
例如，表示序列号或其他唯一ID的属性即使是基本类型的或不可变的，也需要被修正。

这么复杂是否真的有必要？很少。 
如果你继承一个已经实现了Cloneable接口的类，你别无选择，只能实现一个行为良好的clone方法。 
否则，通常你最好提供另一种对象复制方法。 
对象复制更好的方法是提供一个复制构造方法或复制工厂。 
复制构造方法接受参数，其类型为包含此构造方法的类，例如，
```java
// Copy constructor
public Yum(Yum yum) { ... };
复制工厂类似于复制构造方法的静态工厂：

// Copy factory
public static Yum newInstance(Yum yum) { ... };
```
复制构造方法及其静态工厂变体与Cloneable/clone相比有许多优点：它们不依赖风险很大的语言外的对象创建机制；不要求遵守那些不太明确的惯例；不会与final 属性的正确使用相冲突; 不会抛出不必要的检查异常; 而且不需要类型转换。

此外，复制构造方法或复制工厂可以接受类型为该类实现的接口的参数。 
例如，按照惯例，所有通用集合实现都提供了一个构造方法，其参数的类型为Collection或Map。 
基于接口的复制构造方法和复制工厂（更适当地称为转换构造方法和转换工厂）允许客户端选择复制的实现类型，而不是强制客户端接受原始实现类型。 
例如，假设你有一个HashSet，并且你想把它复制为一个TreeSet。 clone方法不能提供这种功能，但使用转换构造方法很容易：new TreeSet<>(s)。

考虑到与Cloneable接口相关的所有问题，新的接口不应该继承它，新的可扩展类不应该实现它。 
虽然实现Cloneable接口对于final类没有什么危害，但应该将其视为性能优化的角度，仅在极少数情况下才是合理的（Item 67）。 
通常，复制功能最好由构造方法或工厂提供。 这个规则的一个明显的例外是数组，它最好用 clone方法复制。

---

# Item 14: 考虑实现 Comparable 接口

与本章中讨论的其他方法不同，compareTo 方法未在 Object 中声明。 
相反，它是 Comparable 接口中的唯一方法。 
它的特征与 Object 的 equals 方法类似，不同之处在于除了简单的相等比较之外它还允许顺序比较，并且它是通用的。 
通过实现Comparable，类指示其实例具有自然顺序。 
对实现 Comparable 的对象数组进行排序就像这样简单：

```java
Arrays.sort(a);
```

它同样易于搜索，计算极值，并维护自动排序的 Comparable 对象集合。 
例如，以下程序依赖于 String 实现了 Comparable，打印其命令行参数的按字母顺序排列的列表，并删除重复项：
```java
public class WordList {
    public static void main(String[] args) {
        Set<String> s = new TreeSet<>();
        Collections.addAll(s, args);
        System.out.println(s);
    }
}
```
通过实现Comparable，您可以让您的类与依赖于此接口的所有许多通用算法和集合实现进行互操作。 
只需少量努力即可获得巨大的动力。 
实际上，Java平台库中的所有值类以及所有枚举类型（第34项）都实现了Comparable。 
如果您正在编写具有明显自然顺序的值类，例如字母顺序，数字顺序或时间顺序，则应实现Comparable接口：
```java
public interface Comparable<T> {
    int compareTo(T t);
}
```

## compareTo方法的常规协定

类似于equals：

将此对象与指定的对象进行比较以获得顺序。 
返回负整数，零或正整数，因为此对象小于，等于或大于指定对象。 
如果指定对象的类型阻止将其与此对象进行比较，则抛出ClassCastException。

在以下描述中，符号sgn（表达式）指定数学符号函数，其被定义为根据表达式的值是负，零还是正而返回-1,0或1。
- 实现者必须保证 sgn(x.compareTo(y)) == -sgn(y.compareTo(x)) 适用于所有 x 和 y。 
    如果 x.compareTo(y) 抛出异常，那么 y.compareTo(x) 也应该抛出异常。
- 同时实现者也应该保证关系是可传递的： (x.compareTo(y) > 0 && y.compareTo(z) > 0) 暗示着 x.compareTo(z) > 0
- 最后，实现者必须保证 x.compareTo(y) == 0 暗示着对于所有 z， sgn(x.compareTo(z)) == sgn(y.compareTo(z))都成立。
- 强烈推荐，但不是必须的， (x.compareTo(y) == 0) == (x.equals(y))。
    通常来讲，任何实现了 Comparable 接口并违反此条件的类都应清楚地表明这一点事实。
    推荐的语言是“注意：此类具有与equals不一致的自然顺序。”
    
不要因为这份约定的数学性质而推迟它。 
就像 equals 一样，这份约定并不像看起来那么复杂。 
与equals方法不同，equals方法在所有对象上强加全局等价关系，compareTo不必跨越不同类型的对象：
当遇到不同类型的对象时，允许compareTo抛出ClassCastException。 
通常，这正是它的作用。 
约定确实允许交互式比较，这通常在由被比较的对象实现的接口中定义。

正如违反 hashCode 契约的类可以破坏依赖于哈希的其他类一样，违反 compareTo 契约的类可能会破坏依赖于比较的其他类。 
依赖于比较的类包括已排序的集合 TreeSet 和 TreeMap 以及包含搜索和排序算法的实用程序集 Collections 和 Arrays。

我们来看看compareTo合同的规定。 

1. 第一条规定说如果你反转两个对象引用之间的比较方向，就会发生预期的事情：
如果第一个对象小于第二个对象，那么第二个对象必须大于第一个对象; 
如果第一个对象等于第二个对象，那么第二个对象必须等于第一个对象; 
如果第一个对象大于第二个对象，那么第二个对象必须小于第一个对象。 
2. 第二条规定如果一个对象大于第二个对象，而第二个对象大于第三个对象，那么第一个对象必须大于第三个对象。
3. 最后的规定说，与任何其他对象相比，所有比较相等的对象必须产生相同的结果。

这三个规定的一个结果是，compareTo 方法所施加的相等性测试必须遵守 equals 合约所施加的相同限制：反身性，对称性和传递性。
因此，同样的警告适用：
除非您愿意放弃面向对象抽象的好处（第10项），否则无法在保留compareTo契约的情况下使用新值组件扩展可实例化类。
同样的解决方法也适用。 
如果要将值组件添加到实现Comparable的类中，请不要扩展它; 
编写一个包含第一个类实例的无关类。 
然后提供一个返回包含实例的“view”方法。 
这使您可以在包含的类上实现您喜欢的任何compareTo方法，同时允许其客户端在需要时查看包含类的实例作为包含类的实例。

compareTo契约的最后一段是一个强烈的建议，而不是一个真正的要求，只是说明compareTo方法所施加的相等测试通常应该返回与equals方法相同的结果。 
如果遵守此规定，则compareTo方法强加的顺序与equals一致。 
如果它被违反，则说明顺序与equals不一致。 
compareTo方法强加与equals不一致的顺序的类仍然有效，但包含该类元素的有序集合可能不遵守相应集合接口（Collection，Set或Map）的常规协定。 
这是因为这些接口的一般契约是根据equals方法定义的，但是有序集合使用compareTo强加的相等性测试来代替equals。 
如果发生这种情况，虽然不是灾难，但是仍然需要注意。

例如，考虑BigDecimal类，其compareTo方法与equals不一致。 
如果您创建一个空的HashSet实例，然后添加新的BigDecimal（“1.0”）和新的BigDecimal（“1.00”），该集将包含两个元素，因为使用equals方法比较时，添加到集合中的两个BigDecimal实例是不相等的。 
但是，如果使用TreeSet而不是HashSet执行相同的过程，则该集合将只包含一个元素，因为使用compareTo方法比较时，两个BigDecimal实例相等。 （有关详细信息，请参阅BigDecimal文档。）

编写compareTo方法与编写equals方法类似，但存在一些关键差异。 
因为Comparable接口是参数化的，所以compareTo方法是静态类型的，因此您不需要键入check或cast其参数。 
如果参数的类型错误，则调用甚至不会编译。 
如果参数为null，则调用应抛出NullPointerException，并且只要方法尝试访问其成员，它就会抛出。

在compareTo方法中，比较字段而不是相等。 
要比较对象引用字段，请递归调用compareTo方法。 
如果某个字段未实现Comparable或您需要非标准排序，请改用Comparator。 
您可以编写自己的比较器或使用现有的比较器，如第10项中的CaseInsensitiveString的compareTo方法：
```java
// Single-field Comparable with object reference field
public final class CaseInsensitiveString
        implements Comparable<CaseInsensitiveString> {
    public int compareTo(CaseInsensitiveString cis) {
        return String.CASE_INSENSITIVE_ORDER.compare(s, cis.s);
    }
    ... // Remainder omitted
}
```

注意 CaseInsensitiveString 实现了 Comparable<CaseInsensitiveString> 接口。
这意味着 CaseInsensitiveString  只能和另一个 CaseInsensitiveString 进行比较。
这是声明一个类来实现 Comparable 时要遵循的正常模式。

## Java 7 写法

本书的先前版本建议compareTo方法使用关系运算符 <and> 比较整数基元字段，使用静态方法Double.compare和Float.compare比较浮点基元字段。 
在Java 7中，静态比较方法被添加到所有Java的盒装基元类中。 
在compareTo方法中使用关系运算符 <and> 是冗长且容易出错的，不再推荐使用。

如果一个类有多个重要字段，那么比较它们的顺序至关重要。 
从最重要的领域开始，向下工作。 
如果比较产生的不是零（代表相等），那么你就完成了; 只返回结果。 
如果最重要的字段相等，则比较次最重要的字段，依此类推，直到找到不相等的字段或比较最不重要的字段。 
这是第11项中PhoneNumber类的compareTo方法，演示了这种技术：

```java
// Multiple-field Comparable with primitive fields
public int compareTo(PhoneNumber pn) {
    int result = Short.compare(areaCode, pn.areaCode);
    if (result == 0) {
        result = Short.compare(prefix, pn.prefix);
        if (result == 0)
            result = Short.compare(lineNum, pn.lineNum);
    }
    return result;
}
```

## Java 8 写法

在Java 8中，Comparator接口配备了一组比较器构造方法，可以精确构建比较器。 
然后，可以使用这些比较器来实现compareTo方法，这是Comparable接口所要求的。 
许多程序员更喜欢这种方法的简洁性，虽然它确实以适度的性能成本：在我的机器上排序PhoneNumber实例的数组大约慢10％。 
使用此方法时，请考虑使用Java的静态导入工具，以便您可以通过简单的名称来引用静态比较器构造方法，以简化和简洁。 
以下是使用此方法的PhoneNumber的compareTo方法的外观：

```java
// Comparable with comparator construction methods
private static final Comparator<PhoneNumber> COMPARATOR =
        comparingInt((PhoneNumber pn) -> pn.areaCode)
            .thenComparingInt(pn -> pn.prefix)
            .thenComparingInt(pn -> pn.lineNum);

public int compareTo(PhoneNumber pn) {
    return COMPARATOR.compare(this, pn);
}
```

此实现中在类初始化时，构建了一个比较器（comparator），它使用了两种比较器构造方法。 
第一个是 comparingInt。 
它是一个静态方法，它接受一个 key 提取器函数，它将对象引用映射到int类型的键，并返回一个比较器，该比较器根据该键对实例进行排序。 
在前面的示例中，comparisonInt 采用 lambda 从 PhoneNumber 中提取区域代码，并返回 Comparator<PhoneNumber>，根据区号对电话号码进行排序。
请注意，lambda 显式指定其输入参数的类型（PhoneNumber pn）。 
事实证明，在这种情况下，Java的类型推断并不足以为自己确定类型，因此我们不得不帮助它来编译程序。

如果两个电话号码具有相同的区号，我们需要进一步细化比较，这正是第二个比较器构造方法 thenComparingInt 所做的事。
它是 Comparator 上的一个实例方法，它接受一个int key提取器函数，并返回一个比较器，该比较器首先应用原始比较器，然后使用提取的键来断开关系。 
您可以根据需要将尽可能多的调用堆叠到 thenComparingInt，从而产生字典顺序。 
在上面的示例中，我们将两个调用堆叠到 thenComparingInt，从而产生一个排序，其二级排序字段是 prefix ，其三级排序字段是 lineNum。 
请注意，我们没有必要指定传递给 thenComparingInt 的任一调用的键提取器函数的参数类型：Java的类型推断足够聪明，可以自己解决这个问题。

Comparator类具有完整的构造方法。
有 comparingInt 和 thenComparingInt 的类似方法来比较基本类型 long 和 double。 
int版本也可以用于较窄的整数类型，例如short，如我们的PhoneNumber示例中所示。double 也可用于 float。
它提供了所有Java的数字基元类型的覆盖。

还有对象引用类型的比较器构造方法。
名为comparison的静态方法有两个重载。
一个需要一个 key 的提取器，并使用键的自然顺序。
第二个采用 key 提取器和比较器来提取 key。
实例方法有三个重载，命名为thenComparing。
一次重载仅使用比较器并使用它来提供二级排序。
第二次重载仅使用一个 key 提取器，并使用 key 的自然顺序作为二级排序。
最后的重载需要一个 key 提取器和一个比较器，用于提取的键。

有时，您可能会看到 compareTo 或比较方法，这些方法依赖于以下事实：
如果第一个值小于第二个值，则两个值之间的差值为负，如果两个值相等则为零，如果第一个值更大则为正值。
这是一个例子：

```java
// BROKEN difference-based comparator - violates transitivity!
static Comparator<Object> hashCodeOrder = new Comparator<>() {
    public int compare(Object o1, Object o2) {
        return o1.hashCode() - o2.hashCode();
    }
};
```

不要使用这种技术。 
它充满了整数溢出和IEEE 754浮点运算伪像的危险[JLS 15.20.1,15.21.1]。 
此外，所得到的方法不太可能比使用本项目中描述的技术编写的方法快得多。 
使用静态比较方法：
```java
// Comparator based on static compare method
static Comparator<Object> hashCodeOrder = new Comparator<>() {
    public int compare(Object o1, Object o2) {
        return Integer.compare(o1.hashCode(), o2.hashCode());
    }
};
```
或者一个 comparator 构造方法：

```java
// Comparator based on Comparator construction method
static Comparator<Object> hashCodeOrder = Comparator.comparingInt(o -> o.hashCode());
```

总之，无论何时实现具有合理排序的值类，都应该让类实现Comparable接口，以便可以在基于比较的集合中轻松地对其实例进行排序，搜索和使用。 
比较compareTo方法实现中的字段值时，请避免使用<and>运算符。 
而是使用盒装基元类中的静态比较方法或比较器接口中的比较器构造方法。
