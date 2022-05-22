---
title: 自定义View(5) - 动画
tags:
  - 图形绘制
categories: Android
date: 2021-11-03 21:59:44
---

> 三者性能是一样的，因为 ViewPropertyAnimator 和 ObjectAnimator 的内部实现其实都是 ValueAnimator，ObjectAnimator 更是本来就是 ValueAnimator 的子类，差别只是使用的便捷性以及功能的灵活性。

![IFf4h9.gif](https://z3.ax1x.com/2021/11/02/IFf4h9.gif)
<!-- more -->
### ViewPropertyAnimator

```java
view.animate()
        .scaleX(1)
        .scaleY(1)
        .alpha(1);
```
### ObjectAnimator

> 多个动画配合执行

```java
PropertyValuesHolder holder1 = PropertyValuesHolder.ofFloat("scaleX", 1);
PropertyValuesHolder holder2 = PropertyValuesHolder.ofFloat("scaleY", 1);
PropertyValuesHolder holder3 = PropertyValuesHolder.ofFloat("alpha", 1);

ObjectAnimator animator = ObjectAnimator.ofPropertyValuesHolder(view, holder1, holder2, holder3)
animator.start();
```
### ValueAnimator 

> ValueAnimator 本身不作用于任何一个属性，也不提供任何一种动画。它就是一个数值发生器，可以产生想要的各种数值

ValueAnimator 并不常用，因为它的功能太基础了。ValueAnimator 是 ObjectAnimator 的父类，实际上，ValueAnimator 就是一个不能指定目标对象版本的 ObjectAnimato