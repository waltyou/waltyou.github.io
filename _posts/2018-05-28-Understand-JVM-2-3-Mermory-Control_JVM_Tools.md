---
layout: post
title: 《深入理解Java虚拟机：JVM高级特性与最佳实践--第二版》学习日志（二）Part 3：虚拟机性能监控与故障处理工具
date: 2018-05-28 19:06:04
author: admin
comments: true
categories: [Java]
tags: [Java, JVM]
---

JDK除了提供编译和运行Java的功能外，也附带了许多命令行工具来分析JVM运行状况。来简单了解一下。

<!-- more -->
---

学习资料主要参考： 《深入理解Java虚拟机：JVM高级特性与最佳实践(第二版)》，作者：周志明

---
# 目录
{:.no_toc}

* 目录
{:toc}

---

# 1. 概述

给一个系统定位问题时，知识、经验是关键基础，数据（运行日志、异常堆栈、GC日志、线程快照、堆转储快照）是依据，工具是运用知识处理数据的手段。

---

# 2. JDK的命令行工具

## 1）jps：虚拟机进程状况工具

**JVM Process Status Tool**

可以列出正在运行的虚拟机进程，并显示虚拟机执行主类（Main Class， main()函数所在的类）名称，以及这些进程的本地虚拟机唯一ID（Local Virtual Machine Identifier， LVMID）。

**命令格式**

```
jps [ options ] [ hostid]
```

**主要选项**

选项 | 作用
---|---
-q | 只输出LVMID
-m | 输出虚拟机启动时，传递给主类main函数的参数
-l | 输出主类的全名，如果进程执行的是Jar包，输出Jar路径
-v | 输出虚拟机进程启动时JVM参数

可以到[官网](https://docs.oracle.com/javase/7/docs/technotes/tools/share/jps.html)上了解更多。

## 2）jstat：虚拟机统计信息监视工具

**JVM Statistics Monitoring Tool**

用于监视虚拟机各种运行状态信息的命令行工具。它可以显示本地或者远程虚拟机进程中的类装载、内存、垃圾收集、JIT编译等运行数据，在没有GUI的服务器上，它是运行期定位虚拟机性能问题的首选工具。

**命令格式**

```
jstat [ options vmid [interval[s|ms] [count]]
```

如果是远程虚拟机进程，VMID的格式应为：[protocol:][//]lvmid[@hostname[:port]/servername]

参数interval和count代表查询间隔和次数，如果省略这两个参数，说明只查询一次。比如要每250ms查询一次id为2764的进程的垃圾收集情况，一共查询20次，命令应该是：jstat -gc 2764 250 20

**主要选项**

主要分为三类：类装载、垃圾收集、运行期编译状况。

选项 | 作用
---|---
-class | 监视类装载、卸载数量、总空间以及类装载所耗费的时间
-gc | 监视Java堆的状况，包括Eden区、两个survivor区、老年代、永久代等的容量，已用空间、GC时间合计等信息。
-gccapacity | 监视内容与-gc基本相同，但输出主要关注Java堆各个区域使用到的最大、最小空间
-gcutil | 监视内容与-gc基本相同，但输出主要关注已使用的空间占总空间的百分比
-gccause | 与-gcutil功能一样，但是会额外输出导致上一次GC产生的原因
-gcnew | 监视新生代GC情况
-gcnewcapacity | 监视内容与-gcnew基本相同，输出主要关注使用到的最大、最小空间
-gcold | 监视老年代GC情况
-gcoldcapacity | 监视内容与-gcold基本相同，输出主要关注使用到的最大、最小空间
-gcpermcapacity | 输出永久代使用到的最大、最小空间
-complier | 输出JIT编译器编译过的方法、耗时等信息
-printcompliation | 输出已经被JIT编译的方法


可以到[官网](https://docs.oracle.com/javase/7/docs/technotes/tools/share/jstat.html)上了解更多。

## 3）jinfo：Java配置信息工具

**Configuration Info for Java**

实时地查看和调整虚拟机各项参数。

jinfo的-flag选项，可以获得未被显式指定的参数的系统默认值。

--sysprops可以打印虚拟机进程的System.getPriorities()内容。

使用-flag[+\|-] name 或者 -flag name=value修改一部分运行期可写的虚拟机参数。

**命令格式**

```
jinfo [ options ] pid
```

可以到[官网](https://docs.oracle.com/javase/7/docs/technotes/tools/share/jinfo.html)上了解更多。

## 4）jmap：Java内存映像工具

**Memory Map for Java**

用于生成堆转储快照（heapdump或dump文件）。它还可以查询finalize执行队列、Java堆和永久代的详细信息，如空间使用率、当前使用的收集器类型。

**命令格式**

```
jmap [ options ] vmid
```

**主要选项**

选项 | 作用
---|---
-dump | 生成Java堆转储快照。格式为：-dump:[live, ]format=b, file=<filename>，其中live子参数说明是否只dump出存活的对象
-finalizerinfo | 显示在F-queue中等待Finalizer线程执行finalize方法的对象
-heap | 显示Java堆详细信息。如使用哪种回收器、参数配置、分代状况等
-histo | 显示堆中对象统计信息，包括类、实例数量、合计容量
-permstat | 以ClassLoader为统计口径显示永久代内存状态
-F | 当虚拟机进程对dump选项没有响应时，可以使用这个强制生成dump快照

可以到[官网](https://docs.oracle.com/javase/7/docs/technotes/tools/share/jmap.html)上了解更多。

## 5）jhat：虚拟机堆转储快照分析工具

**JVM Heap Analysis Tool**

与jmap搭配使用，来分析jmap生成的堆转储快照。它内置了一个微型的HTTP/HTML服务器，生成dump文件的分析结果后，可以在浏览器中查看。

不过很少用它。因为：
- 分析是个耗时、消耗硬件资源的过程，没必要在服务所在机器上分析
- 分析功能相对简陋

## 6）jstack：Java堆栈跟踪工具

**Stack Trace for Java**

用于生成虚拟机当前时刻的线程快照（treaddump或者javacore文件）。

线程快照就是当前虚拟机内每一条线程正在执行的方法堆栈的集合。这个文件的目的主要是定位线程出现长时间停顿的原因，如线程间死锁、死循环、请求外部资源导致的长时间等待等。线程出现停顿的时候，通过jstack来查看各个线程的调用堆栈，就可以知道没有响应的线程到底在后台做些什么事情。

**命令格式**

```
jstack [ options ] vmid
```

**主要选项**

选项 | 作用
---|---
-F | 当正常输出的请求不被响应时，强制输出线程堆栈
-l | 出堆栈外，显示关于锁的附加信息
-m | 如果调用本地方法的话，可以显示c、c++的堆栈

可以到[官网](https://docs.oracle.com/javase/7/docs/technotes/tools/share/jstack.html)上了解更多。

## 7）HSDIS：JIT生成代码反汇编

让Hotspot的-XX:+PrintAssembly指令调用它来吧动态生成的本地代码还原为汇编代码输出，并且还生成了大量非常有价值的注释。

---

# 3. JDK的可视化工具

1. JConsole：Java监视与管理控制台
2. VisualVM：多合一故障处理工具
