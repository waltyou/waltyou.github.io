---
layout: post
title: 《深入理解Java虚拟机：JVM高级特性与最佳实践--第二版》学习日志（二）：自动内存管理机制
date: 2018-5-18 21:06:04
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


---

# 未完待续.....