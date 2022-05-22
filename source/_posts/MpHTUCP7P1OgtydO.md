---
title: Java并发(7) - ThreadLocal
tags:
  - 并发
categories: Java
updated: 1636283135000
date: 2021-11-07 19:06:09
---


> ThreadLocal 是一个关于创建线程局部变量的类。通常情况下，我们创建的变量是可以被任何一个线程访问并修改的。而使用 ThreadLocal 建的变量只能被当前线程访问，其他线程则无法访问和修改


```java
private void testThreadLocal() {
    Thread t = new Thread() {
        ThreadLocal<String> mStringThreadLocal = new ThreadLocal<>();
 
        @Override
        public void run() {
            super.run();
            mStringThreadLocal.set("name");
            mStringThreadLocal.get();
        }
    };
 
    t.start();
}
```
<!-- more -->
### ThreadLocal实例化

> 为 ThreadLocal 设置默认的 get 初始值，需要重写 initialValue 方法

```java
ThreadLocal<String> mThreadLocal = new ThreadLocal<String>() {
    @Override
    protected String initialValue() {
      return Thread.currentThread().getName();
    }
};
```

在 Android 中的应用，Looper 类就是利用了 ThreadLocal 的特性，保证每个线程只存在一个 Looper 对象:
```java
static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
}
```

### 原理

#### ThreadLocal.set()

首先获取当前线程，利用当前线程作为句柄获取一个 ThreadLocalMap 的对象，如果上述 ThreadLocalMap 对象不为空，则设置值，否则创建这个 ThreadLocalMap 对象并设置值
```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) map.set(this, value);
    else createMap(t, value);
}
```
获取Thread 对象的threadLocals 变量
```java
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
...
class Thread implements Runnable {
    ThreadLocal.ThreadLocalMap threadLocals = null;
}
```
如果一开始未设置,则新建 ThreadLocalMap 对象，并设置初始值
```java
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

![InDvFg.png](https://z3.ax1x.com/2021/11/05/InDvFg.png)

#### ThreadLocal.get()

```java
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
```

### 对象存放

- 在 Java 中,栈内存归属于线程私有,每个线程都会有一个栈内存,其存储的变量只能在其所属线程中可见,即栈内存可以理解成线程的私有内存
- 堆内存中的对象对所有线程可见,堆内存中的对象可以被所有线程访问
- ThreadLocal 实例实际上也是被其创建的类持有（更顶端应该是被线程持有）,位于堆上


### 内存泄露问题

当使用 ThreadLocal 保存一个 value 时，会在 ThreadLocalMap 中的数组插入一个Entry对象，ThreadLocal 在没有外部强引用时，发生 GC 时会被回收，但如果创建 ThreadLocal 的线程一直持续运行，那么这个 Entry 对象中的 value 就有可能一直得不到回收，发生内存泄露

解决:使用完 ThreadLocal 之后,记得调用 remove 方法