---
layout: post
title: Java 并发编程实战-学习日志（二）2：取消与关闭
date: 2019-08-15 17:15:04
author: admin
comments: true
categories: [Java]
tags: [Java, Concurrency, Java Concurrency In Practice]
---

Java没有提供任务安全结束线程的机制，提供了中断，这是一种协作机制：使一个线程终止另一个线程。

为什么是协作机制：1、立即停止会造成数据结构的不一致性 2、任务本身比其他线程更懂得如何清除当前正在执行的任务

软件质量的区别：良好的软件能很好的处理失败、关闭、结束等过程。

<!-- more -->

---

* 目录
{:toc}
---

# 任务取消


外部代码，能够将某个操作正常完成之前，将其置入完成状态，那么这个操作就称为**可取消的（Cancellable）**。

取消操作的原因有很多：
1. 用户请求取消。
2. 有时间限制的操作，如超时设定。
3. 应用程序事件。
4. 错误。
5. 关闭。

如下面这种取消操作实现：

```java
/**
 * 一个可取消的素数生成器
 * 使用volatile类型的域保存取消状态
 * 通过循环来检测任务是否取消
 */
@ThreadSafe
public class PrimeGenerator implements Runnable {
	private final List<BigInteger> primes = new ArrayList<>();
	private volatile boolean canceled;
	
	@Override
	public void run() {
		BigInteger p = BigInteger.ONE;
		while (!canceled){
			p = p.nextProbablePrime();
			synchronized (this) { //同步添加素数
				primes.add(p);
			}
		}
	}
	
	/**
	 * 取消生成素数
	 */
	public void cancel(){
		canceled = true;
	}
	
	/**
	 * 同步获取素数
	 * @return 已经生成的素数
	 */
	public synchronized List<BigInteger> get(){
		return new ArrayList<>(primes);
	}
}
```

其测试用例:

```java
public class PrimeGeneratorTest {
	public static void main(String[] args) {
		PrimeGenerator pg = new PrimeGenerator();
		new Thread(pg).start();
		
		try {
			Thread.sleep(1000);
		} catch (InterruptedException e) {
			e.printStackTrace();
		} finally{
			pg.cancel(); //始终取消
		}
		
		System.out.println("all primes: " + pg.get());
	}
}
```



## 1. 中断

- 调用`interrupt`并不意味者立即停止目标线程正在进行的工作，而只是传递了请求中断的消息。会在下一个取消点中断自己，如wait，sleep，join等。

- 通常，中断是实现取消的最合理方式。





## 未完待续。。。。。。




