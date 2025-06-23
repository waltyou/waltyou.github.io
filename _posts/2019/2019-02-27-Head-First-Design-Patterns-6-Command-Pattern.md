---
layout: post
title: 《HeadFirst设计模式》学习日志（六）:命令模式
date: 2019-02-27 19:23:00
author: admin
comments: true
categories: [Java]
tags: [Java,Design Patterns]
---



<!-- more -->

------

学习资料主要参考： 《Head First 设计模式》

------




* 目录
{:toc}
------

命令模式可以将 “动作的请求者” 从 “动作的执行者” 对象中解耦。

## 定义

将“请求”封装成对象，以便使用不同的请求、队列或者日志来参数化其他对象。命令模式也可支持可撤销的操作。

类图：

[![](/images/posts/command-pattern.png)](/images/posts/command-pattern.png)

```java
// 接口
public interface Command{
    public void execute();
    public void undo();
}
// 被封装的执行者
public LightOnCommand implements Command(){
    Light light;
    
    public LightOnCommand(Light light){
        this.light = light;
    }
    public void execute(){
        light.on();
    }
    
    public void undo(){
    	light.off();    
    }
}
```

## 宏命令

一个动作，执行多个命令。

只需要创建一个新的命令类，在它内部保存一个命令的集合即可。

## 更多用途

### 队列请求

可以将许多命令放在队列中，然后依次取出，直接调用它的 execute 方法即可。（有点像 Java 线程池啊）

### 日志请求

可以将日志抽象为一个命令对象，它可以被执行。这样子，只要记录好所有命令对象，然后调用它的 execute 方法就好了。