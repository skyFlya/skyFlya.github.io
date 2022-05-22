---
title: Java并发(3) - Lock
tags:
  - 并发
categories: Java
updated: 1636282691000
date: 2021-11-07 18:59:09
---

> 锁是用于通过多个线程控制对共享资源的访问的工具

```java
Lock lock = new ReentrantLock();
lock.lock();
try{
    //临界区......
}finally{
    //最好不要把获取锁的过程写在 try 语句块中，因为如果在获取锁时发生了异常，异常抛出的同时也会导致锁无法被释放
    lock.unlock();
}
```
<!-- more -->
### 特性和方法

提供synchronized不具备的主要特性:

| 特性 | 描述 |
|:---------------------:| :-----------------------------:|
| 以非阻塞地获取锁 | 当前线程尝试获取锁，如果这一时刻锁没有被其他线程获取到，则成功获取并持有锁|
| 可以被中断地获取锁 | 获取到锁的线程能够响应中断，当获取到锁的线程被中断时，中断异常将会被抛出，同时锁会被释放 |
| 超时的方式获取锁|  在指定的截止时间之前获取锁,超时后仍旧无法获取则返回 |


| Method | 描述 |
|:---------------------:| :-----------------------------:|
| lock() |   
获取锁。如果锁不可用,则当前线程将被禁用以进行线程调度，并处于休眠状态，等待直到获取锁。 |
| lockInterruptibly()| 可以中断的去获取锁 |
| newCondition()| 返回当前锁的Condition(等待条件),在Condition之前，当前线程必须持有锁,调用Condition.wait 当前线程将释放锁 |
| tryLock()| 直接获取锁,可以true,不可以false |
| tryLock(time,unit)| 超时获取锁,有三种情况:1.当前线程在超时时间内获得了锁;2.当前线程被中断;3.超时时间结束，返回false|
| unlock() | 释放锁 |


### ReentrantLock

> ReentrantLock 和 synchronized声明一样可以用来实现线程之间的同步互斥，但在功能上比 synchronized声明 更强大而且更灵活


| 构造Method | 描述 |
|--|--|
| ReentrantLock() | 创建一个 ReentrantLock的实例 |
| ReentrantLock(boolean fair) | 创建一个特定锁类型（公平锁/非公平锁）的ReentrantLock的实例|


- 公平锁: 先来先得, 线程调度有序
- 非公平锁: 一种获取锁的抢占机制,谁快谁得,线程调度无序


| 调用Method | 描述 |
|--|--|
| getHoldCount() | 调用lock()方法的次数 |
| getOwner() | 返回当前拥有此锁的线程，如果不拥有，则返回 null|
| getQueuedThreads() | 返回正在等待获取此锁的线程的集合 |
| getWaitingThreads() | 返回与此锁相关联的Condition下等待的线程的集合 |
| hasQueuedThread() | 查询给定线程是否等待获取锁|
| hasQueuedThreads()| 查询否有线程正在等待获取锁|
| hasWaiters() | 查询任何线程是否等待与此锁相关联的给定条件 |
| isHeldByCurrentThread() |查询此锁是否由当前线程持有|

```java
public class ReentrantLockTest {
    public static void main(String[] args) {
        MyService service = new MyService();
        MyThread a1 = new MyThread(service);
        MyThread a2 = new MyThread(service);
        MyThread a3 = new MyThread(service);
        a1.start();
        a2.start();
        a3.start();
    }
 
    static public class MyService {
        private Lock lock = new ReentrantLock();
        public void testMethod() {
            lock.lock();
            try {
              System.out.println("ThreadName=" + Thread.currentThread().getName() + (" " + (i + 1)));
            } finally {
              lock.unlock();
            }
        }
    }
    // 结果:当一个线程运行完毕后才把锁释放，其他线程才能执行
    static public class MyThread extends Thread {
        private MyService service;
        public MyThread(MyService service) {
            super();
            this.service = service;
        }
        @Override
        public void run() {
            service.testMethod();
        }
    }
}
```

### Condition

> 线程对象可以注册指定的Condition, 从而可以有选择性的进行线程通知,使得线程调度上更加灵活

| Method | 描述 |
|:--:|:--:|
| await() | 使当前线程等待，直到它收到信号或被中断 |
| signal() | 发出信号,唤醒一个等待线程 |

思路理清:
1. 首先`a.start()` 输出`"await"`,然后进入`condition.await()`
2. 3s后`service.signal()` 输出`"signal"`
3. 进入`condition.signal()` , 3s后输出`"最好的signal()的语句"`
4. 再之后输出`"signal之后的语句"`

等待/通知的实现:
```java
public class UseSingleConditionWaitNotify {
    public static void main(String[] args) throws InterruptedException {
        MyService service = new MyService();
        ThreadA a = new ThreadA(service);
        a.start();
        Thread.sleep(3000);
        service.signal();
	}
 
    static public class MyService {
        private Lock lock = new ReentrantLock();
        public Condition condition = lock.newCondition();
        public void await() {
            lock.lock();
            try {
                System.out.println(" await");
                condition.await();
                System.out.println("signal之后的语句");
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                lock.unlock();
            }
        }
 
        public void signal() throws InterruptedException {
            lock.lock();
            try {				
                System.out.println("signal");
                condition.signal();
                Thread.sleep(3000);
                System.out.println("最好的signal()的语句");
            } finally {
                lock.unlock();
            }
        }
    }
    
    static public class ThreadA extends Thread {
        private MyService service;
        public ThreadA(MyService service) {
            super();
            this.service = service;
        }
        @Override
        public void run() {
            service.await();
        }
    }
}
```


### ReadWriteLock 接口的实现类：ReentrantReadWriteLock

>  ReentrantLock（排他锁）具有完全互斥排他的效果，即同一时刻只允许一个线程访问,效率低下,ReadWriteLock 接口的实现类 ReentrantReadWriteLock 读写锁就是为了解决这个问题。读写锁维护了两个锁，一个是读操作相关的锁成为共享锁，一个是写操作相关的锁为排他锁。通过分离读锁和写锁，其并发性比一般排他锁有了很大提升


- 读读共享
- 写写互斥
- 读写互斥

```java
    private ReentrantReadWriteLock lock = new ReentrantReadWriteLock();
    // 读读共享情况,输出可以发现两个线程几乎可以同时运行
    public void read() {
        try {
            try {
                lock.readLock().lock();
                System.out.println("获得读锁" + Thread.currentThread().getName()
                        + " " + System.currentTimeMillis());
                Thread.sleep(10000);
            } finally {
                lock.readLock().unlock();
            }
        } catch (InterruptedException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        }
    }
```