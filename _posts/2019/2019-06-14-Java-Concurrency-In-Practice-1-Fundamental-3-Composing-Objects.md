---
layout: post
title: Java 并发编程实战-学习日志（一）3：对象的组合
date: 2019-06-14 15:03:04
author: admin
comments: true
categories: [Java]
tags: [Java, Concurrency, Java Concurrency In Practice]

---



<!-- more -->

* 目录
{:toc}
---

# 设计线程安全的类

在设计线程安全的类时，需要包含以下三个要素：

- 找出构成对象状态的所有变量
- 找出约束状态变量的不变性条件
- 建立对象状态的并发访问管理策略

## 1. 收集同步需求

要确保类的安全，就需要确保它的不变性条件不会在并发的情况下被破坏，这就需要对其状态进行推断。对象和变量都有一个状态空间，即取值范围。状态空间越小，就越容易判断线程的状态。

> 如果不了解对象的不变性条件与后验条件，那么就不能确保线程安全性。要满足在状态变量的有效值或状态转换上的各种约束条件，就需要借助原子性与封装性。
>



## 2. 依赖状态的操作

在某些对象的方法中还包含一些基于状态的先验条件。比如：不能从空队列中移除一个元素，在删除元素前，队列必须处于非空状态。如果在某个操作中包含有基于状态的先验条件，那么这个操作就称为依赖状态的操作。

## 3. 状态的所有权

对象封装了它拥有的状态，那么它就拥有封装状态的所有权。

状态的所有者将决定采用何种加锁协议来维持状态的完整性。所有权意味着控制权。

如果发布了某个可变对象的引用，那么就不再拥有独占的控制权，最多是共享控制权。为了防止多个线程在并发访问同一个对象时产生相互干扰，这些对象应该要么是线程安全的对象，要么是事实不可变的对象，或者用同一个锁保护的对象。

---

# 实例封闭

> 将数据封装在对象内部，可以将数据的访问限制在对象的方法上，从而更容易确保线程在访问数据是总能持有正确的锁。

被封闭对象一定不能超过它们既定的作用域。

对象可以封闭在类的一个实例中，比如：作为类的一个私有成员；或者封闭在某个作用域内，比如：作为一个局部变量；或者封闭在线程中，比如：在某个线程中将对象从一个方法传递到另一个方法，而不是在多个线程之间共享该对象。

```java
@ThreadSafe
public class PersonSet {
	@GuardedBy("this")
	private final Set<Person> mySet = new HashSet<Person>();
	
	public synchronized void addPerson(Person p){
		mySet.add(p);
	}
	
	public synchronized boolean containsPerson(Person p){
		return mySet.contains(p);
	}
}
```

> 封闭机制更易于构造线程安全的类，因为当封闭类的状态时，在分析类的线程安全性时就无须检测整个程序。

## 1. Java 监视器模式

```java
public class PrivateLock {
    
    private final Object myLock = new Object();
    @GuardedBy("myLock") Widget widget;
    
    void someMethod() {
        synchronized(myLock) {
            // Access or modify the state of widget
        }
    }
} 
```
使用私有对象锁，比起使用对象的内置锁，有许多优点。

- 私有的锁对象可以将锁封装起来，使客户代码无法得到锁，但客户代码可以通过公有方法来访问锁，以便参与到同步策略
- 如果客户代码错误地获得了另一个对象的锁，那么可能会产生活跃性问题
- 要想验证某个公有访问的锁在程序中是否被正确的使用，需要检查整个程序，而私有锁只需检查单个类即可

## 2. 示例：车辆追踪

每台车由一个 string 对象来标识，并且拥有一个相应的位置坐标。`VehicleTracker` 封装了车辆的标识和位置。整个模型包含一个视图线程，和多个执行更新操作的线程。

视图线程会读取车辆的名字和位置，并显示出来：

```java
Map<String, Point> locations = vehicles.getLocations();
for (String key : locations.keySet()){
    renderVehicle(key, locations.get(key)); 
}
```

执行更新操作的线程会修改车辆的位置：

```java
void vehicleMoved(VehicleMovedEvent evt) {
	Point loc = evt.getNewLocation();
	vehicles.setLocation(evt.getVehicleId(), loc.x, loc.y);
}
```

基于监视器模式的“车辆追踪器”：

```java
@ThreadSafe
public class MonitorVehicleTracker {
    @GuardedBy("this")
    private final Map<String, MutablePoint> locations;
    
    public MonitorVehicleTracker(Map<String, MutablePoint> locations) {
        this.locations = deepCopy(locations);
    }
    
    public synchronized Map<String, MutablePoint> getLocations() {
        return deepCopy(locations);
    }
    
    public synchronized MutablePoint getLocation(String id) {
        MutablePoint loc = locations.get(id);
        return loc == null ? null : new MutablePoint(loc);
    }
    
    public synchronized void setLocation(String id, int x, int y) {
        MutablePoint loc = locations.get(id);
        if (loc == null)
            throw new IllegalArgumentException("No such ID: " + id);
        loc.x = x;
        loc.y = y;
    }
    
    private static Map<String, MutablePoint> deepCopy(
        Map<String, MutablePoint> m) {
        Map<String, MutablePoint> result =
            new HashMap<String, MutablePoint>();
        for (String id : m.keySet())
            result.put(id, new MutablePoint(m.get(id)));
        return Collections.unmodifiableMap(result);
    }
}
```

其中用到的 `MutablePoint`定义如下：

```java
@NotThreadSafe
public class MutablePoint {
    public int x, y;
    public MutablePoint() { x = 0; y = 0; }
    public MutablePoint(MutablePoint p) {
        this.x = p.x;
        this.y = p.y;
    }
} 
```

---

# 线程安全性的委托

在组合对象中，如果每个组件都已经是线程安全的，是否需要再加一个额外的“线程安全层“？答案是：需要视情况而定。

## 1. 示例：基于委托的车辆追踪器

构建一个线程安全的 Point 类。

```java
@Immutable
public class Point {
    public final int x, y;
    public Point(int x, int y) {
        this.x = x;
        this.y = y;
    }
} 
```

将线程安全委托给 ConcurrentHashMap：

```java
@ThreadSafe
public class DelegatingVehicleTracker {
    private final ConcurrentMap<String, Point> locations;
    private final Map<String, Point> unmodifiableMap;
    public DelegatingVehicleTracker(Map<String, Point> points) {
        locations = new ConcurrentHashMap<String, Point>(points);
        unmodifiableMap = Collections.unmodifiableMap(locations);
    }
    public Map<String, Point> getLocations() {
        return unmodifiableMap;
    }
    public Point getLocation(String id) {
        return locations.get(id);
    }
    public void setLocation(String id, int x, int y) {
        if (locations.replace(id, new Point(x, y)) == null)
            throw new IllegalArgumentException(
            "invalid vehicle name: " + id);
    }
} 
```

## 2. 独立的状态变量

将线程安全性委托给多个状态变量，这些变量都是各自独立的，所以不存在线程安全问题。

```java
public class VisualComponent { 
    private final List<KeyListener> keyListeners = new CopyOnWriteArrayList<KeyListener>();
    private final List<MouseListener> mouseListeners = new CopyOnWriteArrayList<MouseListener>();

    public void addKeyListener(KeyListener listener) {
        keyListeners.add(listener);
    }

    public void addMouseListener(MouseListener listener) {
        mouseListeners.add(listener);
    }

    public void removeKeyListener(KeyListener listener) {
        keyListeners.remove(listener);
    }

    public void removeMouseListener(MouseListener listener) {
        mouseListeners.remove(listener);
    }
} 
```

## 3. 当委托失效时

以下是一个不好的例子：

```java
public class NumberRange {
    // INVARIANT: lower <= upper
    private final AtomicInteger lower = new AtomicInteger(0);
    private final AtomicInteger upper = new AtomicInteger(0);
    public void setLower(int i) {
        // Warning -- unsafe check-then-act
        if (i > upper.get())
            throw new IllegalArgumentException(
            "can't set lower to " + i + " > upper");
        lower.set(i);
    }
    public void setUpper(int i) {
        // Warning -- unsafe check-then-act
        if (i < lower.get())
            throw new IllegalArgumentException(
            "can't set upper to " + i + " < lower");
        upper.set(i);
    }
    public boolean isInRange(int i) {
        return (i >= lower.get() && i <= upper.get());
    }
} 
```

以上的代码中，使用了两个 AtomicInteger 来管理状态，但是它们之间包含了一个约束条件：第一个要小于第二个。

此时，NumberRange 就不是线程安全的，这时就需要加锁了。

## 4. 发布底层的状态变量

如果一个状态变量是线程安全的，并且没有任何不变性条件来约束它的值，在变量的操作上也不存在任何不允许的状态转换，那么就可以安全地发布这个变量。

