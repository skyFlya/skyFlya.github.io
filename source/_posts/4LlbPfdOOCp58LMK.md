---
title: 安卓优化 - 启动优化
tags:
  - 性能优化
categories: Android
updated: 1635684057000
date: 2021-10-31 20:43:04
---
### 启动流程

#### 相关

> 启动的流程就是通过这六个大类在这三个进程之间不断通信的过程

##### 三个进程
- Launcher进程: 整个App启动流程的起点,负责处理桌面与用户之间的交互事件,可以想象为一个桌面启动器
- SystemServer进程: Android中的所有SystemServer都由其孵化(Fork)出来,例如AMS,WindowsMannager,PackageManagerService等
- App进程: 启动的App所在的进程


<!-- more -->
##### 六个大类

- ActivityManagerService:  即AMS,负责管理系统中四大组件的启动,切换,调度以及应用进程的管理
- Instrumentation: 监控应用程序和系统的交互
- ActivityThread: 应用的入口类，通过调用main方法，开启消息循环队列。ActivityThread所在的线程被称为主线程
- ApplicationThread: 提供Binder通讯接口，AMS则通过代理调用此App进程的本地方法
- ActivityManagerProxy：AMS服务在当前进程的代理类，负责与AMS通信
- ApplicationThreadProxy：ApplicationThread在AMS服务中的代理类，负责与ApplicationThread通信

#### 顺序

[APP启动流程解析,墙裂推荐](https://blog.csdn.net/huangliniqng/article/details/89364064)
[App启动速度优化 T2](https://www.cnblogs.com/not2/p/14326090.html)
[具体代码流程](https://blog.csdn.net/huangliniqng/article/details/89364064)

1. Launcher通知AMS, 要启动某一应用,并说明对应的LauncherActivity
2. AMS表示收到, 等待Launcher进入Pause状态
3. Launcher进入Pause状态, 通知AMS可以启动某一应用了
4. AMS开始检查某一应用是否启动
	- 是,则直接启动,流程终止
	- 否,AMS则在的进程中创建ActivityThread对象,并启动main函数
5. 某一应用通知AMS启动准备就绪
6. AMS通知某一应用要启动的页面,某一应用启动对应页面


![57Z1PJ.png](https://z3.ax1x.com/2021/10/27/57Z1PJ.png)


### 启动分类

- 冷启动
	- 耗时最多,优化重点
	- ![57ZxJJ.png](https://z3.ax1x.com/2021/10/27/57ZxJJ.png)
- 热启动
	- 最快.即后台到前台的切换
- 温启动
	- 较快,只重走Activity的生命周期,即销毁后重建


### 耗时统计
#### Systrace
```java
TraceCompat.beginSection("sectionName")
TraceCompat.endSection();
```
 `python systrace.py -t 10 [other-options] [categories]`
#### Traceview

![57llge.png](https://z3.ax1x.com/2021/10/27/57llge.png)

使用方式
```java
Debug.startMethodTracing("fileName")
Debug.stopMethodTracing()
```
运行之后可以在目录下生成文件：内部存储/android/data/${application}/files/fileName.trace，此文件可以使用Android Studio Profile打开

- Wall Clock time : 是线程真正执行的时间
- Thread time : CPU执行的时间,比Wall Clock Time少,不包含锁时间,等待时间
- Top Down:就是函数的调用列表
- Call Chart: 系统Api黄色，应用调用的方法绿色，第三方Api(java sdk也属于第三方)蓝色
- Flame Chart:  主要的作用是收集调用方法的时间，比如多次调用LayoutInflate.inflate，Flame Chart会把他们都收集到一起。
- Bottom Up: 和Top Down是相反的

#### Adb 命令统计
`adb shell am start -S -W 包名/启动类的全限定名`

ThisTime : 最后一个 Activity 的启动耗时 
TotalTime : 启动一连串的 Activity 总耗时
WaitTime : 应用进程的创建过程 + TotalTime

#### 系统日志统计

过滤`displayed`输出的启动日志

![57mU4e.png](https://z3.ax1x.com/2021/10/27/57mU4e.png)

### 冷启动优化

[ App 启动时间优化详解](https://jishuin.proginn.com/p/763bfbd345f0)

优化方向:
-   延迟加载 / 懒加载
-   异步线程执行耗时操作，如图片加载、网络访问、IO操作等
-   ViewStub的使用
-   减少布局层次和嵌套布局