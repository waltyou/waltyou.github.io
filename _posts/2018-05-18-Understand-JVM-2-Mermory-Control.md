---
layout: post
title: 《深入理解Java虚拟机：JVM高级特性与最佳实践--第二版》学习日志（二）：自动内存管理机制
date: 2018-5-17 21:06:04
author: admin
comments: true
categories: [Java]
tags: [Java，JVM]
---

学习JVM，首先要了解JVM是如何划分内存，然后引出垃圾回收算法，最后介绍了常用的JVM调试工具和JVM调优的几个实例。

<!-- more -->
---

学习资料主要参考： 《深入理解Java虚拟机：JVM高级特性与最佳实践(第二版)》，作者：周志明

---
## 目录
{:.no_toc}

* 目录
{:toc}

---

# Java内存区域与内存溢出异常

## 1. 概述

C、C++程序员，既拥有对每一个对象的“所有权”，又担负它们生命开始到终结的维护责任。

Java程序员，把内存控制的权利交给了JVM。

## 2. 运行时数据区域

如下图：

[![](/images/posts/JVM_Area.png)](/images/posts/JVM_Area.png)

分别介绍各个区域。

### 1）程序计数器Program Counter Register

**定义与功能：**

**当前线程**所执行的**字节码**的**行号**指示器。

字节码解释器工作时就是通过改变这个计数器的值来选取下一条需要执行的字节码指令，分支、循环、跳转、异常处理、线程恢复都需要依赖这个计数器。

**特点：**
1. 每条线程都需要一个独立的程序计数器，各个线程间互不影响。
2. 它是线程私有的内存。

### 2）Java虚拟机栈

**定义与功能：**

它描述的是**Java方法执行**的内存模型：每个方法在执行的同时都会创建一个栈帧（Stack Frame），用于存储局部变量表、操作数栈、动态链接、方法出口等信息。

每个方法从**调用**到**执行完成**的过程，就对应一个栈帧在虚拟机中**入栈**到**出栈**的过程。

**特点：**
1. 线程私有
2. 生命周期和线程相同。
2. 是局部变量存放的地方，所以又叫它：局部变量表。其中存放了各种基本数据类型、对象引用、returnAddress类型（指向一条字节码的地址）。
3. 局部变量中，64位的long和double类型会占用2个局部变量空间（Slot），其余都占用1个。**局部变量表所需内存在编译期间完成分配。**

**异常：**

1. StackOverflowError: 当线程请求的栈深度大于虚拟机所允许的深度
2. OutOfMemoryError：如果虚拟机栈允许动态扩展，而且扩展到无法申请足够的内存时。


### 3）本地方法栈

**定义与功能：**

与虚拟机栈所发挥的作用非常相似，只不过虚拟机栈为虚拟机执行Java方法（字节码）服务，而本地方法栈为虚拟机使用Native方法服务。

**特点：**

虚拟机规范中，对本地方法栈中方法使用的语言、方式、数据结构没有强制规定，因此虚拟机可以自由实现它。HotSpot虚拟机直接合并了本地方法栈和虚拟机栈。

**异常：**

也会抛出StackOverflowError 和OutOfMemoryError异常。

### 4）Java堆

**定义与功能：**

唯一目的就是存放对象实例，几乎所有的对象实例都在这里分配内存。
Java堆是垃圾收集器管理的主要区域，因此也称为“GC堆”。

内部也可以细分，如下图：


**特点：**

1. 是虚拟机中内存最大的一块
2. 它被所有线程共享
3. 可以物理上不连续，只需逻辑上连续

**异常：**

当堆无法再拓展时，抛出OutOfMemoryError异常。

### 5）方法区

**定义与功能：**

它用于存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据。虽然在虚拟机规范中，它被描述为堆的一个逻辑部分，但是它有一个别名叫做“Non-Heap”，目的是与Heap分开。

**特点：**

1. 它被所有线程共享
2. 在HotSpot上，方法区被称为永久代，因为HotSpot把GC非带收集扩展至方法区。
3. 可以物理上不连续，只需逻辑上连续
4. 可以选择不实现垃圾收集。这个区域主要是针对常量池的回收和对类型的卸载，然而这个区域的回收效果不好，但是确实是必要的，因为此区域未完全回收会导致内存泄漏。

**异常：**

当方法区无法满足内存分配的需求时，抛出OutOfMemoryError异常。

### 6）运行时常量池

**定义与功能：**

是方法区的一部分。Class文件中除了类的版本、字段、方法、接口等描述性信息外，还有一项是常量池，用来存放**编译期**产生的各种字面量和符号引用，这部分内容将在类加载后进入方法区的运行时常量池。

**特点：**

1. 相对于Class文件常量池，运行时常量池具有动态性。不一定都要编译期产生，可以在运行期间进入，比较常见的是String类的intern()方法。

**异常：**

当常量池无法申请到内存时，抛出OutOfMemoryError异常。

### 7）直接内存

**定义与功能：**

JDK1.4中的NIO类，引入了一种基于通道（Channel）和缓冲区（Buffer）的I/O方法，它可以使用Native函数库直接分配**堆外内存**，然后通过一个存储**在Java堆**中的DirectByteBuffer对象作为堆外内存的**引用**进行操作，在一些场景下，避免了在Native堆和Java堆之间来回复制数据。

**特点：**

并不是虚拟机运行时数据区的一部分，也不是Java虚拟机规范中定义的内存区域。

**异常：**

人们往往会忽略虚拟机之外的这个直接内存，所以当各个内存区域总和大于物理内存限制，就会导致动态扩展时，抛出OutOfMemoryError异常。


## 3. HotSpot虚拟机对象探秘

### 1）对象的创建

1. 遇到**new**指令
2. 检查new指令参数是否能在**常量池**中定位到一个**类的符号引用**，并且检查这个符号引用代表的类是否被加载、解析和初始化过；如果没有，执行相应的**类加载**过程。
3. 为新生对象分配内存。

    通常有两种分配方式：指针碰撞和空闲列表。

    **指针碰撞**：在堆内存规整的情况下，即使用Serial、ParNew等带压缩整理（Compact）过程的收集器时，一个指针记录着空闲内存和使用内存的分界点，通过移动指针给新生对象分配内存。

    **空闲列表**：在堆内存不规整的情况下，即使用CMS这种基于Mark-Sweep算法的收集器时，虚拟机需要维护一个列表，记录那些内存卡是可用的，在分配时从列表中找到一块足够大的空间划分给对象，并且更新列表。

    分配内存的行为，**在并发情况下可能不是线程安全的**。通常有两个方法解决。

    一种是对分配内存空间的动作进行**同步处理**（JVM使用[CAS](https://blog.csdn.net/liubenlong007/article/details/53761730)保证更新操作的原子性）。

    另外一种方法是把内存分配的动作按照线程划分在不同的空间之中进行，即每个线程在Java堆中预先分配一小块内存，称为**本地线程分配缓冲**（Thread Local Allocation Buffer，TLAB）。哪个线程要分配内存，就在哪个线程的TLAB上分配，只有TLAB用完并分配新的TLAB时，才需要同步锁定。可以通过参数-XX:+/-UseTLAB参数来设定是否使用TLAB。

4. 将分配到的内存空间都初始化为零值（不包含对象头）。如果使用了TLAB，可以提前在分配TLAB时进行这一操作。这步操作保证了对象的实例字段，在Java代码中可以不赋值就能直接使用它们对于数据类型的零值。
5. 对对象进行必要的设置。例如对象是哪个类的实例、如何才能找到类的元数据信息、对象的哈希码、对象的GC分代年龄等信息。这些信息都存在对象头中。
6. 执行<init>方法。把对象按照程序员的意愿进行初始化。

### 2）对象的内存布局

1. 对象头 Header

    第一部分存储对象自身的运行时数据，如Hashcode、GC分代年龄、锁状态标志、线程持有的锁、偏向线程ID、偏向时间戳等。
    第二部分是类型指针，即对象指向他的类元数据的指针，虚拟机通过这个指针来确定对象是哪个类的实例。如果对象是一个Java数字，那在对象头中还有一块记录数组长度的数据，因为虚拟机无法从数组的元数据中确定数组的大小。
2. 实例数据 Instance Data

    对象真正存储的有效信息，也是在代码中定义的各种类型的字段内容。无论是父类继承下来的，还是子类定义的，都要记录。存储顺序受虚拟机分配策略参数和字段在java源码中定义顺序的影响。父类变量优先在子类之前。
3. 对齐填充 Padding

    不是必然存在的，也没有特别的含义，只是占位符的作用。

### 3）对象的访问定位

Java程序需要通过栈上的reference数据来操作堆上的具体对象。

主流的访问方式有下面两种。
1. 句柄方式

    Java堆中划分出一块作为句柄池。reference中存储的就是对象的句柄地址，句柄中包含了对象实例数据与类型数据各自具体的地址信息。

    **优点**：对象移动后，无需修改reference。
2. 直接指针

    reference中存储的直接是对象地址，这时候在堆对象布局中就必须要考虑如何放置访问类型数据的相关信息。

    **优点**：速度快，节省了一次指针定位的时间开销。

## 4. 实战：OutOfMemoryError异常

### 1）Java堆溢出

```java
/**
 * VM Args：-Xms20m -Xmx20m -XX:+HeapDumpOnOutOfMemoryError
 * @author zzm
 */
public class HeapOOM {

	static class OOMObject {
	}

	public static void main(String[] args) {
		List<OOMObject> list = new ArrayList<OOMObject>();

		while (true) {
			list.add(new OOMObject());
		}
	}
}
```

### 2）虚拟机栈和本地方法栈溢出

栈容量用**-Xss**参数设定。

Java虚拟机规范中描述了两种异常：
- 如果线程请求的栈深度大于虚拟机所允许的最大深度，将抛出StackOverflowError异常
- 如果虚拟机在扩展栈时无法申请到足够的内存空间，则抛出OutOfMemoryError异常

上述两种异常，存在重叠的地方，当栈空间无法继续分配时，到底是是内存太小，还是已使用的栈空间太大，本质上是对同一件事的两种描述。

1. 使用-Xss参数缩小栈内存容量，抛出StackOverflowError异常
2. 定义大量的本地变量，增大方法帧中本地变量表的长度，抛出StackOverflowError异常

```java
/**
 * VM Args：-Xss128k
 * @author zzm
 */
public class JavaVMStackSOF {

	private int stackLength = 1;

	public void stackLeak() {
		stackLength++;
		stackLeak();
	}

	public static void main(String[] args) throws Throwable {
		JavaVMStackSOF oom = new JavaVMStackSOF();
		try {
			oom.stackLeak();
		} catch (Throwable e) {
			System.out.println("stack length:" + oom.stackLength);
			throw e;
		}
	}
}
```

如果通过不断建立线程的方式，倒是可以产生内存溢出异常，但是这和栈空间的大小无关，代码如下：
```java
/**
 * VM Args：-Xss2M （这时候不妨设大些）
 * @author zzm
 */
public class JavaVMStackOOM {

       private void dontStop() {
              while (true) {
              }
       }

       public void stackLeakByThread() {
              while (true) {
                     Thread thread = new Thread(new Runnable() {
                            @Override
                            public void run() {
                                   dontStop();
                            }
                     });
                     thread.start();
              }
       }

       public static void main(String[] args) throws Throwable {
              JavaVMStackOOM oom = new JavaVMStackOOM();
              oom.stackLeakByThread();
       }
}
```

因为虚拟机可以通过参数来控制堆和方法区的内存最大值，物理内存去除Xmx和MaxPermSize后，剩下的内存由虚拟机栈和本地方法栈瓜分。这时候如果每个线程分配到的栈容量越大，可以建立的线程就越少。


### 3）方法区和运行时常量池溢出

可以通过-XX:PermSize和-XX:MaxPermSize设定方法区的大小。

String.intern()方法是一个Native方法，它的作用是：如果常量池中包含了等于这个String对象的字符串，就返回池中的字符串对象，否则将此对象加入常量池，并返回此对象的引用。

```java
/**
 * VM Args：-XX:PermSize=10M -XX:MaxPermSize=10M
 * @author zzm
 */
public class RuntimeConstantPoolOOM {

	public static void main(String[] args) {
		// 使用List保持着常量池引用，避免Full GC回收常量池行为
		List<String> list = new ArrayList<String>();
		// 10MB的PermSize在integer范围内足够产生OOM了
		int i = 0;
		while (true) {
			list.add(String.valueOf(i++).intern());
		}
	}
}
```

上面的代码在JDK1.6及之前的版本中，会产生异常，1.7之后就不会了。这是为啥呢？来看下面代码：

```
public class RuntimeConstantPoolOOM {

	public static void main(String[] args) {
		public static void main(String[] args) {
		String str1 = new StringBuilder("计算机").append("软件").toString();
		System.out.println(str1.intern() == str1);

		String str2 = new StringBuilder("ja").append("va").toString();
		System.out.println(str2.intern() == str2);
	}	}
}
```

在1.6之前会打印两个false，1.7则会得到一个true和一个false。
1.6中，intern方法会吧首次遇到的字符串实例复制到永久代中，返回的也是永久代中这个实例的引用，而stringBuffer创建的string实例在堆中，所以必然不是同一个引用。而1.7的intern不会复制实例，只是在常量池中记录**首次出现**的实例引用，所以由于“计算机软件”是首次出现，所以返回true，“java”这个不是首次出现，所以返回false。

除了上述利用string.intern产生方法区的异常外，也可以产生大量的动态类去填满方法区，如下（借助了CGLib）：

```java
/**
 * VM Args： -XX:PermSize=10M -XX:MaxPermSize=10M
 * @author zzm
 */
public class JavaMethodAreaOOM {

	public static void main(String[] args) {
		while (true) {
			Enhancer enhancer = new Enhancer();
			enhancer.setSuperclass(OOMObject.class);
			enhancer.setUseCache(false);
			enhancer.setCallback(new MethodInterceptor() {
				public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
					return proxy.invokeSuper(obj, args);
				}
			});
			enhancer.create();
		}
	}

	static class OOMObject {

	}
}
```

### 4）本机直接内存溢出

可以使用**-XX:MaxDirectMemorySize**指定，如果不指定，则默认与Java堆最大值一样。

```java
/**
 * VM Args：-Xmx20M -XX:MaxDirectMemorySize=10M
 * @author zzm
 */
public class DirectMemoryOOM {

	private static final int _1MB = 1024 * 1024;

	public static void main(String[] args) throws Exception {
		Field unsafeField = Unsafe.class.getDeclaredFields()[0];
		unsafeField.setAccessible(true);
		Unsafe unsafe = (Unsafe) unsafeField.get(null);
		while (true) {
			unsafe.allocateMemory(_1MB);
		}
	}
}
```

有直接内存导致的内存溢出，不会在Heap Dump文件中看见明显的异常，如果发现OOM后dump的文件很小，而程序又使用了NIO，那就可以考虑检查一下是不是这方面的原因。

---

# 垃圾收集器与内存分配策略

## 1. 概述

关于垃圾回收的3个问题:
1. 那些内存需要回收？
2. 什么时候回收？
3. 如何回收？

关于第一个问题，在上一章中，我们已经知道了程序计数器、虚拟机栈、本地方法栈，这3个地方，和线程的生命周期是一样的，而且栈帧的大小，在类结构确定下来后就已经基本可知了。因此这3部分的内存具有确定性，而堆却不一样。一个接口的实现类的内存可能会不一样，只有在运行时才知道创建了哪些对象，它们的创建和回收都是动态的。

所以需要关注的是**堆中内存**的回收。

## 2. 对象已死吗？

### 1）引用计数法

**定义**：给对象添加一个引用计数器，被引用时，计数器加一，引用失效时，计数器减一，当数值为0时，该对象就是不可用的。

**优缺点**：虽然这个方法简单高效，但是它无法解决对象之间循环引用的问题。

### 2）可达性分析算法

[![](/images/posts/GC-Roots.png)](/images/posts/GC-Roots.png)

**定义**：通过一系列的“GC Roots”作为起点，然后向下搜索，搜索走的地方成为“引用链”，当一个对象到GC roots没有任何引用链时，这个对象就是不可用的。

Java中可以作为GC Roots的对象有以下几种：
- 虚拟机栈中（栈帧中本地变量表）引用的对象
- 方法区中类静态属性引用的对象
- 方法区中常量引用的对象
- 本地方法栈中引用的对象

### 3）再谈引用

以上两种方法，都和“引用”有关。

在实际情况中，我们希望描述这样子一种情况：当内存足够时，可以保留，不足时就回收。这个对应了缓存功能。所以引出了下面四类引用：

1. 强引用 strong reference

    在代码中普遍存在，类似“Object a = new Object()”，只有强引用在，永不回收。
2. 软引用 soft reference

    描述一些还有用但并非必需的对象。对于弱引用的对象，会在即将发生内存溢出之前，会将它们列入回收范围，再次进行垃圾回收。SoftReference类来实现软连接。
3. 弱引用 week reference

    也是用来描述非必须对象的，但是它比软引用更弱一些，它只能生存到下雨一次垃圾收集发生之前。WeakReference类来实现弱引用。
4. 虚引用 phantom reference

    最弱的一种引用关系。一个对象是否有虚引用的存在，对其生存时间毫无影响，也无法通过虚引用来取得一个对象实例。设置它的唯一目的是：能在这个对象被回收时，获得一个系统通知。PhantomReference类来实现虚引用。

### 4）生存还是死亡

要宣告一个对象的死亡，至少要经历两次标记过程。

如果在进行可达性分析时，此对象没有和GC Roots相连接的引用链，那么它会被第一次标记并进行一次筛选。筛选的标准是对象是否有必要执行finalize()方法。当对象没有覆盖finalize方法，或者它已经被虚拟机调用过，虚拟机将这两种情况都视为“没有必要执行”。

如果这个对象被判断为有必要执行finalize方法，那么这个对象会被放在一个F-Queue的队列之中，之后会有一个虚拟机自动建立的、低优先级的Finalizer线程去执行，但不保证等待方法运行结束，以避免队列永久堵塞。

稍后GC会对F-Queue中的对象进行第二次小规模的标记。如果在finalize方法中，对象重新与引用链上任何一个对象建立了关联，那么它就会被移除出“即将回收”的集合，如果它没有，就会被真正回收。

注意，finalize方法只会被系统自动调用一次。

### 5）回收方法区

主要分为两部分：废弃常量和无用的类。

判断常量是否废弃比较简单，只需判断是否在其他地方有这个常量的引用即可。

判断类是否无用，相对苛刻，需要同时满足下面3个条件，才算是无用的类：
1. 该类所有的实例都已经被回收。
2. 加载该类的ClassLoader已经被回收。
3. 该类对于的java.lang.Class对象没有在任何地方被引用，无法再任何地方通过反射访问该类的方法。

## 3. 垃圾回收算法

### 1）标记-清除算法

[![](/images/posts/Mark-Sweep.jpg)](/images/posts/Mark-Sweep.jpg)

Mark-Sweep: 标记需要回收的对象，标记完成后，统一回收。

不足：
1. 效率问题：标记和回收效率都不高
2. 空间问题：该算法执行过后，会产生大量不连续的内存碎片，会导致之后需要分配大对象时，无法找到足够的连续内存而不得不提前触发另一次GC动作。

### 2）复制算法

[![](/images/posts/Copy-GC.jpg)](/images/posts/Copy-GC.jpg)

为了解决效率问题，将内存分为两块，每次只使用其中一块，当这一块内存使用完了之后，就将还存活的对象复制到另一块内存上，然后再把这一块内存全部清理掉。

这昂子就不用考虑内存碎片等问题，实现简单，运行高效，只是这种算法的代价是将可使用的内存变小了。

HotSpot虚拟机将新生代分为Eden和两个Survivor空间，它们的大小比例是8：1：1。每次回收时，就将Eden和其中一个Survivor中存活的对象，全部copy到另一块Survivor中。当Survivor内存不够用时，对象会通过分配担保机制，进入老年代。

### 3）标记-整理算法

[![](/images/posts/Mark-Compact.jpg)](/images/posts/Mark-Compact.jpg)

复制算法在遇到那些对象存活率较高的情况时，复制操作就会变多，效率也会降低。所以老年代不适合复制算法。

根据老年代特点，提出“标记-整理”（Mark-Compact）算法。标记过程，和标记-清理算法一致，只是后续步骤不是直接对可回收对象进行清理，而是让所有存活对象，都向一端移动，最后直接清理掉端边界外的内存。

### 4）分代收集算法

根据对象存活周期的不同，将内存分为几块，一般是把堆分为新生代和老年代。

新生代，对象存活率低，可采用复制算法。
老年代对象存活率高，必须采用“标记-清理”或者“标记-整理”算法来回收。

## 4. HotSpot的算法实现

### 1）枚举根节点

在可达性分析中，从GC Roots节点找引用链这个操作，可作为GC Roots的节点在全局性的引用和执行上下文中，而且方法区很大，如果逐个检查引用，会消耗很多时间。

另外，可达性分析对执行时间的敏感还体现在GC停顿上，因为在标记过程中，不希望引用关系还在变动。这也是“Stop The World”的其中一个重要原因。

在HotSpot的实现中，使用一组称为OopMap的数据结构，来直接得知哪些地方存放着对象引用，而不需要扫描所有引用。

### 2）安全点

Hotspot没有为每条指令都生成OopMap，只是在安全点（Safepoint）记录这些信息。

安全点通常选用在方法调用、循环跳转、异常跳转等，因为具有让程序长时间运行的特征。

另外需要考虑的是，多线程情况下，如果让所有线程读到最近的安全点上停下来。这里有两种方案：
- 抢先式中断：GC时，直接中断所有线程，如果发现某个线程不在安全点是，就运行它，让它运行到安全点。这种方式现在没有虚拟机使用。
- 主动式中断：当GC要中断线程时，去设置一个标志，各个线程在执行时，主动轮询这个标志，当标志为真时就自己中断挂起。

### 3）安全区域

当线程处理sleep状态或者blocked状态，这时候线程无法响应JVM的中断请求。对于这种情况，需要引入安全区域（Safe Region）来解决。

安全区域指在一段代码片段中，引用关系不会发生变化，在这个区域的任意地方开始GC都是安全的。

## 5. 垃圾收集器

[![](/images/posts/7-GC.jpg)](/images/posts/7-GC.jpg)

以上是 HotSpot 虚拟机中的7个垃圾收集器，连线表示垃圾收集器可以配合使用。


### 1）Serial收集器

[![](/images/posts/Serial.jpg)](/images/posts/Serial.jpg)

垃圾收集和用户程序交替执行，这意味着在执行垃圾收集的时候需要停顿用户程序。以串行的方式执行。

它是单线程的收集器，只会使用一个线程进行垃圾收集工作。

它的优点是简单高效，对于单个 CPU 环境来说，由于没有线程交互的开销，因此拥有最高的单线程收集效率。

它是 Client 模式下的默认新生代收集器，因为在用户的桌面应用场景下，分配给虚拟机管理的内存一般来说不会很大。Serial 收集器收集几十兆甚至一两百兆的新生代停顿时间可以控制在一百多毫秒以内，只要不是太频繁，这点停顿是可以接受的。

### 2）ParNew收集器

[![](/images/posts/ParNew.jpg)](/images/posts/ParNew.jpg)

它是 Serial 收集器的多线程版本。

是 Server 模式下的虚拟机首选新生代收集器，除了性能原因外，主要是因为除了 Serial 收集器，只有它能与 CMS 收集器配合工作。

默认开始的线程数量与 CPU 数量相同，可以使用 -XX:ParallelGCThreads 参数来设置线程数。

### 3）Parallel Scavenger收集器

与 ParNew 一样是并行的多线程收集器。

其它收集器关注点是尽可能缩短垃圾收集时用户线程的停顿时间，而它的目标是达到一个可控制的吞吐量，它被称为“吞吐量优先”收集器。这里的吞吐量指 CPU 用于运行用户代码的时间占总时间的比值。

停顿时间越短就越适合需要与用户交互的程序，良好的响应速度能提升用户体验。而高吞吐量则可以高效率地利用 CPU 时间，尽快完成程序的运算任务，主要适合在后台运算而不需要太多交互的任务。

提供了两个参数用于精确控制吞吐量，分别是控制最大垃圾收集停顿时间 -XX:MaxGCPauseMillis 参数以及直接设置吞吐量大小的 -XX:GCTimeRatio 参数（值为大于 0 且小于 100 的整数）。缩短停顿时间是以牺牲吞吐量和新生代空间来换取的：新生代空间变小，垃圾回收变得频繁，导致吞吐量下降。

还提供了一个参数 -XX:+UseAdaptiveSizePolicy，这是一个开关参数，打开参数后，就不需要手工指定新生代的大小（-Xmn）、Eden 和 Survivor 区的比例（-XX:SurvivorRatio）、晋升老年代对象年龄（-XX:PretenureSizeThreshold）等细节参数了，虚拟机会根据当前系统的运行情况收集性能监控信息，动态调整这些参数以提供最合适的停顿时间或者最大的吞吐量，这种方式称为 GC 自适应的调节策略（GC Ergonomics）。

### 4）Serial Old收集器

[![](/images/posts/SerialOld.jpg)](/images/posts/SerialOld.jpg)

是 Serial 收集器的老年代版本，也是给 Client 模式下的虚拟机使用。如果用在 Server 模式下，它有两大用途：

- 在 JDK 1.5 以及之前版本（Parallel Old 诞生以前）中与 Parallel Scavenge 收集器搭配使用。
- 作为 CMS 收集器的后备预案，在并发收集发生 Concurrent Mode Failure 时使用。

### 5）Parallel Old收集器

[![](/images/posts/ParOld.jpg)](/images/posts/ParOld.jpg)

是 Parallel Scavenge 收集器的老年代版本。

在注重吞吐量以及 CPU 资源敏感的场合，都可以优先考虑 Parallel Scavenge 加 Parallel Old 收集器。

### 6）CMS收集器

[![](/images/posts/CMS.jpg)](/images/posts/CMS.jpg)

CMS（Concurrent Mark Sweep），Mark Sweep 指的是标记 - 清除算法。

特点：并发收集、低停顿。并发指的是用户线程和 GC 线程同时运行。

分为以下四个流程：

1. 初始标记：仅仅只是标记一下 GC Roots 能直接关联到的对象，速度很快，需要停顿。
2. 并发标记：进行 GC Roots Tracing 的过程，它在整个回收过程中耗时最长，不需要停顿。
3. 重新标记：为了修正并发标记期间因用户程序继续运作而导致标记产生变动的那一部分对象的标记记录，需要停顿。
4. 并发清除：不需要停顿。

在整个过程中耗时最长的并发标记和并发清除过程中，收集器线程都可以与用户线程一起工作，不需要进行停顿。

具有以下缺点：

- 吞吐量低：低停顿时间是以牺牲吞吐量为代价的，导致 CPU 利用率不够高。
- 无法处理浮动垃圾，可能出现 Concurrent Mode Failure。浮动垃圾是指并发清除阶段由于用户线程继续运行而产生的垃圾，这部分垃圾只能到下一次 GC 时才能进行回收。由于浮动垃圾的存在，因此需要预留出一部分内存，意味着 CMS 收集不能像其它收集器那样等待老年代快满的时候再回收。可以使用 -XX:CMSInitiatingOccupancyFraction 来改变触发 CMS 收集器工作的内存占用百分，如果这个值设置的太大，导致预留的内存不够存放浮动垃圾，就会出现 Concurrent Mode Failure，这时虚拟机将临时启用 Serial Old 来替代 CMS。
- 标记 - 清除算法导致的空间碎片，往往出现老年代空间剩余，但无法找到足够大连续空间来分配当前对象，不得不提前触发一次 Full GC。


### 7）G1收集器

G1（Garbage-First），它是一款面向服务端应用的垃圾收集器，在多 CPU 和大内存的场景下有很好的性能。HotSpot 开发团队赋予它的使命是未来可以替换掉 CMS 收集器。

Java 堆被分为新生代、老年代和永久代，其它收集器进行收集的范围都是整个新生代或者老生代，而 G1 可以直接对新生代和永久代一起回收。

G1 把新生代和老年代划分成多个大小相等的独立区域（Region），新生代和永久代不再物理隔离。

[![](/images/posts/G1-Region.png)](/images/posts/G1-Region.png)

通过引入 Region 的概念，从而将原来的一整块内存空间划分成多个的小空间，使得每个小空间可以单独进行垃圾回收。这种划分方法带来了很大的灵活性，使得可预测的停顿时间模型成为可能。通过记录每个 Region 记录垃圾回收时间以及回收所获得的空间（这两个值是通过过去回收的经验获得），并维护一个优先列表，每次根据允许的收集时间，优先回收价值最大的 Region。

每个 Region 都有一个 Remembered Set，用来记录该 Region 对象的引用对象所在的 Region。通过使用 Remembered Set，在做可达性分析的时候就可以避免全堆扫描。

[![](/images/posts/G1.jpg)](/images/posts/G1.png)

如果不计算维护 Remembered Set 的操作，G1 收集器的运作大致可划分为以下几个步骤：
1. 初始标记
2. 并发标记
3. 最终标记：为了修正在并发标记期间因用户程序继续运作而导致标记产生变动的那一部分标记记录，虚拟机将这段时间对象变化记录在线程的 Remembered Set Logs 里面，最终标记阶段需要把 Remembered Set Logs 的数据合并到 Remembered Set 中。这阶段需要停顿线程，但是可并行执行。
4. 筛选回收：首先对各个 Region 中的回收价值和成本进行排序，根据用户所期望的 GC 停顿是时间来制定回收计划。此阶段其实也可以做到与用户程序一起并发执行，但是因为只回收一部分 Region，时间是用户可控制的，而且停顿用户线程将大幅度提高收集效率。

具备如下特点：
- 空间整合：整体来看是基于“标记 - 整理”算法实现的收集器，从局部（两个 Region 之间）上来看是基于“复制”算法实现的，这意味着运行期间不会产生内存空间碎片。
- 可预测的停顿：能让使用者明确指定在一个长度为 M 毫秒的时间片段内，消耗在 GC 上的时间不得超过 N 毫秒。

### 8）理解GC日志

通过设置VM参数"XX:+PrintGCDetails"就可以打印出GC日志。

大概是这个样子：
```
5.617:[GC 5.617:[ParNew: 43296K->7006K(47808K), 0.0136826 secs] 44992K->8702K(252608K), 0.0137904 secs][Times: user=0.03 sys=0.00, real=0.02 secs]
```

以下是每个字段的意思：
```
5.617（时间戳）:
[
GC（Young GC） 5.617（时间戳）:
    [ParNew（使用ParNew作为年轻代的垃圾回收期）:
        43296K（年轻代垃圾回收前的大小）
            ->7006K（年轻代垃圾回收以后的大小）(47808K)（年轻代的总大小）,
        0.0136826 secs（回收时间）]

    44992K（堆区垃圾回收前的大小）
        ->8702K（堆区垃圾回收后的大小）(252608K)（堆区总大小）,
    0.0137904 secs（回收时间）
]

[
Times:
user=0.03（Young GC用户耗时）
sys=0.00（Young GC系统耗时）,
real=0.02 secs（Young GC实际耗时）
]
```

## 6. 内存分配与回收策略

先了解两个概念：
- Minor GC：发生在新生代上，因为新生代对象存活时间很短，因此 Minor GC 会频繁执行，执行的速度一般也会比较快。
- Full GC：发生在老年代上，老年代对象和新生代的相反，其存活时间长，因此 Full GC 很少执行，而且执行速度会比 Minor GC 慢很多。

### 1）对象优先在 Eden 分配

大多数情况下，对象在新生代 Eden 区分配，当 Eden 区空间不够时，发起 Minor GC。

### 2）大对象直接进入老年代

大对象是指需要连续内存空间的对象，最典型的大对象是那种很长的字符串以及数组。

经常出现大对象会提前触发垃圾收集以获取足够的连续空间分配给大对象。

-XX:PretenureSizeThreshold，大于此值的对象直接在老年代分配，避免在 Eden 区和 Survivor 区之间的大量内存复制。

### 3）长期存活的对象进入老年代

为对象定义年龄计数器，对象在 Eden 出生并经过 Minor GC 依然存活，将移动到 Survivor 中，年龄就增加 1 岁，增加到一定年龄则移动到老年代中。

-XX:MaxTenuringThreshold 用来定义年龄的阈值。

### 4）动态对象年龄判定

虚拟机并不是永远地要求对象的年龄必须达到 MaxTenuringThreshold 才能晋升老年代，如果在 Survivor 区中相同年龄所有对象大小的总和大于 Survivor 空间的一半，则年龄大于或等于该年龄的对象可以直接进入老年代，无需等到 MaxTenuringThreshold 中要求的年龄。

### 5）空间分配担保

在发生 Minor GC 之前，虚拟机先检查老年代最大可用的连续空间是否大于新生代所有对象总空间，如果条件成立的话，那么 Minor GC 可以确认是安全的；如果不成立的话虚拟机会查看 HandlePromotionFailure 设置值是否允许担保失败，如果允许那么就会继续检查老年代最大可用的连续空间是否大于历次晋升到老年代对象的平均大小，如果大于，将尝试着进行一次 Minor GC，尽管这次 Minor GC 是有风险的；如果小于，或者 HandlePromotionFailure 设置不允许冒险，那这时也要改为进行一次 Full GC。

---

# 未完待续.....