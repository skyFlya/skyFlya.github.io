---
title: Java 理论 - JVM
tags:
  - 理论学习
categories: Java
updated: 1635687171000
date: 2021-10-31 21:33:20
---

## JVM

> 即Java虚拟机,是java程序与OS之间的处理中介,屏蔽了与OS平台相关的具体信息.从而实现跨平台

JRE: java运行环境
JDK:java开发工具包

![5XeI1O.png](https://z3.ax1x.com/2021/10/29/5XeI1O.png)

组成:
- 类装载子系统（ClassLoader）
- 运行时数据区
- 执行引擎
- 内存回收
<!-- more -->
### 面向对象和面向过程


面向过程 
- 将一个问题分解为具体过程的行为实现
- 性能比面向对象高，因为类调用时需要实例化，开销比较大，比较消耗资源
- 没有面向对象易维护、易复用、易扩展

面向对象
- 将一个问题分解为通过不同对象的属性和功能来实现


## Refer
[Jvm系列-Jvm概述（一）](https://blog.csdn.net/qq_38163244/article/details/109551205)
[一张图看懂JVM（升级版） (qq.com)](https://mp.weixin.qq.com/s?__biz=MzU3NDY4NzQwNQ==&mid=2247483820&idx=1&sn=8418f0f6a618bb0f0ca0980af09a816f&chksm=fd2fd06eca5859786ab124dd204a7ec9b1ad3ed230b9b531086cc6729a277a05d3e8307b7e0d&mpshare=1&scene=1&srcid=10298k3e78U6EvGwOFzdlSka&sharer_sharetime=1635474328007&sharer_shareid=7cb3b4a44a48b28c2337c68afe383da6&exportkey=A7V%2B0C1tGaUKgyZIx1CLZhU%3D&pass_ticket=jfeKdbe8hPuxHSyoUa4LK3n22co%2B%2BeHANszsiwjQ22G6HpzZYr5LX%2BaxLyzvvutj&wx_header=0#rd)
[JVM原理摘要 - 有容乃大 - 博客园 (cnblogs.com)](https://www.cnblogs.com/mrhgw/p/10341660.html)
