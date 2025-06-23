---
layout: post
title: 《Effective Java》学习日志（四）29：偏爱泛型类型
date: 2018-11-08 19:01:04
author: admin
comments: true
categories: [Java]
tags: [Java,Effective Java]
---

参数化声明并使用JDK提供的泛型类型和方法通常不会太困难。 
但编写自己的泛型类型有点困难，但值得努力学习。


<!-- more -->

---

学习资料主要参考： 《Effective Java Third Edition》，作者：Joshua Bloch

---




* 目录
{:toc}

---

## 实际例子

简单堆栈实现：

    // Object-based collection - a prime candidate for generics
    public class Stack {
        private Object[] elements;
        private int size = 0;
        private static final int DEFAULT_INITIAL_CAPACITY = 16;
    
        public Stack() {
            elements = new Object[DEFAULT_INITIAL_CAPACITY];
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
    
        public boolean isEmpty() {
            return size == 0;
        }
    
        private void ensureCapacity() {
            if (elements.length == size)
                elements = Arrays.copyOf(elements, 2 * size + 1);
        }
    }
    
这个类应该已经被参数化了，但是由于事实并非如此，我们可以对它进行泛型化。 
换句话说，我们可以参数化它，而不会损害原始非参数化版本的客户端。 
就目前而言，客户端必须强制转换从堆栈中弹出的对象，而这些强制转换可能会在运行时失败。
 
## 第一步

泛型化类的第一步是在其声明中添加一个或多个类型参数。 
在这种情况下，有一个类型参数，表示堆栈的元素类型，这个类型参数的常规名称是E（条目 68）。

下一步是用相应的类型参数替换所有使用的Object类型，然后尝试编译生成的程序：

    // Initial attempt to generify Stack - won't compile!
    public class Stack<E> {
        private E[] elements;
        private int size = 0;
        private static final int DEFAULT_INITIAL_CAPACITY = 16;
    
        public Stack() {
            elements = new E[DEFAULT_INITIAL_CAPACITY];
        }
    
        public void push(E e) {
            ensureCapacity();
            elements[size++] = e;
        }
    
        public E pop() {
            if (size == 0)
                throw new EmptyStackException();
            E result = elements[--size];
            elements[size] = null; // Eliminate obsolete reference
            return result;
        }
        ... // no changes in isEmpty or ensureCapacity
    }
    
你通常会得到至少一个错误或警告，这个类也不例外。 幸运的是，这个类只产生一个错误：

    Stack.java:8: generic array creation
            elements = new E[DEFAULT_INITIAL_CAPACITY];
                   ^
如条目 28所述，你不能创建一个不可具体化类型的数组，例如类型E。每当编写一个由数组支持的泛型时，就会出现此问题。 

有两种合理的方法来解决它。 

### 第一种解决方案

第一种解决方案直接规避了对泛型数组创建的禁用：创建一个Object数组并将其转换为泛型数组类型。 
现在没有了错误，编译器会发出警告。 
这种用法是合法的，但不是（一般）类型安全的：

    Stack.java:8: warning: [unchecked] unchecked cast
    found: Object[], required: E[]
            elements = (E[]) new Object[DEFAULT_INITIAL_CAPACITY];
                       ^
                       
编译器可能无法证明你的程序是类型安全的，但你可以。 你必须说服自己，不加限制的类型强制转换不会损害程序的类型安全。 
有问题的数组（元素）保存在一个私有属性中，永远不会返回给客户端或传递给任何其他方法。 
保存在数组中的唯一元素是那些传递给push方法的元素，它们是E类型的，所以未经检查的强制转换不会造成任何伤害。

一旦证明未经检查的强制转换是安全的，请尽可能缩小范围（条目 27）。 在这种情况下，构造方法只包含未经检查的数组创建，所以在整个构造方法中抑制警告是合适的。 

通过添加一个注解来执行此操作，Stack可以干净地编译，并且可以在没有显式强制转换或担心ClassCastException异常的情况下使用它：

    // The elements array will contain only E instances from push(E).
    // This is sufficient to ensure type safety, but the runtime
    // type of the array won't be E[]; it will always be Object[]!
    @SuppressWarnings("unchecked")
    public Stack() {
        elements = (E[]) new Object[DEFAULT_INITIAL_CAPACITY];
    }
    
### 第二种解决方案

消除Stack中的泛型数组创建错误的第二种方法是将属性元素的类型从E[]更改为Object []。
如果这样做，会得到一个不同的错误：

    Stack.java:19: incompatible types
    found: Object, required: E
            E result = elements[--size];
                               ^
                               
可以通过将从数组中检索到的元素转换为E来将此错误更改为警告：

    Stack.java:19: warning: [unchecked] unchecked cast
    found: Object, required: E
            E result = (E) elements[--size];
                               ^
因为E是不可具体化的类型，编译器无法在运行时检查强制转换。 
再一次，你可以很容易地向自己证明，不加限制的转换是安全的，所以可以适当地抑制警告。 
根据条目 27的建议，我们只在包含未经检查的强制转换的分配上抑制警告，而不是在整个pop方法上：

    // Appropriate suppression of unchecked warning
    public E pop() {
        if (size == 0)
            throw new EmptyStackException();
    
        // push requires elements to be of type E, so cast is correct
        @SuppressWarnings("unchecked") E result =
            (E) elements[--size];
    
        elements[size] = null; // Eliminate obsolete reference
        return result;
    }

## 两种方法的比较

两种消除泛型数组创建的技术都有其追随者。 

第一个更可读：数组被声明为E []类型，清楚地表明它只包含E实例。 
它也更简洁：在一个典型的泛型类中，你从代码中的许多点读取数组; 第一种技术只需要一次转换（创建数组的地方），而第二种技术每次读取数组元素都需要单独转换。 
因此，第一种技术是优选的并且在实践中更常用。 

但是，它确实会造成堆污染（heap pollution）（条目 32）：数组的运行时类型与编译时类型不匹配（除非E碰巧是Object）。 
这使得一些程序员非常不安，他们选择了第二种技术，尽管在这种情况下堆的污染是无害的。

下面的程序演示了泛型Stack类的使用。 
该程序以相反的顺序打印其命令行参数，并将其转换为大写。 
对从堆栈弹出的元素调用String的toUpperCase方法不需要显式强制转换，而自动生成的强制转换将保证成功：

    // Little program to exercise our generic Stack
    public static void main(String[] args) {
        Stack<String> stack = new Stack<>();
        for (String arg : args)
            stack.push(arg);
        while (!stack.isEmpty())
            System.out.println(stack.pop().toUpperCase());
    }
    
上面的例子似乎与条目 28相矛盾，条目 28中鼓励使用列表优先于数组。 
在泛型类型中使用列表并不总是可行或可取的。 
Java本身生来并不支持列表，所以一些泛型类型（如ArrayList）必须在数组上实现。 其他的泛型类型，比如HashMap，是为了提高性能而实现的。

绝大多数泛型类型就像我们的Stack示例一样，它们的类型参数没有限制：可以创建一个Stack<Object>，Stack<int[]>，Stack<List<String>>或者其他任何对象的Stack引用类型。 
请注意，不能创建基本类型的堆栈：尝试创建Stack<int>或Stack<double>将导致编译时错误。 

这是Java泛型类型系统的一个基本限制。 可以使用基本类型的包装类（条目 61）来解决这个限制。

有一些泛型类型限制了它们类型参数的允许值。 例如，考虑java.util.concurrent.DelayQueue，它的声明如下所示：

    class DelayQueue<E extends Delayed> implements BlockingQueue<E>
    
类型参数列表（<E extends Delayed>）要求实际的类型参数E是java.util.concurrent.Delayed的子类型。 
这使得DelayQueue实现及其客户端可以利用DelayQueue元素上的Delayed方法，而不需要显式的转换或ClassCastException异常的风险。 
类型参数E被称为限定类型参数。 
请注意，子类型关系被定义为每个类型都是自己的子类型[JLS，4.10]，因此创建DelayQueue <Delayed>是合法的。

## 总结

总之，泛型类型比需要在客户端代码中强制转换的类型更安全，更易于使用。 
当你设计新的类型时，确保它们可以在没有这种强制转换的情况下使用。 
这通常意味着使类型泛型化。 

如果你有任何现有的类型，应该是泛型的但实际上却不是，那么把它们泛型化。 这使这些类型的新用户的使用更容易，而不会破坏现有的客户端（条目 26）。