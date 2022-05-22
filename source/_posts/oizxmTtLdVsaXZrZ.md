---
title: Java并发(6) - 线程池
tags:
  - 并发
categories: Java
updated: 1636283081000
date: 2021-11-07 19:05:17
---

线程池对比new Thread()
- 重用存在的线程，减少线程创建、消亡的开销，性能佳。 
- 可有效控制最大并发线程数，提高系统资源的使用率，同时避免过多资源竞争，避免堵塞
- 提供定时执行、定期执行、单线程、并发数控制等功能

ExecutorService 是最初的线程池接口，ThreadPoolExecutor 类是对线程池的具体实现:

| 构造参数 | 描述 |
|:--------:| -------------:|
| corePoolSize| 核心线程的数量,默认情况下,即使核心线程没有任务在执行它也存在的,我们固定一定数量的核心线程且它一直存活,这样就避免了一般情况下CPU创建和销毁线程带来的开销。 |
| maximumPoolSize| 最大线程数，当任务数量超过最大线程数时其它任务可能就会被阻塞。最大线程数 = 核心线程 + 非核心线程。非核心线程只有当核心线程不够用且线程池有空余时才会被创建，执行完任务后非核心线程会被销毁。 |
| keepAliveTime| 非核心线程的超时时长,当执行时间超过这个时间时,非核心线程就会被回收。当 allowCoreThreadTimeOut设置为true时,此属性也作用在核心线程上 |
| unit| 时间单位 |
| workQueue| 任务队列,我们提交给线程池的 runnable 会被存储在这个对象上 |
| threadFactory| 线程工厂,用于指定为线程池创建新线程的方式 |
| handler| 拒绝策略 |
<!-- more -->

### 线程池分配原则

- 当 currentSize < corePoolSize 时,直接启动一个核心线程执行任务
- 当 currentSize >= corePoolSize 并且`workQueue`未满时，添加进来的任务会被安排到`workQueue`中等待执行
- 当`workQueue`已满，但是 currentSize < maximumPoolSize 时，会立即开启一个非核心线程来执行任务
- 当 currentSize >= corePoolSize并且workQueue 已满和currentSize > maximumPoolSize 时, 线程池则拒绝执行该任务,且 ThreadPoolExecutor 会调用 RejectedtionHandler 的`rejectedExecution()`来通知调用者

### 任务队列

>  对于新的任务.要根据任务场景考虑使用什么类型的容器缓存新任务

| 实现类 | 类型 |说明 |
|:--------:| :-------------:|:-------------:|
| SynchronousQueue| 同步队列 | 该队列不存储元素，每个插入操作必须等待另一个线程调用移除操作，否则插入操作会一直阻塞 |
| ArrayBlockingQueue| 有界队列 | 基于数组的阻塞队列，按照 FIFO 原则对元素进行排序 |
| LinkedBlockingQueue| 无界队列 | 基于链表的阻塞队列，按照 FIFO 原则对元素进行排序 |
| PriorityBlockingQueue| 优先级队列 | 具有优先级的阻塞队列 |
| DelayQueue| | 类似于PriorityBlockingQueue，是二叉堆实现的无界优先级阻塞队列。要求元素都实现Delayed接口，通过执行时延从队列中提取任务，时间没到任务取不出来.|
| LinkedBlockingDeque| | 使用双向队列实现的有界双端阻塞队列。双端意味着可以像普通队列一样FIFO（先进先出），也可以像栈一样FILO（先进后出）|
| LinkedTransferQueue| | 它是ConcurrentLinkedQueue、LinkedBlockingQueue和SynchronousQueue的结合体,但是把它用在ThreadPoolExecutor中,和LinkedBlockingQueue行为一致，但是是无界的阻塞队列。|


### 线程工厂

> 线程工厂指定创建线程的方式,需要实现 ThreadFactory 接口,并实现 newThread(Runnable r) 方法

Executors 框架已经为我们实现了一个默认的线程工厂:
```java
private static class DefaultThreadFactory implements ThreadFactory {
    private static final AtomicInteger poolNumber = new AtomicInteger(1);
    private final ThreadGroup group;
    private final AtomicInteger threadNumber = new AtomicInteger(1);
    private final String namePrefix;
 
    DefaultThreadFactory() {
        SecurityManager s = System.getSecurityManager();
        group = (s != null) ? s.getThreadGroup() :
                              Thread.currentThread().getThreadGroup();
        namePrefix = "pool-" +poolNumber.getAndIncrement() +"-thread-";
    }
 
    public Thread newThread(Runnable r) {
        Thread t = new Thread(group, r,namePrefix + threadNumber.getAndIncrement(),0);
        if (t.isDaemon())
            t.setDaemon(false);
        if (t.getPriority() != Thread.NORM_PRIORITY)
            t.setPriority(Thread.NORM_PRIORITY);
        return t;
    }
}
```


### 拒绝策略

> 线程数量大于等于 maximumPoolSize 且 workQueue 已满，则使用拒绝策略处理新任务

|实现类	|说明|
|:--------:| :-------------:|
|AbortPolicy|	丢弃新任务，并抛出 RejectedExecutionException|
|DiscardPolicy	|不做任何操作，直接丢弃新任务|
|DiscardOldestPolicy|	丢弃队列队首的元素，并执行新任务|
|CallerRunsPolicy|	由调用线程执行新任务|


### 线程池


#### FixThreadPool

> FixThreadPool 只有核心线程，并且数量固定的，也不会被回收。使用 LinkedBlockingQueue 无界队列，所有线程都活动时，因为队列没有限制大小，新任务会等待执行。由于线程不会回收，FixThreadPool会更快地响应外界请求

```java
public static ExecutorService newFixThreadPool(int nThreads){
    return new ThreadPoolExecutor(nThreads, nThreads, 0L, TimeUnit.MILLISECONDS, 
                        new LinkedBlockingQueue<Runnable>());
}
//使用
Executors.newFixThreadPool(5).execute(r);
```

#### SingleThreadPool

> SingleThreadPool 只有一个核心线程，确保所有任务都在同一线程中按顺序完成。因此不需要处理线程同步的问题

```java
public static ExecutorService newSingleThreadPool (){
    return new FinalizableDelegatedExecutorService ( new ThreadPoolExecutor (1, 1, 0, TimeUnit. MILLISECONDS, 
                        new LinkedBlockingQueue<Runnable>()) );
}
//使用
Executors.newSingleThreadPool ().execute(r);
```


#### CachedThreadPool

> CachedThreadPool 只有非核心线程，最大线程数非常大，所有线程都活动时，会为新任务创建新线程，否则利用空闲线程处理任务。采用 SynchronousQueue 同步队列相当于一个空集合，导致任何任务都会被立即执行。比较适合执行大量的耗时较少的任务。

```java
public static ExecutorService newCachedThreadPool(int nThreads){
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60L, TimeUnit. SECONDS, 
                        new SynchronousQueue<Runnable>());
}
//使用
Executors.newCachedThreadPool().execute(r);
```

#### ScheduledThreadPool

> 核心线程数固定，非核心线程（闲着没活干会被立即回收）数没有限制

```java
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize){
    return new ScheduledThreadPoolExecutor(corePoolSize);
}
public ScheduledThreadPoolExecutor(int corePoolSize){
    super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS, new DelayedQueue ());
}
//使用，延迟1秒执行，每隔2秒执行一次Runnable r
Executors.newScheduledThreadPool(5).scheduleAtFixedRate(r, 1000, 2000, TimeUnit.MILLISECONDS);
```


### 关闭线程池


-   shutdown()：执行后停止接受新任务，会把队列的任务执行完毕。将线程池状态变为 SHUTDOWN。
-   shutdownNow()：也是停止接受新任务，但会中断所有的任务，将线程池状态变为 STOP