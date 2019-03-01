---
layout: post
title: 《HeadFirst设计模式》学习日志（九）:迭代器与组合模式
date: 2019-03-01 16:58:00
author: admin
comments: true
categories: [Java]
tags: [Java,Design Patterns]
---


<!-- more -->

------

学习资料主要参考： 《Head First 设计模式》

------

## 目录
{:.no_toc}

* 目录
{:toc}
------

# 1. 迭代器模式

## 定义

> 提供了一种方法顺序访问一个集合对象中的各个元素，而又不暴露其内部的表示。

```java
public Interface Iterator<E>{
    boolean hasNext();
    E next();
}
```

## 单一责任原则

> 一个类应该只有一个引起变化的原因。

## 迭代器与集合

Java Collection Framework 中包含了很多集合实现，如 ArrayList，LinkedList，HashTable等。这些类都实现了java.util.Collection 接口，这个接口中包含了许多方法。

Java 5 之后，允许 for/in 的方法遍历集合。

## 局限性

- 当面对树形结构的遍历时，迭代器力有不逮。
- 遍历的灵活度较低

# 2. 组合模式

## 定义

> 允许你将对象组合成树形结构来表现“整体/部分”层次结构。
>
> 组合能让客户以一致的方式处理个别对象以及对象组合。

[![](/images/posts/composite-pattern.png)](/images/posts/composite-pattern.png)

## 组合迭代器

```java
public class CompositeIterator implements Iterator{
    // 堆栈数据结构:后进先出 pop取后删掉，peek只取不删
    Stack<Iterator> stack = new Stack<Iterator>();

    /**
     * 将顶层迭代器对象抛入一个堆栈数据结构中
     * @param iterator  顶层迭代器对象
     */
    public CompositeIterator(Iterator iterator) {
        stack.push(iterator);
    }

    @Override
    public boolean hasNext() {
        if(stack.empty()){
            return false;
        }else{
            // 如果不为空,从堆顶取出迭代器
            Iterator iterator = stack.peek();
            //如果没有下一个元素,弹出堆栈
            if(!iterator.hasNext()){
                // 移除此迭代器对象
                stack.pop();
                //递归检查下一个迭代器
                return hasNext();
            }else{
                return true;
            }
        }
    }

    @Override
    public Object next() {
        if(hasNext()){
            Iterator iterator = stack.peek();
            MenuComponent component = (MenuComponent)iterator.next();

            //如果是菜单对象,将本层迭代器放入堆栈,准备读取
            if(component instanceof Menu){
                stack.push(component.createIterator());
            }
            return component;
        }else{
            return null;
        }

    }

}
```



## 空迭代器

```java
public class NullIterator implements Iterator{
    public Object next(){
        return null;
    }
    
    public boolean hasNext(){
        return false;
    }
    
    public void remove(){
        throw new UnsupportedException();
    }
}
```

