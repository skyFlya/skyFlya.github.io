---
title: Java并发(2) - synchronize
tags:
  - 并发
categories: Java
updated: 1636271467000
date: 2021-11-07 18:55:09
---

> 在 Java 中，关键字 synchronized 可以保证在同一个时刻，只有一个线程可以执行某个方法或者某个代码块,也可以保证一个线程的变化(主要是共享数据的变化)被其他线程所看到

锁就是为达到这种互斥访问目的诞生的,从宏观上锁分为乐观锁和悲观锁

- 乐观锁：认为读多写少，遇到并发写情况较少
	- Java 中 java.util.concurrent.atomic 包下面的原子变量类就是使用了乐观锁的一种实现方式 CAS 实现的
- 悲观锁：即认为写多，遇到并发写情况较多
	- Java 中 synchronized 和 ReentrantLock 等独占锁就是悲观锁思想的实现。

<!-- more -->

> 锁其实就是各个房子的门的关门开门的实现, 不用线程访问不同的锁(门)互不干扰

### 用法

```java
//this,当前实例对象锁
synchronized(this){
    for(int j=0;j<1000000;j++){
        i++;
    }
}
//class对象锁
synchronized(AccountingSync.class){
    for(int j=0;j<1000000;j++){
        i++;
    }
}
```

#### 作用于实例方法

> 所谓的实例对象锁就是用 synchronized 修饰实例对象中的实例方法，注意这不包括静态方法

```java
public class AccountingSync implements Runnable{
    //共享资源(临界资源)
    static int i=0;
 
    /**
     * synchronized 修饰实例方法
     */
    public synchronized void increase(){
        i++;
    }
    @Override
    public void run() {
        for(int j=0;j<1000000;j++){
            increase();
        }
    }
    public static void main(String[] args) throws InterruptedException {
	    test1()
    }
    public void test1(){
        //这个时候两个线程,对象锁是一致的,所以结果无误
        AccountingSync instance=new AccountingSync();
        Thread t1=new Thread(instance);
        Thread t2=new Thread(instance);
        t1.start();
        t2.start();
        t1.join();
        t2.join();
        System.out.println(i);
	}
    /**
     * 输出结果:
     * 2000000
     */
    public void test2(){
        //这个时候两个线程,对象锁是不一致的,所以结果有误
        //将 synchronized 作用于静态的 increase 方法，这样的话，对象锁就当前类对象
        //由于无论创建多少个实例对象，但对于的类对象拥有只有一个，所有在这样的情况下对象锁就是唯一的
        //new新实例
        Thread t1=new Thread(new AccountingSyncBad());
        //new新实例
        Thread t2=new Thread(new AccountingSyncBad());
        t1.start();
        t2.start();
        t1.join();
        t2.join();
        System.out.println(i);
	}
    /**
     * 输出结果有问题:
     * 123537
     */
}
```

#### 作用于静态方法

> 当 synchronized 作用于静态方法时，其锁就是当前类的 class 对象锁。由于静态成员不专属于任何一个实例对象，是类成员，因此通过 class 对象锁可以控制静态成员的并发操作

```java
public class AccountingSyncClass implements Runnable{
    static int i=0;
    // 作用于静态方法,锁是当前class对象,也就是AccountingSyncClass类对应的class对象
    public static synchronized void increase(){
        i++;
    }
 
    // 非静态,访问时锁不一样不会发生互斥
    public synchronized void increase4Obj(){
        i++;
    }
    @Override
    public void run() {
        for(int j=0;j<1000000;j++){
            increase();
        }
    }
    public static void main(String[] args) throws InterruptedException {
        //new新实例
        Thread t1=new Thread(new AccountingSyncClass());
        //new心事了
        Thread t2=new Thread(new AccountingSyncClass());
        //启动线程
        t1.start();t2.start();
        t1.join();t2.join();
        System.out.println(i);
    }
}
```

#### 同步代码块

- synchronized 同步块对同一条线程来说是可重入的，不会出现自己把自己锁死的问题
- 同步块在已进入的线程执行完之前，会阻塞后面其他线程的进入。

```java
public class AccountingSync implements Runnable{
    static AccountingSync instance = new AccountingSync();
    static int i=0;
    @Override
    public void run() {
        //省略其他耗时操作....
        //使用同步代码块对变量i进行同步操作,锁对象为instance
        synchronized(instance){
            for(int j=0;j<1000000;j++){
                    i++;
              }
        }
    }
    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(instance);
        Thread t2 = new Thread(instance);
        t1.start();t2.start();
        t1.join();t2.join();
        System.out.println(i);
    }
}
```

### 原理(理解不了的)
> monitor 和 Java 对象头是实现 synchronized 的基础

#### moniter

JVM 基于进入和退出 Monitor 对象来实现方法同步和代码块同步。代码块同步是使用 monitorenter 和 monitorexit 指令实现的， monitorenter 指令是在编译后插入到同步代码块的开始位置，而 monitorexit 是插入到方法结束处和异常处。任何对象都有一个 monitor 与之关联，当且一个 monitor 被持有后，它将处于锁定状态。

根据虚拟机规范的要求，在执行 monitorenter 指令时，首先要去尝试获取对象的锁，如果这个对象没被锁定，或者当前线程已经拥有了那个对象的锁，把锁的计数器加 1；相应地，在执行 monitorexit 指令时会将锁计数器减 1，当计数器被减到 0 时，锁就释放了。如果获取对象锁失败了，那当前线程就要阻塞等待，直到对象锁被另一个线程释放为止。

#### 对象头

在 JVM 中，对象在内存中的布局分为三块区域：对象头、实例数据和对齐填充。Java 头对象是实现 synchronized 的锁对象的基础,一般而言,synchronized 使用的锁对象是存储在 Java 对象头里的。

![IeGDeI.png](https://z3.ax1x.com/2021/11/04/IeGDeI.png)

### synchronized与ReentrantLock 
- 开发
	- synchronized 隐式锁, 锁的持有与释放都是隐式的
	- ReentrantLock 显示锁, 锁的持有和释放都必须由我们手动编写
- 性能:由于JDK1.6中加入了针对锁的优化措施（见后面），使得synchronized 与 ReentrantLock 的性能基本持平。ReentrantLock 只是提供了 synchronized 更丰富的功能，而不一定有更优的性能，所以在 synchronized 能实现需求的情况下，优先考虑使用 synchronized 来进行同步。

### synchronize 与 volatile 异同

- 性能差异
	- 加锁、解锁的过程是要有性能损耗的
	- volatile 变量的读操作的性能几乎与普通变量无差别,虽说写操作由于需要插入内存屏障所以会慢一些,但开销也比volatile低
- 阻塞
	- volatile 是Java虚拟机提供的一种轻量级同步机制，基于内存屏障实现的。不是锁带来的阻塞和性能损耗的问题
	- synchronize 实现的锁本质上是一种阻塞锁


### Refer

[暴力突破 Java 并发 - synchronize 解析_lerendan的博客-CSDN博客](https://blog.csdn.net/u010289802/article/details/104228091)