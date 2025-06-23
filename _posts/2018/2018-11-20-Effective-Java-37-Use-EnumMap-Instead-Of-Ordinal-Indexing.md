---
layout: post
title: 《Effective Java》学习日志（五）37：使用EnumMap替代序数索引
date: 2018-11-20 18:29:04
author: admin
comments: true
categories: [Java]
tags: [Java,Effective Java]
---

有时可能会看到使用ordinal方法来索引到数组或列表的代码。最好用 EnumMap 替换。

<!-- more -->

---

学习资料主要参考： 《Effective Java Third Edition》，作者：Joshua Bloch

---




* 目录
{:toc}

---

例如，考虑一下这个简单的类来代表一种植物：

    class Plant {
    
        enum LifeCycle { ANNUAL, PERENNIAL, BIENNIAL }
    
        final String name;
        final LifeCycle lifeCycle;
        
        Plant(String name, LifeCycle lifeCycle) {
            this.name = name;
            this.lifeCycle = lifeCycle;
        }
        
        @Override public String toString() {
            return name;
        }
    }
    
现在假设你有一组植物代表一个花园，想要列出这些由生命周期组织的植物(一年生，多年生，或双年生)。
为此，需要构建三个集合，每个生命周期作为一个，并遍历整个花园，将每个植物放置在适当的集合中。
一些程序员可以通过将这些集合放入一个由生命周期序数索引的数组中来实现这一点：

    // Using ordinal() to index into an array - DON'T DO THIS!
    Set<Plant>[] plantsByLifeCycle =
        (Set<Plant>[]) new Set[Plant.LifeCycle.values().length];
    
    for (int i = 0; i < plantsByLifeCycle.length; i++)
        plantsByLifeCycle[i] = new HashSet<>();
    
    for (Plant p : garden)
        plantsByLifeCycle[p.lifeCycle.ordinal()].add(p);
    
    // Print the results
    for (int i = 0; i < plantsByLifeCycle.length; i++) {
        System.out.printf("%s: %s%n",
            Plant.LifeCycle.values()[i], plantsByLifeCycle[i]);
    }
    
这种方法是有效的，但充满了问题。 
因为数组不兼容泛型（Item 28），程序需要一个未经检查的转换，并且不会干净地编译。 
由于该数组不知道索引代表什么，因此必须手动标记索引输出。 
但是这种技术最严重的问题是，当你访问一个由枚举序数索引的数组时，你有责任使用正确的int值; int不提供枚举的类型安全性。 
如果你使用了错误的值，程序会默默地做错误的事情，如果你幸运的话，抛出一个ArrayIndexOutOfBoundsException异常。

有一个更好的方法来达到同样的效果。 
该数组有效地用作从枚举到值的映射，因此不妨使用Map。 
更具体地说，有一个非常快速的Map实现，设计用于枚举键，称为java.util.EnumMap。 

下面是当程序重写为使用EnumMap时的样子：

    // Using an EnumMap to associate data with an enum
    Map<Plant.LifeCycle, Set<Plant>>  plantsByLifeCycle =
        new EnumMap<>(Plant.LifeCycle.class);
    
    for (Plant.LifeCycle lc : Plant.LifeCycle.values())
        plantsByLifeCycle.put(lc, new HashSet<>());
    
    for (Plant p : garden)
        plantsByLifeCycle.get(p.lifeCycle).add(p);

    System.out.println(plantsByLifeCycle);
    
这段程序更简短，更清晰，更安全，运行速度与原始版本相当。 
没有不安全的转换; 无需手动标记输出，因为map键是知道如何将自己转换为可打印字符串的枚举; 并且不可能在计算数组索引时出错。 
EnumMap与序数索引数组的速度相当，其原因是EnumMap内部使用了这样一个数组，但它对程序员的隐藏了这个实现细节，将Map的丰富性和类型安全性与数组的速度相结合。 
请注意，EnumMap构造方法接受键类型的Class对象：这是一个有限定的类型令牌（bounded type token），它提供运行时的泛型类型信息（Item 33）。

通过使用stream（Item 45）来管理Map，可以进一步缩短以前的程序。 
以下是最简单的基于stream的代码，它们在很大程度上重复了前面示例的行为：

    // Naive stream-based approach - unlikely to produce an EnumMap!
    System.out.println(Arrays.stream(garden)
        .collect(groupingBy(p -> p.lifeCycle)));

这个代码的问题在于它选择了自己的Map实现，实际上它不是EnumMap，所以它不会与显式EnumMap的版本的空间和时间性能相匹配。 
为了解决这个问题，使用Collectors.groupingBy的三个参数形式的方法，它允许调用者使用mapFactory参数指定map的实现：

    // Using a stream and an EnumMap to associate data with an enum
    System.out.println(Arrays.stream(garden)
            .collect(groupingBy(p -> p.lifeCycle,
        () -> new EnumMap<>(LifeCycle.class), toSet()))); 
        
这样的优化在像这样的示例程序中是不值得的，但是在大量使用Map的程序中可能是至关重要的。

基于stream版本的行为与EmumMap版本的行为略有不同。
EnumMap版本总是为每个工厂生命周期生成一个嵌套map类，而如果花园包含一个或多个具有该生命周期的植物时，则基于流的版本才会生成嵌套map类。 
因此，例如，如果花园包含一年生和多年生植物但没有两年生的植物，plantByLifeCycle的大小在EnumMap版本中为三个，在两个基于流的版本中为两个。

你可能会看到(两次)数组索引的数组，用序数来表示从两个枚举值的映射。

例如，这个程序使用这样一个数组来映射两个阶段到一个阶段转换（phase transition）（液体到固体表示凝固，液体到气体表示沸腾等等）：

    // Using ordinal() to index array of arrays - DON'T DO THIS!
    public enum Phase {
        SOLID, LIQUID, GAS;
    
        public enum Transition {
            MELT, FREEZE, BOIL, CONDENSE, SUBLIME, DEPOSIT;
            
            // Rows indexed by from-ordinal, cols by to-ordinal
            private static final Transition[][] TRANSITIONS = {
                { null,    MELT,     SUBLIME },
                { FREEZE,  null,     BOIL    },
                { DEPOSIT, CONDENSE, null    }
            };
    
            // Returns the phase transition from one phase to another
            public static Transition from(Phase from, Phase to) {
                return TRANSITIONS[from.ordinal()][to.ordinal()];
            }
        }
    }
    
这段程序可以运行，甚至可能显得优雅，但外观可能是骗人的。 
就像前面显示的简单的花园示例一样，编译器无法知道序数和数组索引之间的关系。 
如果在转换表中出错或者在修改Phase或Phase.Transition枚举类型时忘记更新它，则程序在运行时将失败。 
失败可能是ArrayIndexOutOfBoundsException，NullPointerException或（更糟糕的）沉默无提示的错误行为。 
即使非空Item的数量较小，表格的大小也是phase的个数的平方。

同样，可以用EnumMap做得更好。
因为每个阶段转换都由一对阶段枚举来索引，所以最好将关系表示为从一个枚举（from 阶段）到第二个枚举（to阶段）到结果（阶段转换）的map。 
与阶段转换相关的两个阶段最好通过将它们与阶段转换枚举相关联来捕获，然后可以用它来初始化嵌套的EnumMap：

    // Using a nested EnumMap to associate data with enum pairs
    public enum Phase {
       SOLID, LIQUID, GAS;
    
       public enum Transition {
          MELT(SOLID, LIQUID), FREEZE(LIQUID, SOLID),
          BOIL(LIQUID, GAS),   CONDENSE(GAS, LIQUID),
          SUBLIME(SOLID, GAS), DEPOSIT(GAS, SOLID);
    
          private final Phase from;
          private final Phase to;
    
          Transition(Phase from, Phase to) {
             this.from = from;
             this.to = to;
          }
    
          // Initialize the phase transition map
          private static final Map<Phase, Map<Phase, Transition>>
              m = Stream.of(values()).collect(groupingBy(t -> t.from,
                  () -> new EnumMap<>(Phase.class),
                  toMap(t -> t.to, t -> t,
                    (x, y) -> y, () -> new EnumMap<>(Phase.class))));
    
          public static Transition from(Phase from, Phase to) {
             return m.get(from).get(to);
          }
       }
    }
    
初始化阶段转换的map的代码有点复杂。
map的类型是Map<Phase, Map<Phase, Transition>>，意思是“从（源）阶段映射到从（目标）阶段到阶段转换映射。”
这个map的map使用两个收集器的级联序列进行初始化。 
第一个收集器按源阶段对转换进行分组，第二个收集器使用从目标阶段到转换的映射创建一个EnumMap。 
第二个收集器((x, y) -> y))中的合并方法未使用；仅仅因为我们需要指定一个map工厂才能获得一个EnumMap，并且Collectors提供伸缩式工厂，这是必需的。 
本书的前一版使用显式迭代来初始化阶段转换map。 
代码更详细，但可以更容易理解。

现在假设想为系统添加一个新阶段：等离子体或电离气体。 
这个阶段只有两个转变：电离，将气体转化为等离子体; 和去离子，将等离子体转化为气体。 
要更新基于数组的程序，必须将一个新的常量添加到Phase，将两个两次添加到Phase.Transition，并用新的十六个元素版本替换原始的九元素阵列数组。 
如果向数组中添加太多或太少的元素或者将元素乱序放置，那么如果运气不佳：程序将会编译，但在运行时会失败。 
要更新基于EnumMap的版本，只需将PLASMA添加到阶段列表中，并将IONIZE(GAS, PLASMA)和DEIONIZE(PLASMA, GAS)添加到阶段转换列表中：

    // Adding a new phase using the nested EnumMap implementation
    public enum Phase {
    
        SOLID, LIQUID, GAS, PLASMA;
    
        public enum Transition {
            MELT(SOLID, LIQUID), FREEZE(LIQUID, SOLID),
            BOIL(LIQUID, GAS),   CONDENSE(GAS, LIQUID),
            SUBLIME(SOLID, GAS), DEPOSIT(GAS, SOLID),
            IONIZE(GAS, PLASMA), DEIONIZE(PLASMA, GAS);
    
            ... // Remainder unchanged
        }
    }
    
该程序会处理所有其他事情，并且几乎不会出现错误。 
在内部，map的map是通过数组的数组实现的，因此在空间或时间上花费很少，以增加清晰度，安全性和易于维护。

为了简便起见，上面的示例使用null来表示状态更改的缺失(其从目标到源都是相同的)。
这不是很好的实践，很可能在运行时导致NullPointerException。
为这个问题设计一个干净、优雅的解决方案是非常棘手的，而且结果程序足够长，以至于它们会偏离这个Item的主要内容。

总之，使用序数来索引数组很不合适：改用EnumMap。 
如果你所代表的关系是多维的，请使用EnumMap <...，EnumMap <... >>。 
应用程序员应该很少使用Enum.ordinal（Item 35），如果使用了，也是一般原则的特例。

