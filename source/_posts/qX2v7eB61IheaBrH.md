---
title: Java并发(4) - CAS
tags:
  - 并发
categories: Android
updated: 1636282770000
date: 2021-11-07 18:59:57
---


锁机制问题:
- 在多线程竞争下，加锁、释放锁会导致比较多的上下文切换和调度延时，引起性能问题
- 一个线程持有锁会导致其它所有需要此锁的线程挂起
- 如果一个优先级高的线程等待一个优先级低的线程释放锁会导致优先级倒置，引起性能风险

volatile 是不错的机制，但是 volatile 不能保证原子性，因此对于同步最终还是要回到锁机制上来。独占锁是一种悲观锁，synchronized 就是一种独占锁，会导致其它所有需要锁的线程挂起，等待持有锁的线程释放锁。而另一个更加有效的锁就是乐观锁

> 乐观锁即总是假设最好的情况，每次去拿数据的时候都认为别人不会修改，所以不会上锁，但是在更新的时候会判断一下在此期间别人有没有去更新这个数据，可以使用版本号机制和 CAS 算法实现
<!-- more -->

### 原子操作

> 所谓“原子”操作，是指一组不可分割的操作：操作者对目标对象进行操作时，要么完成所有操作后其他操作者才能操作；要么这个操作者不能进行任何操作



![Ie5YA1.png](https://z3.ax1x.com/2021/11/04/Ie5YA1.png)

```
public class TestAtomic {
    public static void main(String[] args) throws Exception {
        // 实例化了一个AtomicInteger类的对象atomic并定义初始值为1
        AtomicInteger atomic = new AtomicInteger(1);
        // 进行atomic的原子化操作：增加1并且获取这个增加后的新值
        atomic.incrementAndGet();
    }
}
```

### CAS 原理

> CAS 的思想很简单，三个参数：当前内存值 V、旧的预期值 A、即将更新的值 B，当且仅当预期值 A 和内存值 V 相同时，将内存值修改为 B 并返回 true，否则什么都不做，并返回 false。我们拿 AtomicInteger 类来分析，先来看看 AtomicInteger 静态代码块片段


### CAS缺点

- 循环时间长开销很大
- 只能保证一个共享变量的原子操作
- ABA问题

### 例子

1.在内存地址V当中，存储着值为10的变量。


![q9xFtf.png](https://s1.ax1x.com/2022/03/17/q9xFtf.png)

2.此时线程1想要把变量的值增加1。对线程1来说，旧的预期值A=10，要修改的新值B=11。


![q9x90I.png](https://s1.ax1x.com/2022/03/17/q9x90I.png)

3.在线程1要提交更新之前，另一个线程2抢先一步，把内存地址V中的变量值率先更新成了11。


![q9vv1e.png](https://s1.ax1x.com/2022/03/17/q9vv1e.png)

4.线程1开始提交更新，首先进行A和地址V的实际值比较（Compare），发现A不等于V的实际值，提交失败。


![q9v56J.png](https://s1.ax1x.com/2022/03/17/q9v56J.png)

5.线程1重新获取内存地址V的当前值，并重新计算想要修改的新值。此时对线程1来说，A=11，B=12。这个重新尝试的过程被称为自旋。


![q9v8OA.png](https://s1.ax1x.com/2022/03/17/q9v8OA.png)

6.这一次比较幸运，没有其他线程改变地址V的值。线程1进行Compare，发现A和地址V的实际值是相等的。

![q9vmo6.png](https://s1.ax1x.com/2022/03/17/q9vmo6.png)

7.线程1进行SWAP，把地址V的值替换为B，也就是12。

![q9j9vd.png](https://s1.ax1x.com/2022/03/17/q9j9vd.png)

从思想上来说，Synchronized属于悲观锁，悲观地认为程序中的并发情况严重，所以严防死守。CAS属于乐观锁，乐观地认为程序中的并发情况不那么严重，所以让线程不断去尝试更新，最终可以理解成一个无阻塞多线程争抢资源的模型

 
### 参考

[Java：CAS(乐观锁) ](https://www.jianshu.com/p/ae25eb3cfb5d)