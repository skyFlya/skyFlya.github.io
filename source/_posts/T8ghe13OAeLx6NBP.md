---
title: 安卓优化 - 内存优化
tags:
  - 性能优化
categories: Android
updated: 1635684336000
date: 2021-10-31 20:46:21
---

### 内存信息查看

[App内存优化实践：一步一步做内存分析与优化](https://www.jianshu.com/p/28b9cd87e667)

查看每个App进程可以分配到的最大内存
```undefined
adb shell getprop | grep dalvik.vm.heapsize
```
App的内存使用情况概览
```undefined
adb shell dumpsys meminfo 包名
```

### 优化工具

- CPU Profiler
- Memory Analyzer（MAT）
- LeakCannary

<!-- more -->
### 优化方向

[Android 如何优化APP内存 ](https://www.cnblogs.com/wangjie1990/p/11327112.html)

- 谨慎使用Services
	* 启动一个Service时， 系统需要始终保持运行该Service的进程,该Service占用的RAM对其他进程不共享
	* 避免使用持久性服务,如`JobScheduler`之类
- 使用经过优化的多数据容器
	* 如SparseArray，SparseBooleanArray和LongSparseArray
	* 如有必要，您可以随时切换到原始数组以获得精简的数据结构
- 使用nano protobufs进行序列化数据
- 避免内存泄漏
	* 内存泄露会导致大量的垃圾收集事件发生,从而导致系统执行其他内容(如渲染或者传输)的时间变少 
- 移除内存密集型资源，以及lib库
	* 减小APK的大小
	* 请使用不进行反射扫描的依赖注入库(Dagger2),频繁的反射需要更多的CPU和内存消耗
	* 谨慎使用外部库,外部库可能对同一个功能有不一样的实现,这可能导致预期之外的事情 