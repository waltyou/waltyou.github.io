---
layout: post
title: 《HeadFirst设计模式》学习日志（十一）:代理模式
date: 2019-03-04 13:45:00
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

代理模式，控制和管理对象的访问。

## 远程代理

远程代理就好比“远程对象的本地代表”。远程对象是指活在不同的 JVM 堆中的对象，本地代表是指可以由本地方法调用的对象，其行为会转发到远程对象中。

## RMI(Remote Method Invocation）

RMI(Remote Method Invocation），远程方法调用。

它提供了客户端辅助对象和服务端辅助对象，为客户端辅助对象创建与客户端辅助对象相同的方法。

使用 RMI ，你不必亲自写任何网络和 I/O 代码。

**术语:** 客户端辅助对象称为 stub，服务端辅助对象成为 skeleton。

### 制作远程服务

1. 制作远程接口。stub 和实际服务者都需要实现这个接口。
   1. 扩展 java.rmi.Remote 接口
   2. 声明所有方法都抛出 RemoteException
   3. 确定变量属于原语（primitive），还是可序列化对象（Serializable）
2. 制作远程的实现。这是客户真正想要调用的对象。
   1. 实现远程接口
   2. 拓展 java.rmi.server.UnicastRemoteObject ，这个超类可以帮我们完成一些“远程”的功能
   3. 设计一个不带变量的构造器，并声明 RemoteException
   4. 用 RMI Registry 注册此服务：使用 java.rmi.Naming 的静态 rebind() 方法。
3. 利用 rmic 方法产生的 stub 和 skeleton。这是 JDK 自带的命令。
4. 启动 RMI registry，客户可以从中查到代理位置。
5. 开始远程服务。先让服务对象运行起来，服务实现类会实例化一个对象，注册到 RMI Registry 中，以供客户端调用。

使用 Naming.lookup 方法来寻找 stub 对象。

## 代理模式定义

> 为另一个对象提供一个替身或占位符以控制对这个对象的访问。

**类图**

[![](/images/posts/proxy-pattern.png)](/images/posts/proxy-pattern.png)

### 虚拟代理

作为创建开销大的对象的代表。

当对象在创建前或创建中时，由虚拟代理来扮演对象的替身。当对象创建完成后，代理就会将请求直接委托给对象。

### 动态代理

在运行时动态创建代理。
