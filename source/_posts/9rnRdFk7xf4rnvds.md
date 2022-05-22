---
title: Android AOP - 概论
tags:
  - AOP
updated: 1646888940000
categories: Android
thumbnail: 'https://s1.ax1x.com/2022/03/14/bLD5RS.md.png'
date: 2022-03-10 09:26:27
---

### 编程架构思想

-   面向对象(Object Oriented Programming)
    
-   面向过程(Procedure Oriented Programming)
    
-   面向切面(Aspect Oriented Programming)
<!--more-->
AOP 意为面向切面编程，通过预编译方式和运行期动态代理实现程序功能统一维护的一种技术，和 OOP 以对象为核心不同，AOP是针对业务处理过程中的类似的代码逻辑进行切入，然后统一处理。

AOP其实是OOP的补充，OOP从横向上区分出一个个的类来，而AOP则从纵向上向对象中加入特定的代码。

![bfRiDK.png](https://s1.ax1x.com/2022/03/10/bfRiDK.png)

### 常见概念

- 连接点(*Joinpoint*) 
	- 程序中可能作为代码注入目标的特定的点，例如一个方法调用或者方法入口。
- 切点（*Pointcut*） 
	- 告诉代码注入工具，在任何注入一段特定代码的表达式。例如，在哪些joint points应用一个特定的Advice。
	- 切入点可以选择唯一一个，比如执行某一个方法，也可以有多个选择，比如，标记了一个定义成@DebugLog的自定义注解的所有方法。
- 增强（*Advice*）  
	- 注入到class文件中的代码。
	- 典型的Advice类型有before、after和around，分别表示在目标方法执行之前、执行后和完全代替目标方法执行的代码。
- 目标对象 (*Target*) 
- 引介(*Introduction*) 
- 织入（*Weaving*） 
	-  注入代码（advices）到目标位置（joint points）的过程
- 代理（*Proxy*）  
- 切面（*Aspect*）
	- Pointcut和Advice的组合看做切面。例如，我们在应用中通过定义一个Pointcut和给定恰当的Advice，添加一个日志切面

### 分类

> 一类是，例如Java的动态代理；另一类可以归结为，例如经常听说的，AspecJ等框架

- 运行期的AOP
	- Java的动态代理
- 编译期的AOP
	- ASM
	- AspecJ

### 应用

- Hot Fix
	- 基于AOP技术，编译期修改Java字节码，对每一个方法进行插桩操作，以便于hook每一个方法，做到方法级别的热修复
- 监测方法耗时
- 日志记录
- 统一处理点击抖动
- 其他的监控方面

### AOP与APT区别

APT(Annotation Processing Tool)即注解处理器，是一种处理注解的工具，确切的说它是javac的一个工具，它用来在编译时扫描和处理注解。注解处理器以Java代码(或者编译过的字节码)作为输入，生成`.java文件`作为输出。

- APT是在编译期，通过注解生成`java`文件，然后.java文件仍然需要进一步编译生成`.class`文件
- AOP是在编译完成后直接通过修改`.class`文件，添加或者修改代码逻辑

### Refer

[AOP技术在客户端的应用与实践](https://mp.weixin.qq.com/s?__biz=MzUxODg0MzU2OQ==&mid=2247483887&idx=1&sn=d54e3f210a4f31f477dba06c3dcd352e&scene=21#wechat_redirect)