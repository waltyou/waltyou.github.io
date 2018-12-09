---
layout: post
title: 《Effective Java》学习日志（七）52：谨慎的使用重载
date: 2018-12-06 19:53:04
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

以下程序是一种善意的尝试，去试着区分集合是set，list还是其他类型的集合：

    // Broken! - What does this program print?
    public class CollectionClassifier {
        public static String classify(Set<?> s) {
            return "Set";
        }
        public static String classify(List<?> lst) {
            return "List";
        }
        public static String classify(Collection<?> c) {
            return "Unknown Collection";
        }
        public static void main(String[] args) {
            Collection<?>[] collections = {
                new HashSet<String>(),
                new ArrayList<BigInteger>(),
                new HashMap<String, String>().values()
            };
            
            for (Collection<?> c : collections)
                System.out.println(classify(c));
        }
    }
    
您可能希望此程序打印Set，然后是List和Unknown Collection，但它不会。
它打印三次Unknown Collection。
为什么会这样？因为classify方法被重载了，**而哪个重载会被调用是在编译时确定的**。
对于循环的所有三次迭代，参数的编译时类型是相同的：Collection <？>。
每次迭代的运行时类型都不同，但这不会影响重载的选择。
因为参数的编译时类型是Collection <？>，所以唯一适用的重载是第三个，classify（Collection <？>），并且在循环的每次迭代中调用这个重载。

此程序的行为是违反直觉的，**因为重载方法之间的选择是静态的，而重写方法之间的选择是动态的**。
根据调用方法的对象的运行时类型，在运行时选择正确版本的重写方法。
作为提醒，当子类包含与父类中的方法声明具有相同签名的方法声明时，将覆盖方法。
如果在子类中重写实例方法并且在子类的实例上调用此方法，则无论子类实例的编译时类型如何，子类的重写方法都会执行。

为了使这个具体，请考虑以下程序：

    class Wine {
        String name() { return "wine"; }
    }
    class SparklingWine extends Wine {
        @Override String name() { return "sparkling wine"; }
    }
    class Champagne extends SparklingWine {
        @Override String name() { return "champagne"; }
    }
    public class Overriding {
        public static void main(String[] args) {
            List<Wine> wineList = List.of(
                new Wine(), new SparklingWine(), new Champagne());
            for (Wine wine : wineList)
                System.out.println(wine.name());
        }
    }

name方法在Wine类中声明，并在子类SparklingWine和Champagne中重写。
正如您所料，此程序打印出葡萄酒，起泡酒和香槟，即使实例的编译时类型在循环的每次迭代中都是Wine。
当调用重写方法时，对象的编译时类型对执行哪个方法没有影响;总是会执行“最具体”的重写方法。
将此与重载进行比较，其中对象的运行时类型对执行的重载没有影响;
选择是在编译时完成的，完全基于参数的编译时类型。

在CollectionClassifier示例中，程序的目的是通过基于参数的运行时类型自动调度到适当的方法重载来辨别参数的类型，就像Wine方法中的名称方法一样。
方法重载根本不提供此功能。
假设需要一个静态方法，修复CollectionClassifier程序的最佳方法是用一个执行显式instanceof测试的方法替换classify的所有三个重载：

    public static String classify(Collection<?> c) {
        return c instanceof Set ? "Set" :
            c instanceof List ? "List" : "Unknown Collection";
    }

因为重写是常态，而重载是例外，所以重写设置了人们对方法调用行为的期望。
正如CollectionClassifier示例所示，重载很容易混淆这些期望。
编写行为可能会使程序员感到困惑的代码是不好的做法。
对于API尤其如此。如果API的典型用户不知道将为给定的参数集调用多个方法重载中的哪一个，则使用API​​可能会导致错误。
这些错误很可能表现为运行时的不稳定行为，许多程序员很难诊断它们。因此，**您应该避免混淆使用重载**。

对于重载混淆使用的原因，是有一些争论的。
**安全，保守的策略永远不会产出具有相同数量参数的两个重载。**
如果方法使用varargs，保守策略根本不会使其重载，除非在第53项中描述。
如果遵守这些限制，程序员将永远不会怀疑哪些重载适用于任何实际参数集。
**这些限制并不是非常繁重，因为您始终可以为方法提供不同的名称而不是重载它们。**

例如，考虑ObjectOutputStream类。
它为每种基本类型和几种引用类型提供了write方法的变体。
这些变体不是重载write方法，而是具有不同的名称，例如writeBoolean（boolean），writeInt（int）和writeLong（long）。
与重载相比，此命名模式的另一个好处是可以为读取方法提供相应的名称，例如readBoolean（），readInt（）和readLong（）。
事实上，ObjectInputStream类提供了这样的读取方法。

对于构造函数，您没有使用不同名称的选项：类的多个构造函数总是被重载。
在许多情况下，您可以选择导出静态工厂而不是构造函数（第1项）。
此外，使用构造函数，您不必担心重载和重写之间的交互，因为构造函数不能被覆盖。
您可能有机会使用相同数量的参数导出多个构造函数，因此知道如何安全地执行它是值得的。

如果始终清楚哪些重载适用于任何给定的实际参数集，则导出具有相同数量参数的多个重载不太可能使程序员感到困惑。
当两对重载中的至少一个相应的形式参数在两个重载中具有“完全不同”类型时就是这种情况。
如果显然不可能将任何非空表达式转换为两种类型，则两种类型完全不同。
在这些情况下，哪些重载适用于给定的一组实际参数完全由参数的运行时类型决定，并且不受其编译时类型的影响，因此混淆的主要原因消失了。

例如，ArrayList有一个带有int的构造函数和带有Collection的第二个构造函数。
很难想象在任何情况下都会混淆这两个构造函数中的哪一个。
在Java 5之前，所有原始类型都与所有引用类型完全不同，但在自动装箱存在的情况下并非如此，并且它已经造成了真正的麻烦。
考虑以下程序：

    public class SetList {
        public static void main(String[] args) {
            Set<Integer> set = new TreeSet<>();
            List<Integer> list = new ArrayList<>();
            for (int i = -3; i < 3; i++) {
                set.add(i);
                list.add(i);
            }
            for (int i = 0; i < 3; i++) {
                set.remove(i);
                list.remove(i);
            }
            System.out.println(set + " " + list);
        }
    }

首先，程序将从-3到2的整数添加到有序set和列表中。 
然后，它在集合和列表上进行三次相同的调用。 
如果你和大多数人一样，你希望程序从集合和列表中删除非负值（0,1和2）并打印[-3，-2，-1], [-3，-2，-1]。 
实际上，程序从集合中删除非负值，从列表中删除奇数值并打印[-3，-2，-1] [-2,0,2]。 
称这种行为令人困惑是一种保守的说法。

这是正在发生的事情：调用set.remove（i）选择重载remove（E），其中E是集合的元素类型（Integer），以及从int到Integer的autobox。 
这是您期望的行为，因此程序最终会从集合中删除正值。 
另一方面，对list.remove（i）的调用选择重载remove（int i），它删除列表中指定位置的元素。 
如果你从列表[-3，-2，-1,0,1,2]开始并删除第0个元素，那么第一个，然后是第二个，你留下[-2,0,2] ，这个谜就解决了。 
要解决此问题，请将list.remove的参数强制转换为Integer，强制选择正确的重载。 或者
，您可以在i上调用Integer.valueOf并将结果传递给list.remove。 
无论哪种方式，程序按预期打印[-3，-2，-1] [-3，-2，-1]：

    for (int i = 0; i < 3; i++) {
        set.remove(i);
        list.remove((Integer) i);
    }

之前的示例演示了令人困惑的行为，因为List <E>接口有两个remove方法：remove（E）和remove（int）。 
在Java 5之前，当List接口被“生成”时，它有一个remove（Object）方法代替remove（E），相应的参数类型Object和int完全不同。 
但是在存在泛型和自动装箱的情况下，两种参数类型不再完全不同。 
换句话说，向该语言添加泛型和自动装箱会损坏List接口。 
幸运的是，很少有Java库中的其他API被类似地损坏，但是这个故事清楚地表明自动装箱和泛型增加了重载时的注意事项的重要性。

Java 8中添加lambdas和方法引用进一步增加了重载混淆的可能性。 
例如，考虑以下两个片段：

    new Thread(System.out::println).start();
    ExecutorService exec = Executors.newCachedThreadPool();
    exec.submit(System.out::println);

虽然Thread构造函数调用和submit方法调用看起来类似，但前者编译而后者不编译。
参数是相同的（System.out :: println），构造函数和方法都有一个带有Runnable的重载。
这里发生了什么？令人惊讶的答案是，submit方法有一个带有Callable <T>的重载，而Thread构造函数却没有。
您可能认为这不应该有任何区别，因为println的所有重载都返回void，因此方法引用不可能是Callable。
这很有道理，但这不是重载解析算法的工作方式。
也许同样令人惊讶的是，如果println方法也没有重载，则submit方法调用将是合法的。
它是引用方法（println）和调用方法（submit）的重载的组合，它可以防止重载决策算法按照您的预期运行。

从技术上讲，问题是System.out::println是一个不精确的方法引用[JLS，15.13.1]，
并且“包含隐式类型的lambda表达式或不精确的方法引用的某些参数表达式被适用性测试忽略，因为它们的在选择目标类型之前无法确定其含义[JLS，15.12.2]。
“如果您不理解这段经文，请不要担心;它针对的是编译器编写者。
关键是在相同的参数位置中具有不同功能接口的重载方法或构造函数会导致混淆。
**因此，不要重载方法以在同一参数位置采用不同的功能接口。**
在这个Item的说法中，不同的功能接口并没有根本不同。
如果您传递命令行开关-Xlint：overloads，Java编译器将警告您这种有问题的重载。

除Object之外的数组类型和类类型完全不同。
此外，Serializable和Cloneable以外的数组类型和接口类型完全不同。
如果两个类都不是另一个类的后代，那么两个不同的类被认为是无关的[JLS，5.5]。
例如，String和Throwable是无关的。任何对象都不可能是两个不相关的类的实例，所以不相关的类也是根本不同的。

还有其他一对类型无法在任何一个方向上进行转换[JLS，5.1.12]，但是一旦超出上述简单情况，大多数程序员就很难辨别哪些（如果有的话）重载适用一组实际参数。
确定选择哪个重载的规则非常复杂，并且每个版本都会变得越来越复杂。
很少有程序员能够理解他们所有的细微之处。

有时您可能觉得有必要违反本条款中的指导原则，特别是在改进现有课程时。
例如，考虑String，自Java 4以来已经有一个contentEquals（StringBuffer）方法。
在Java 5中，添加了CharSequence以提供StringBuffer，StringBuilder，String，CharBuffer和其他类似类型的通用接口。
在添加CharSequence的同时，String配备了一个带有CharSequence的contentEquals方法的重载。

虽然导致的重载明显违反了此项中的准则，但它不会造成任何伤害，因为重载方法在同一对象引用上调用它们时完全相同。
程序员可能不知道将调用哪个重载，但只要它们的行为相同，它就没有任何意义。
确保此行为的标准方法是将更具体的重载转发到更一般的：

    // Ensuring that 2 methods have identical behavior by forwarding
    public boolean contentEquals(StringBuffer sb) {
        return contentEquals((CharSequence) sb);
    }

虽然Java库很大程度上遵循了这个Item中的建议精神，但是有许多类违反了它。
例如，String导出两个重载的静态工厂方法valueOf（char[]）和 valueOf（Object），它们在传递相同的对象引用时会执行完全不同的操作。
对此没有任何正当理由，它应被视为具有真正混淆可能性的异常现象。

总结一下，仅仅因为你可以重载方法并不意味着你应该。
通常最好不要使用具有相同参数数量的多个签名来重载方法。
在某些情况下，尤其是涉及构造函数的情况下，可能无法遵循此建议。
在这些情况下，您应该至少避免通过添加强制转换将相同的参数集传递给不同的重载的情况。
如果无法避免这种情况，例如，因为您正在改进现有类以实现新接口，则应确保在传递相同参数时所有重载的行为都相同。
如果你不这样做，程序员将很难有效地使用重载的方法或构造函数，他们将无法理解为什么它不起作用。
    