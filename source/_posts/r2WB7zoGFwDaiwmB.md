---
title: Android - 组件化
tags: []
updated: 1646702340000
categories: Android
thumbnail: 'https://s1.ax1x.com/2022/03/14/bLBAmR.md.png'
date: 2022-03-08 09:17:11
---

> 组件是可拆除的就表示组件与组件之间是不会耦合的。 组件化就是需要去拆分出组件和实现组件之间通信的过程

项目发展到一定程度，随着人员的增多，代码越来越臃肿，这时候就必须进行模块化的拆分。模块化是一种指导理念，其核心思想就是分而治之、降低耦合最终目的方便项目开发更加流畅，分工更加明确。同时业务线只需要依赖自己的模块无需引入其他业务模块从而可以加快编译速度
<!--more-->
### 组件拆分

![bWG2b8.png](https://s1.ax1x.com/2022/03/09/bWG2b8.png)


### 实现组件化

> 核心即是**路由**，**反射**，**通信协议**

组件之间没有依赖关系所以组件向外暴露的方法要进行注册，我们通过我们起的别名去寻找他，这就是路由的过 程，然后通过反射去实例化，通过通信协议与组件交互。 

组件化通信一定要有统一的协议去约束，不然即使找到了目标对象但是不知道如何使用也是无济于事的。 我们需要明确一点组件需要向外暴露什么东西呢?
- 一个是组件提供的Activity/Fragment路由
- 一个是组件提供的一些服务方法，如定位库提供定位信息获取

开源方案：[alibaba/ARouter: 💪 A framework for assisting in the renovation of Android componentization (帮助 Android App 进行组件化改造的路由框架)](https://github.com/alibaba/ARouter)


### 文章来源

[老生常谈：组件化 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/365501317)