---
layout: post
title: 《Effective Java》学习日志（八）67:谨慎地调优
date: 2018-12-28 18:45:04
author: admin
comments: true
categories: [Java]
tags: [Java,Effective Java]
---



<!-- more -->

------

学习资料主要参考： 《Effective Java Third Edition》，作者：Joshua Bloch

------




* 目录
{:toc}

------

有三个关于优化的格言，每个人都应该知道：

- 越来越多的计算罪恶被冠以效率之名（不一定达到它）而不是去深究其他原因 - 包括盲目的愚蠢。 —William A. Wulf [Wulf72]
- 我们应该忘记小的效率，大约97％的时间说：过早的优化是所有邪恶的根源。 —Donald E. Knuth [Knuth74]
- 我们在优化方面遵循两条规则：规则1.不要这样做；规则2（仅限专家），暂时不要这样做 - 除非你有一个完全清晰和未经优化的解决方案。 —M. A. Jackson [Jackson75]

所有这些格言都比Java编程语言早了二十年。 他们讲述了优化的深层真理：弊大于利，特别是如果你过早优化的话。 在此过程中，您可能会生成既不快又不正确且无法轻松修复的软件。

不要为了表现而牺牲合理的建筑原则。 **努力编写好的程序而不是快速的程序**。 如果一个好的程序不够快，它的架构将允许它进行优化。 好的程序体现了信息隐藏的原则：在可能的情况下，它们将设计决策本地化为单个组件，因此可以在不影响系统其余部分的情况下更改个别决策（第15项）。

这并不意味着在您的程序完成之前，您可以忽略性能问题。 实现的问题可以通过以后的优化来解决，但是如果不重写系统，就无法修复限制性能的普遍存在的架构缺陷。 事后改变设计的基本方面可能导致结构不良的系统难以维护和发展。 因此，您必须在设计过程中考虑性能。

**努力避免限制性能的设计决策**。 事后最难改变的设计组件是指定组件之间和外部世界之间的交互。 这些设计组件中最主要的是API，线级协议和持久数据格式。 事实上，这些设计组件不仅难以或不可能改变，而且所有这些都可能对系统可以实现的性能产生重大限制。

**考虑你的API设计决策所产生的性能影响**。 使公共类型可变可能需要大量不必要的防御性复制（第50项）。 类似地，在公共类中使用继承，其中组合适当地将类永远地绑定到其超类，这可以对子类的性能设置人为限制（第18项）。 作为最后一个示例，使用实现类型而不是API中的接口将您绑定到特定实现，即使将来可能会编写更快的实现（第64项）。

API设计对性能的影响是非常真实的。考虑java.awt.Component类中的getSize方法。这个性能关键方法返回Dimension实例的决定，加上Dimension实例可变的决定，强制此方法的任何实现都在每次调用时分配一个新的Dimension实例。尽管在现代VM上分配小对象的成本很低，但是不必要地分配数百万个对象会对性能造成实际损害。

存在几种API设计替代方案。理想情况下，Dimension应该是不可变的（第17项）;或者，getSize可能已被两个返回Dimension对象的各个基本组件的方法所取代。实际上，出于性能原因，在Java 2中将两个这样的方法添加到Component中。但是，预先存在的客户端代码仍然使用getSize方法，并且仍然会受到原始API设计决策的性能影响。

幸运的是，通常情况下，良好的API设计与良好的性能一致。**扭曲API以获得良好的性能是一个非常糟糕的主意**。导致您变形API的性能问题可能会在未来版本的平台或其他底层软件中消失，但扭曲的API及其附带的支持头痛将永远伴随着您。

一旦您仔细设计了程序并生成了清晰，简洁且结构良好的实现，那么可能是时候考虑优化，假设您对程序的性能不满意。

回想一下Jackson’s的两个优化规则是“不要这样做”和“（仅限专家）。暂时不要这样做。“他本可以再添加一个：**在每次尝试优化之前和之后测量性能**。 您可能会对所发现的内容感到惊讶。 通常，尝试优化对性能没有可测量的影响; 有时，他们会使情况变得更糟。 主要原因是很难猜出你的程序在哪里花费时间。 您认为程序部分很慢可能没有错，在这种情况下，您将浪费时间尝试优化它。 常识说，程序将90％的时间花在10％的代码上。

分析工具可以帮助您确定优化工作的重点。 这些工具为您提供运行时信息，例如每个方法消耗的大致时间以及调用的次数。 除了重点调整工作之外，这还可以提醒您需要进行算法更改。 如果一个二次（或更差）算法潜伏在你的程序中，那么没有多少调整可以解决问题。 您必须将算法替换为更有效的算法。 系统中的代码越多，使用分析器就越重要。 就像在大海捞针一样：大海捞针越大，金属探测器就越有用。 值得特别提及的另一个工具是jmh，它不是分析器，而是一个微基准测试框架，可以提供对Java代码详细性能的无与伦比的可视性[JMH]。

在Java中，测量优化尝试的效果的需求比在C和C ++等更传统的语言中更大，因为Java具有较弱的性能模型：各种原始操作的相对成本定义不太明确。 程序员编写的内容与CPU执行的内容之间的“抽象差距”更大，这使得更可靠地预测优化的性能结果变得更加困难。 有很多表演神话浮出水面，结果证明是半真半假或彻头彻尾的谎言。

Java的性能模型不仅定义不明确，而且从实现到实现，从发布到发布，从处理器到处理器都有所不同。 如果您将在多个实现或多个硬件平台上运行程序，那么衡量优化对每个实现的影响非常重要。 有时，您可能不得不在不同实现或硬件平台上的性能之间进行权衡。

自该项目首次编写以来近二十年，Java软件堆栈的每个组件都变得越来越复杂，从处理器到虚拟机再到类库，以及Java运行的各种硬件都在不断增长。 所有这些结合起来使得Java程序的性能现在比2001年更难以预测，并且相应地增加了测量它的需求。

总而言之，不要努力写出快速的程序 - 努力写出好的程序; 速度将随之而来。 但是在设计系统时要考虑性能，尤其是在设计API，线级协议和持久数据格式时。 完成系统构建后，测量其性能。 如果它足够快，你就完成了。 如果没有，请借助分析器找到问题的根源，然后开始优化系统的相关部分。 第一步是检查您选择的算法：没有多少低级优化可以弥补差的算法选择。 根据需要重复此过程，在每次更改后测量性能，直到您满意为止。















