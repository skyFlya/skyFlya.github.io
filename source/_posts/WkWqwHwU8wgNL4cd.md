---
title: Java并发(5) - 线程协作
tags:
  - 并发
categories: Java
date: 2021-11-07 19:04:05
---

### 线程状态

![InuMpd.png](https://z3.ax1x.com/2021/11/05/InuMpd.png)

<!-- more -->
### wait、notify、notifyAll

> 实现线程间的协作


#### wait

- `wait()`将当前运行的线程挂起(即让其进入阻塞状态)，直到`notify`或`notifyAll`方法来唤醒线程
- `wait(long timeout)`跟上面类似,但会超时唤醒

```java
public class WaitTest {
    // 写法错误:这里会抛出的 IllegalMonitorStateException 异常
    // 因为在调用wait方式时没有获取到monitor对象的所有权，那如何获取monitor对象所有权？Java中只能通过Synchronized关键字来完成
    public void testWait(){
        System.out.println("Start-----");
        wait(1000);
        System.out.println("End-------");
    }
    // 写法正确
    public synchronized void testWait(){
        System.out.println("Start-----");
        wait(1000);
        System.out.println("End-------");
    }
 
    public static void main(String[] args) {
        final WaitTest test = new WaitTest();
        new Thread(() -> {
            test.testWait();
        }).start();
    }
}
```

`wait()`的使用必须在`synchronized `的范围内，否则就会抛出IllegalMonitorStateException异常，`wait()`的作用就是阻塞当前线程等待notify/notifyAll方法的唤醒，或等待超时后自动唤醒

#### notify/notifyAll

- `notify()`唤醒monitor上的一个线程
- `notifyAll()`唤醒所有的线程

### sleep、yield、join 

- `sleep()`延时
- `start()`启动线程，让线程变成就绪状态等待 CPU 调度后执行
- `yield()`让掉当前线程资源占有，使正在运行中的线程重新变成就绪状态，并重新竞争 CPU 的调度权。它可能会获取到，也有可能被其他线程获取到
- `join()`使主线程进入等待池并等待当前线程执行完毕后才会被唤醒


`yield()`可以很好的控制多线程，如执行某项复杂的任务时，如果担心占用资源过多，可以在完成某个重要的工作后使用 yield 方法让掉当前 CPU 的调度权，等下次获取到再继续执行，这样不但能完成自己的重要工作，也能给其他线程一些运行的机会，避免一个线程长时间占有 CPU 资源。