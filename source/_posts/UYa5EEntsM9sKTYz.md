---
title: 安卓优化 - 卡顿优化
tags:
  - 性能优化
categories: Android
updated: 1635684411000
date: 2021-10-31 20:47:12
---

> 卡顿产生的根本原因就是CPU和GPU没有及时处理好数据，针对卡顿的优化就有思路了：尽可能减少 CPU 和 GPU 资源的消耗

- CPU：中央处理器（CPU，central processing unit）作为计算机系统的运算和控制核心，是信息处理、程序运行的最终执行单元
- GPU：图形处理器（英语：Graphics Processing Unit，缩写：GPU），又称显示核心,做图像和图形相关运算工作的微处理器



### 卡顿检测

[Android UI性能优化 检测应用中的UI卡顿](https://blog.csdn.net/lmj623565791/article/details/58626355)

[Android性能优化-检测App卡顿 - 简书 (jianshu.com)](https://www.jianshu.com/p/9e8f88eac490)

<!-- more -->
#### 用UI线程Looper打印的日志

开源工具:
[Kyson/AndroidGodEye: An app performance monitor(APM) , like "Android Studio profiler", you can easily monitor the performance of your app real time in browser (github.com)](https://github.com/Kyson/AndroidGodEye)

[markzhai/AndroidPerformanceMonitor: A transparent ui-block detection library for Android. (known as BlockCanary) ](https://github.com/markzhai/AndroidPerformanceMonitor)

[BzCoder/BlockCanaryCompat: 卡顿监控，BlockCanary 适配Android O 以上系统 (github.com)](https://github.com/BzCoder/BlockCanaryCompat)
#### Choreographer

Android系统每隔16ms发出VSYNC信号，触发对UI进行渲染。开发者可以使用Choreographer#postFrameCallback设置自己的callback与Choreographer交互，你设置的FrameCallCack（doFrame方法）会在下一个frame被渲染时触发。理论上来说两次回调的时间周期应该在16ms，如果超过了16ms我们则认为发生了卡顿，我们主要就是利用两次回调间的时间周期来判断

开源工具:
[wasabeef/Takt: Takt is Android library for measuring the FPS using Choreographer](https://github.com/wasabeef/Takt)

[friendlyrobotnyc/TinyDancer: An android library for displaying fps from the choreographer and percentage of time with two or more frames dropped ](https://github.com/friendlyrobotnyc/TinyDancer)


### ANR分析

> Application Not Responding，也就是应用程序无响应


#### 产生原因

-   InputDispatching Timeout：5秒内无法响应屏幕触摸事件或键盘输入事件
-   BroadcastQueue Timeout ：在执行前台广播（BroadcastReceiver）的`onReceive()`函数时10秒没有处理完成，后台为60秒
-   Service Timeout：前台服务20秒内，后台服务在200秒内没有执行完毕
-   ContentProvider Timeout：ContentProvider的publish在10s内没进行完
-   其他
	+  主线程阻塞或主线程数据读取 
	+  CPU满负荷，I/O阻塞
	+  内存不足


分析:
- log上的anr reason
- adb 导出ANR日志
	+ `adb pull /data/anr/traces.txt`