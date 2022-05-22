---
title: Android AOP - AspectJ
tags:
  - AOP
categories: Android
updated: 1646966700000
thumbnail: 'https://s1.ax1x.com/2022/03/14/bLyk8K.md.png'
date: 2022-03-11 09:40:38
---

> AspectJ通过注解的形式来标注切入点、切入对象等，然后在代码编译期间将代码织入到java的字节码中
<!--more-->
Android开源方案：[AspectJX](https://github.com/HujiangTechnology/gradle_plugin_android_aspectjx)

### 引入依赖

gradle引入：
```groovy
buildscript {
    dependencies {
        classpath 'com.hujiang.aspectjx:gradle-android-plugin-aspectjx:2.0.4'
    }
}
```

应用plugin：
```groovy
apply plugin: 'android-aspectjx'
```

AspectJX配置：

AspectJX默认会处理所有的二进制代码文件和库，为了提升编译效率及规避部分第三方库出现的编译兼容性问题，AspectJX提供`include`,`exclude`命令来过滤需要处理的文件及排除某些文件(包括class文件及jar文件)。

```groovy
aspectjx {
    //排除所有package路径中包含`android.support`的class文件及库（jar文件）
	exclude 'android.support'
}
```

开关配置：
```groovy
aspectjx {
    //关闭AspectJX功能
	enabled false
}
```

### 简单使用

1. 准备切入的类
```java
public class Animal {
    public void fly() {
        Log.e(TAG, "animal fly method:" + this.toString() + "#fly");
    }
}
```

2. 编写对应的切面类，用@Aspect注解
```java
@Aspect  //①
public class MethodAspect {
    // 标注切入的方法，看到没有这里用的Poincut的 call
    @Pointcut("call(* com.wandering.sample.aspectj.Animal.fly(..))")//②
    public void callMethod() {
    }

    // 标注函数执行前处理
    @Before("callMethod()")//③
    public void beforeMethodCall(JoinPoint joinPoint) {
        Log.e(TAG, "before->" + joinPoint.getTarget().toString()); //④
    }
}
```
3. 编译后就可以看到类的方法在执行前有代码插入了
![bhAMh8.png](https://s1.ax1x.com/2022/03/10/bhAMh8.png)


### 语法

|JPoint  | 说明 | Pointcut语法 |
| :------|:----| :------------ |
| method call | 函数调用  | call(MethodSignature)|
| method execution | 函数内部执行  | execution(MethodSignature）|
| method execution | 构造函数调用 | call(ConstructorSignature）|
| constructor execution | 构造函数内部执行 | execution(ConstructorSignature）|
| field get | 读变量 | get(FieldSignature）|
| field set | 写变量 | set(FieldSignature）|
| static initialization | 静态代码块初始化 | staticinitialization(TypeSignature）|
| handler | 异常处理 | handler(TypeSignature),只能与@Before()配置使用|


<br/>

|Advice  | 说明 | 
| :------|:----| 
| @Before(Pointcut) | 执行Jpoint之前 |
| @After(Pointcut)  | 执行Jpoint之后 |
| @Around(Pointcut) | 替换原理的代码 |

完整语法看这[AspectJ](https://github.com/hyvenzhu/Android-Demos/blob/master/AspectJDemo/AspectJ.pdf)

### 注意点


标注`call`和`execution`执行的时候，This和Target不同的

```java
@Aspect 
public class MethodAspect {
    @Pointcut("call(* com.wandering.sample.aspectj.Animal.fly(..))")
    public void callMethod() {
        Log.e(TAG, "callMethod->");
    }

    @Before("callMethod()")
    public void beforeMethodCall(JoinPoint joinPoint) {
        Log.e(TAG, "getTarget->" + joinPoint.getTarget());
        Log.e(TAG, "getThis->" + joinPoint.getThis());
    }
}

```

切入方：
```tsx
protected void onCreate(@Nullable Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    Animal animal = new Animal();
    animal.fly();
}
```

运行结果如下：

```groovy
getTarget->com.wandering.sample.aspectj.Animal@509ddfd
getThis->com.wandering.sample.aspectj.MainActivity@98c38bf
```

> 说明target指代的是切入点方法的所有者，而this指代的是被织入代码所属类的实例对象。

将切点的`call`改为`execution`：
```groovy
getTarget->com.wandering.sample.aspectj.Animal@509ddfd
getThis->com.wandering.sample.aspectj.Animal@509ddfd
```

###  缺点
-   如果相应的class没有实现相应的对点方法将无法织入，如Activity没有实现onResume方法的话，将无法织入代码。
-   无法处理Lambda语法
-   会有一系列兼容性问题，如R8、gradle版本不同等
-   性能较差，APP项目比较大时编译时间明显加长


### Refer

[Android-Demos/AspectJ.pdf](https://github.com/hyvenzhu/Android-Demos/blob/master/AspectJDemo/AspectJ.pdf)
[Android AOP方案(一)——AspectJ](https://juejin.cn/post/6888548726424469511)
[Android AOP - 简书](https://www.jianshu.com/p/28aa352af7fb)
[Android AspectJ](https://www.jianshu.com/p/d07c996ea13c)