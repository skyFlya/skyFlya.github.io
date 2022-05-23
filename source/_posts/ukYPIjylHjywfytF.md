---
title: Android黑科技 - 热修复和插件化基础
tags:
  - 黑科技
updated: 1646662373000
categories: Android
thumbnail: 'https://s1.ax1x.com/2022/03/07/b6aO6P.md.png'
date: 2022-03-07 22:31:41
---

### Dalvik和 ART

DVM也是实现了JVM规范的一个虚拟器，默认使用CMS垃圾回收器，但是与JVM运行 Class 字节码不同，DVM 执行 Dex(Dalvik Executable Format) ——专为 Dalvik 设计的一种压缩格式。Dex 文件是很多 .class 文件处理压缩后的产物，最终可以在 Android 运行时环境执行。

<!--more-->
ART（Android Runtime） 是在 Android 4.4 中引入的一个开发者选项，也是 Android 5.0 及更高版本的默认 Android 运行时。

ART 和 Dalvik 都是运行 Dex 字节码的兼容运行时，因此针对 Dalvik 开发的应用也能在 ART 环境中运作。

Dalvik和 ART编译区别
![bydjWn.png](https://s1.ax1x.com/2022/03/07/bydjWn.png)

[source.android.google.cn/devices/tec…](https://source.android.google.cn/devices/tech/dalvik/gc-debug)    

dexopt与dexaot区别:
- dexopt
  在Dalvik中虚拟机在加载一个dex文件时，对 dex 文件 进行 验证 和 优化的操作，其对 dex 文件的优化结果变成了 odex(Optimized dex) 文件，这个文件和 dex 文件很像，只是使用了一些优化操作码。
- dex2oat
   ART 预先编译机制，在安装时对 dex 文件执行dexopt优化之后再将odex进行 AOT 提前编译操作，编译为OAT（实际上是ELF文件）可执行文件（机器码）。（相比做过ODEX优化，未做过优化的DEX转换成OAT要花费更长的时间）


### 类加载
> 任何一个 Java 程序都是由一个或多个 class 文件组成，在程序运行时，需要将 class 文件加载到 JVM 中才可以使用，负责加载这些 class 文件的就是 Java 的类加载机制。

Android中常用的有两种类加载器，DexClassLoader和PathClassLoader，它们都继承于BaseDexClassLoader。相关源码如下：

![](https://pic1.zhimg.com/80/v2-43a7045200722c1aea24966765f82570_720w.jpg)

区别在于调用父类构造器时，DexClassLoader多传了一个optimizedDirectory参数，这个目录必须是内部存储路径，用来缓存系统创建的Dex文件。而PathClassLoader该参数为null，只能加载内部存储目录的Dex文件。所以我们可以用DexClassLoader去加载外部的apk，用法如下：

![](https://pic3.zhimg.com/80/v2-280d8e49c4b6aab388bc1f8b906f13f2_720w.jpg)

#### Refer
[Android 热修复核心原理，ClassLoader类加载 ](https://juejin.cn/post/7018534681620512805)
[Android ClassLoader类加载机制](https://blog.csdn.net/qq_35090026/article/details/122909541#:~:text=Android%E4%B8%AD%E7%9A%84ClassLoader%E7%B1%BB%E5%9E%8B%E5%88%86%E5%88%AB%E6%98%AF%E7%B3%BB%E7%BB%9F%E7%B1%BB%E5%8A%A0%E8%BD%BD%E5%99%A8%E5%92%8C%E8%87%AA%E5%AE%9A%E4%B9%89%E5%8A%A0%E8%BD%BD%E5%99%A8%E3%80%82%20%E5%85%B6%E4%B8%AD%E7%B3%BB%E7%BB%9F%E7%B1%BB%E5%8A%A0%E8%BD%BD%E5%99%A8%E4%B8%BB%E8%A6%81%E5%8C%85%E6%8B%AC3%E7%A7%8D%EF%BC%8C%E5%88%86%E5%88%AB%E6%98%AF%20BootClassLoader%20%E3%80%81,PathClassLoader%20%E5%92%8C%20DexClassLoader%20%E3%80%82)
[Android ClassLoader机制与热修复](https://blog.csdn.net/qq_32506429/article/details/122783941)
[Android ClassLoader详解](https://blog.csdn.net/xiangzhihong8/article/details/52880327)

### 双亲委托机制

ClassLoader调用loadClass方法加载类，代码如下：

![](https://pic2.zhimg.com/80/v2-20dca70a7fca2413de158563cb7d27d1_720w.jpg)

可以看出ClassLoader加载类时，先查看自身是否已经加载过该类，如果没有加载过会首先让父加载器去加载，如果父加载器无法加载该类时才会调用自身的findClass方法加载，该机制很大程度上避免了类的重复加载。

### DexClassLoader的DexPathList

> DexPathList是在构造DexClassLoader时生成的，其内部包含了DexFile

![by01BT.png](https://s1.ax1x.com/2022/03/07/by01BT.png)

DexPathList的loadClass会去遍历DexFile直到找到需要加载的类。

![](https://pic3.zhimg.com/80/v2-ddf43519948c3b9ab3f15ff19cf77dfe_720w.jpg)

腾讯的qq空间热修复技术正是利用了DexClassLoader的加载机制，将需要替换的类添加到dexElements的前面，这样系统会使用先找到的修复过的类。

### 资源加载

Android系统通过Resource对象加载资源，下面代码展示了该对象的生成过程。

![](https://pic1.zhimg.com/80/v2-ab6ea5aad79904422d2fa7d8bc87a64c_720w.jpg)

因此，只要将插件apk的路径加入到AssetManager中，便能够实现对插件资源的访问。

具体实现时，由于AssetManager并不是一个public的类，需要通过反射去创建，并且部分Rom对创建的Resource类进行了修改，所以需要考虑不同Rom的兼容性。


### Refer

[深入理解Android插件化技术 ](https://zhuanlan.zhihu.com/p/33017826)