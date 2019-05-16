---
layout: post
title: 《Effective Java》学习日志（五）34：使用枚举而不是int常量
date: 2018-11-17 18:08:04
author: admin
comments: true
categories: [Java]
tags: [Java,Effective Java]
---

接下来，进入了新的一大章节：枚举与注释。

JAVA支持两个特殊用途的引用类型系列：一种称为枚举类型的类，以及一种称为注释类型的接口。

先来看看第一个：使用枚举而不是int常量。

<!-- more -->

---

学习资料主要参考： 《Effective Java Third Edition》，作者：Joshua Bloch

---




* 目录
{:toc}

---

# 枚举

枚举是其合法值由一组固定的常量组成的一种类型，例如一年中的季节，太阳系中的行星或一副扑克牌中的套装。 

在将枚举类型添加到该语言之前，表示枚举类型的常见模式是声明一组名为int的常量，每个类型的成员都有一个常量：

    // The int enum pattern - severely deficient!
    public static final int APPLE_FUJI         = 0;
    public static final int APPLE_PIPPIN       = 1;
    public static final int APPLE_GRANNY_SMITH = 2;
    public static final int ORANGE_NAVEL  = 0;
    public static final int ORANGE_TEMPLE = 1;
    public static final int ORANGE_BLOOD  = 2;
    
这种被称为int枚举模式的技术有许多缺点。 
它没有提供类型安全的方式，也没有提供任何表达力。 
如果你将一个Apple传递给一个需要Orange的方法，那么编译器不会出现警告，还会用==运算符比较Apple与Orange，或者更糟糕的是：

    // Tasty citrus flavored applesauce!
    int i = (APPLE_FUJI - ORANGE_TEMPLE) / APPLE_PIPPIN;
    
请注意，每个Apple常量的名称前缀为APPLE_，每个Orange常量的名称前缀为ORANGE_。 
这是因为Java不为int枚举组提供名称空间。 

当两个int枚举组具有相同的命名常量时，前缀可以防止名称冲突，例如在ELEMENT_MERCURY和PLANET_MERCURY之间。

**使用int枚举的程序很脆弱**。 

因为int枚举是编译时常量[JLS，4.12.4]，所以它们的int值被编译到使用它们的客户端中[JLS，13.1]。 
如果与int枚举关联的值发生更改，则必须重新编译其客户端。 
如果没有，客户仍然会运行，但他们的行为将是不正确的。

没有简单的方法将int枚举常量转换为可打印的字符串。 
如果你打印这样一个常量或者从调试器中显示出来，你看到的只是一个数字，这不是很有用。 
没有可靠的方法来迭代组中的所有int枚举常量，甚至无法获得int枚举组的大小。

你可能会遇到这种模式的变体，其中使用了字符串常量来代替int常量。 这
种称为字符串枚举模式的变体更不理想。 
尽管它为常量提供了可打印的字符串，但它可以导致初级用户将字符串常量硬编码为客户端代码，而不是使用属性名称。 
如果这种硬编码的字符串常量包含书写错误，它将在编译时逃脱检测并导致运行时出现错误。 
此外，它可能会导致性能问题，因为它依赖于字符串比较。

幸运的是，Java提供了一种避免int和String枚举模式的所有缺点的替代方法，并提供了许多额外的好处。 
它是枚举类型[JLS，8.9]。 以下是它最简单的形式：

    public enum Apple  { FUJI, PIPPIN, GRANNY_SMITH }
    public enum Orange { NAVEL, TEMPLE, BLOOD }
    
从表面上看，这些枚举类型可能看起来与其他语言类似，比如C，C ++和C＃，但事实并非如此。 
Java的枚举类型是完整的类，比其他语言中的其他语言更强大，**其枚举本质本上是int值**。

Java枚举类型背后的基本思想很简单：
它们是通过公共静态final属性为每个枚举常量导出一个实例的类。 
由于没有可访问的构造方法，枚举类型实际上是final的。 
由于客户既不能创建枚举类型的实例也不能继承它，除了声明的枚举常量外，不能有任何实例。 
换句话说，枚举类型是实例控制的（第6页）。 它们是单例（条目 3）的泛型化，基本上是单元素的枚举。

# 枚举的优点

## 枚举提供了编译时类型的安全性

如果声明一个参数为Apple类型，则可以保证传递给该参数的任何非空对象引用是三个有效Apple值中的一个。 
尝试传递错误类型的值将导致编译时错误，因为会尝试将一个枚举类型的表达式分配给另一个类型的变量，或者使用==运算符来比较不同枚举类型的值。

具有相同名称常量的枚举类型可以和平共存，因为每种类型都有其自己的名称空间。 
可以在枚举类型中添加或重新排序常量，而无需重新编译其客户端，因为导出常量的属性在枚举类型与其客户端之间提供了一层隔离：
常量值不会编译到客户端，因为它们位于int枚举模式中。 
最后，可以通过调用其toString方法将枚举转换为可打印的字符串。

## 允许添加任意方法和属性并实现任意接口

它们提供了所有Object方法的高质量实现（第3章），它们实现了Comparable（条目 14）和Serializable（第12章），
并针对枚举类型的可任意改变性设计了序列化方式。

那么，为什么你要添加方法或属性到一个枚举类型？ 对于初学者，可能想要将数据与其常量关联起来。
 
 例如，我们的Apple和Orange类型可能会从返回水果颜色的方法或返回水果图像的方法中受益。 
 还可以使用任何看起来合适的方法来增强枚举类型。 
 枚举类型可以作为枚举常量的简单集合，并随着时间的推移而演变为全功能抽象。

对于丰富的枚举类型的一个很好的例子，考虑我们太阳系的八颗行星。 
每个行星都有质量和半径，从这两个属性可以计算出它的表面重力。 
从而在给定物体的质量下，计算出一个物体在行星表面上的重量。 

下面是这个枚举类型。 每个枚举常量之后的括号中的数字是传递给其构造方法的参数。 
在这种情况下，它们是地球的质量和半径：

    // Enum type with data and behavior
    public enum Planet {
        MERCURY(3.302e+23, 2.439e6),
        VENUS  (4.869e+24, 6.052e6),
        EARTH  (5.975e+24, 6.378e6),
        MARS   (6.419e+23, 3.393e6),
        JUPITER(1.899e+27, 7.149e7),
        SATURN (5.685e+26, 6.027e7),
        URANUS (8.683e+25, 2.556e7),
        NEPTUNE(1.024e+26, 2.477e7);
    
        private final double mass;           // In kilograms
        private final double radius;         // In meters
        private final double surfaceGravity; // In m / s^2
        // Universal gravitational constant in m^3 / kg s^2
        private static final double G = 6.67300E-11;
    
        // Constructor
        Planet(double mass, double radius) {
            this.mass = mass;
            this.radius = radius;
            surfaceGravity = G * mass / (radius * radius);
        }
    
        public double mass()           { return mass; }
        public double radius()         { return radius; }
        public double surfaceGravity() { return surfaceGravity; }
    
        public double surfaceWeight(double mass) {
            return mass * surfaceGravity;  // F = ma
        }
    }
    
编写一个丰富的枚举类型比如Planet很容易。 
要将数据与枚举常量相关联，请声明实例属性并编写一个构造方法，构造方法带有数据并将数据保存在属性中。 

枚举本质上是不变的，所以所有的属性都应该是final的（条目 17）。 

属性可以是公开的，但最好将它们设置为私有并提供公共访问方法（条目16）。 

在Planet的情况下，构造方法还计算和存储表面重力，但这只是一种优化。 
每当重力被SurfaceWeight方法使用时，它可以从质量和半径重新计算出来，该方法返回它在由常数表示的行星上的重量。

虽然Planet枚举很简单，但它的功能非常强大。 
这是一个简短的程序，它将一个物体在地球上的重量（任何单位），打印一个漂亮的表格，显示该物体在所有八个行星上的重量（以相同单位）：

    public class WeightTable {
       public static void main(String[] args) {
          double earthWeight = Double.parseDouble(args[0]);
          double mass = earthWeight / Planet.EARTH.surfaceGravity();
          for (Planet p : Planet.values())
              System.out.printf("Weight on %s is %f%n",
                                p, p.surfaceWeight(mass));
          }
    }
    
请注意，Planet和所有枚举一样，都有一个静态values方法，该方法以声明的顺序返回其值的数组。 

另请注意，toString方法返回每个枚举值的声明名称，使println和printf可以轻松打印。 
如果你对此字符串表示形式不满意，可以通过重写toString方法来更改它。
 这是使用命令行参数185运行WeightTable程序（不重写toString）的结果：

    Weight on MERCURY is 69.912739
    Weight on VENUS is 167.434436
    Weight on EARTH is 185.000000
    Weight on MARS is 70.226739
    Weight on JUPITER is 467.990696
    Weight on SATURN is 197.120111
    Weight on URANUS is 167.398264
    Weight on NEPTUNE is 210.208751
    
# 从枚举类型中移除一个元素会发生什么

直到2006年，在Java中加入枚举两年之后，冥王星不再是一颗行星。 
这引发了一个问题：“当你从枚举类型中移除一个元素时会发生什么？”

答案是，任何不引用移除元素的客户端程序都将继续正常工作。 

所以，举例来说，我们的WeightTable程序只需要打印一行少一行的表格。 
什么是客户端程序引用删除的元素（在这种情况下，Planet.Pluto）？ 
如果重新编译客户端程序，编译将会失败并在引用前一个星球的行处提供有用的错误消息; 如果无法重新编译客户端，它将在运行时从此行中引发有用的异常。 
这是你所希望的最好的行为，远远好于你用int枚举模式得到的结果。

# 枚举的范围

一些与枚举常量相关的行为只需要在定义枚举的类或包中使用。 
这些行为最好以私有或包级私有方式实现。 
然后每个常量携带一个隐藏的行为集合，允许包含枚举的类或包在与常量一起呈现时作出适当的反应。 
与其他类一样，除非你有一个令人信服的理由将枚举方法暴露给它的客户端，否则将其声明为私有的，如果需要的话将其声明为包级私有（条目 15）。

如果一个枚举是广泛使用的，它应该是一个顶级类; 如果它的使用与特定的顶级类绑定，它应该是该顶级类的成员类（条目 24）。 

例如，java.math.RoundingMode枚举表示小数部分的舍入模式。 
BigDecimal类使用了这些舍入模式，但它们提供了一种有用的抽象，它并不与BigDecimal有根本的联系。 
通过将RoundingMode设置为顶层枚举，类库设计人员鼓励任何需要舍入模式的程序员重用此枚举，从而提高跨API的一致性。

# 几个特殊场景下的写法

## 拒绝switch

Planet示例中演示的技术对于大多数枚举类型都足够，但有时您需要更多。 
每个Planet常量都有不同的数据，但有时您需要将基本不同的行为与每个常量相关联。 

例如，假设您正在编写枚举类型来表示基本四函数计算器上的操作，并且您希望提供一种方法来执行由每个常量表示的算术运算。 
实现此目的的一种方法是switch枚举的值：

    // Enum type that switches on its own value - questionable
    public enum Operation {
        PLUS, MINUS, TIMES, DIVIDE;
    
        // Do the arithmetic operation represented by this constant
        public double apply(double x, double y) {
            switch(this) {
                case PLUS:   return x + y;
                case MINUS:  return x - y;
                case TIMES:  return x * y;
                case DIVIDE: return x / y;
            }
            throw new AssertionError("Unknown op: " + this);
        }
    }
    
此代码有效，但不是很漂亮。 
如果没有throw语句，就不能编译，因为该方法的结束在技术上是可达到的，尽管它永远不会被达到[JLS，14.21]。 
更糟的是，代码很脆弱。 
如果添加新的枚举常量，但忘记向switch语句添加相应的条件，枚举仍然会编译，但在尝试应用新操作时，它将在运行时失败。

幸运的是，有一种更好的方法可以将不同的行为与每个枚举常量关联起来：在枚举类型中声明一个抽象的apply方法，并用常量特定的类主体中的每个常量的具体方法重写它。 

这种方法被称为特定于常量（constant-specific）的方法实现：

    // Enum type with constant-specific method implementations
    public enum Operation {
      PLUS  {public double apply(double x, double y){return x + y;}},
      MINUS {public double apply(double x, double y){return x - y;}},
      TIMES {public double apply(double x, double y){return x * y;}},
      DIVIDE{public double apply(double x, double y){return x / y;}};
    
      public abstract double apply(double x, double y);
    }
    
如果向第二个版本的操作添加新的常量，则不太可能会忘记提供apply方法，因为该方法紧跟在每个常量声明之后。 
万一忘记了，编译器会提醒你，因为枚举类型中的抽象方法必须被所有常量中的具体方法重写。

## toString

特定于常量的方法实现可以与特定于常量的数据结合使用。 

例如，以下是Operation的一个版本，它重写toString方法以返回通常与该操作关联的符号：

    // Enum type with constant-specific class bodies and data
    public enum Operation {
        PLUS("+") {
            public double apply(double x, double y) { return x + y; }
        },
        MINUS("-") {
            public double apply(double x, double y) { return x - y; }
        },
        TIMES("*") {
            public double apply(double x, double y) { return x * y; }
        },
        DIVIDE("/") {
            public double apply(double x, double y) { return x / y; }
        };
        private final String symbol;
        Operation(String symbol) { this.symbol = symbol; }
        @Override public String toString() { return symbol; }
        public abstract double apply(double x, double y);
    }
    
显示的toString实现可以很容易地打印算术表达式，正如这个小程序所展示的那样：

    public static void main(String[] args) {
        double x = Double.parseDouble(args[0]);
        double y = Double.parseDouble(args[1]);
        for (Operation op : Operation.values())
            System.out.printf("%f %s %f = %f%n",
                              x, op, y, op.apply(x, y));
    }
    
以2和4作为命令行参数运行此程序会生成以下输出：

    2.000000 + 4.000000 = 6.000000
    2.000000 - 4.000000 = -2.000000
    2.000000 * 4.000000 = 8.000000
    2.000000 / 4.000000 = 0.500000
    
枚举类型具有自动生成的valueOf(String)方法，该方法将常量名称转换为常量本身。 

## fromString

如果在枚举类型中重写toString方法，请考虑编写fromString方法将自定义字符串表示法转换回相应的枚举类型。 
下面的代码（类型名称被适当地改变）将对任何枚举都有效，只要每个常量具有唯一的字符串表示形式：

    // Implementing a fromString method on an enum type
    private static final Map<String, Operation> stringToEnum =
            Stream.of(values()).collect(
                toMap(Object::toString, e -> e));
    
    // Returns Operation for string, if any
    public static Optional<Operation> fromString(String symbol) {
        return Optional.ofNullable(stringToEnum.get(symbol));
    }
    
请注意，Operation枚举常量被放在stringToEnum的map中，它来自于创建枚举常量后运行的静态属性初始化
。前面的代码在values()方法返回的数组上使用流（第7章）；在Java 8之前，我们创建一个空的hashMap并遍历值数组，将字符串到枚举映射插入到map中，如果愿意，仍然可以这样做。

但请注意，尝试让每个常量都将自己放入来自其构造方法的map中不起作用。
这会导致编译错误，这是好事，因为如果它是合法的，它会在运行时导致NullPointerException。
除了编译时常量属性（条目 34）之外，枚举构造方法不允许访问枚举的静态属性。
此限制是必需的，因为静态属性在枚举构造方法运行时尚未初始化。这种限制的一个特例是枚举常量不能从构造方法中相互访问。

另请注意，fromString 方法返回一个Optional<String>。 
这允许该方法指示传入的字符串不代表有效的操作，并且强制客户端面对这种可能性（条目 55）。

## 私有嵌套枚举

特定于常量的方法实现的一个缺点是它们使得难以在枚举常量之间共享代码。 
例如，考虑一个代表工资包中的工作天数的枚举。 
该枚举有一个方法，根据工人的基本工资（每小时）和当天工作的分钟数计算当天工人的工资。 
在五个工作日内，任何超过正常工作时间的工作都会产生加班费; 在两个周末的日子里，所有工作都会产生加班费。 
使用switch语句，通过将多个case标签应用于两个代码片段中的每一个，可以轻松完成此计算：

    // Enum that switches on its value to share code - questionable
    enum PayrollDay {
        MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY,
        SATURDAY, SUNDAY;
    
        private static final int MINS_PER_SHIFT = 8 * 60;
    
        int pay(int minutesWorked, int payRate) {
            int basePay = minutesWorked * payRate;
            int overtimePay;
            switch(this) {
              case SATURDAY: case SUNDAY: // Weekend
                overtimePay = basePay / 2;
                break;
              default: // Weekday
                overtimePay = minutesWorked <= MINS_PER_SHIFT ?
                  0 : (minutesWorked - MINS_PER_SHIFT) * payRate / 2;
            }
    
            return basePay + overtimePay;
        }
    }
    
这段代码无可否认是简洁的，但从维护的角度来看是危险的。 
假设你给枚举添加了一个元素，可能是一个特殊的值来表示一个假期，但忘记在switch语句中添加一个相应的case条件。 
该程序仍然会编译，但付费方法会默默地为工作日支付相同数量的休假日，与普通工作日相同。

要使用特定于常量的方法实现安全地执行工资计算，必须为每个常量重复加班工资计算，或将计算移至两个辅助方法，一个用于工作日，另一个用于周末，并调用适当的辅助方法来自每个常量。 
这两种方法都会产生相当数量的样板代码，大大降低了可读性并增加了出错机会。

通过使用执行加班计算的具体方法替换PayrollDay上的抽象overtimePay方法，可以减少样板。 
那么只有周末的日子必须重写该方法。 但是，这与switch语句具有相同的缺点：如果在不重写overtimePay方法的情况下添加另一天，则会默默继承周日计算方式。

你真正想要的是每次添加枚举常量时被迫选择加班费策略。 

幸运的是，有一个很好的方法来实现这一点。 
这个想法是将加班费计算移入私有嵌套枚举中，并将此策略枚举的实例传递给PayrollDay枚举的构造方法。 
然后，PayrollDay枚举将加班工资计算委托给策略枚举，从而无需在PayrollDay中实现switch语句或特定于常量的方法实现。 
虽然这种模式不如switch语句简洁，但它更安全，更灵活：

    // The strategy enum pattern
    enum PayrollDay {
        MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY,
        SATURDAY(PayType.WEEKEND), SUNDAY(PayType.WEEKEND);
    
        private final PayType payType;
    
        PayrollDay(PayType payType) { this.payType = payType; }
        PayrollDay() { this(PayType.WEEKDAY); }  // Default
    
        int pay(int minutesWorked, int payRate) {
            return payType.pay(minutesWorked, payRate);
        }
    
        // The strategy enum type
        private enum PayType {
            WEEKDAY {
                int overtimePay(int minsWorked, int payRate) {
                    return minsWorked <= MINS_PER_SHIFT ? 0 :
                      (minsWorked - MINS_PER_SHIFT) * payRate / 2;
                }
            },
            WEEKEND {
                int overtimePay(int minsWorked, int payRate) {
                    return minsWorked * payRate / 2;
                }
            };
    
            abstract int overtimePay(int mins, int payRate);
            private static final int MINS_PER_SHIFT = 8 * 60;
    
            int pay(int minsWorked, int payRate) {
                int basePay = minsWorked * payRate;
                return basePay + overtimePay(minsWorked, payRate);
            }
        }
    }
    
## switch 的好处

如果对枚举的switch语句不是实现常量特定行为的好选择，那么它们有什么好处呢?

枚举类型的switch有利于用常量特定的行为增加枚举类型。
例如，假设Operation枚举不在你的控制之下，你希望它有一个实例方法来返回每个相反的操作。你可以用以下静态方法模拟效果:

// Switch on an enum to simulate a missing method
public static Operation inverse(Operation op) {
    switch(op) {
        case PLUS:   return Operation.MINUS;
        case MINUS:  return Operation.PLUS;
        case TIMES:  return Operation.DIVIDE;
        case DIVIDE: return Operation.TIMES;
        default:  throw new AssertionError("Unknown op: " + op);
    }
}

如果某个方法不属于枚举类型，则还应该在你控制的枚举类型上使用此技术。 
该方法可能需要用于某些用途，但通常不足以用于列入枚举类型。

# 总结

一般而言，枚举通常在性能上与int常数相当。 
枚举的一个小小的性能缺点是加载和初始化枚举类型存在空间和时间成本，但在实践中不太可能引人注意。

那么你应该什么时候使用枚举呢？ 
任何时候使用枚举都需要一组常量，这些常量的成员在编译时已知。 

当然，这包括“天然枚举类型”，如行星，星期几和棋子。 
但是它也包含了其它你已经知道编译时所有可能值的集合，例如菜单上的选项，操作代码和命令行标志。**一个枚举类型中的常量集不需要一直保持不变**。 

枚举功能是专门设计用于允许二进制兼容的枚举类型的演变。

总之，枚举类型优于int常量的优点是令人信服的。 
枚举更具可读性，更安全，更强大。 

许多枚举不需要显式构造方法或成员，但其他人则可以通过将数据与每个常量关联并提供行为受此数据影响的方法而受益。 

使用单一方法关联多个行为可以减少枚举。 在这种相对罕见的情况下，更喜欢使用常量特定的方法来枚举自己的值。 
如果一些（但不是全部）枚举常量共享共同行为，请考虑策略枚举模式。