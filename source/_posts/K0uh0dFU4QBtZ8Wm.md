---
title: 自定义View(1) - 基本绘制
tags:
  - 图形绘制
categories: Android
updated: 1635776226000
date: 2021-11-01 22:17:37
---

> 简单的绘制基本由`Canvas.drawxxx()`和Paint的配置组成

![IC9uZj.png](https://z3.ax1x.com/2021/11/01/IC9uZj.png)

```kotlin
class PaintView @JvmOverloads constructor(context: Context, attrs: AttributeSet? = null, defStyleAttr: Int = 0) : View(context, attrs, defStyleAttr) {
  private val mPaint by lazy { Paint() }
  override fun onDraw(canvas: Canvas) {  
    super.onDraw(canvas)
    paint.setColor(Color.RED)// 设置为红色
    canvas.drawCircle(300, 300, 200, paint)
  }
}
```
<!-- more -->

### Canvas

- `Canvas.drawArc()` 绘制弧形或扇形
		+ `left`, `top`, `right`, `bottom` 描述的是这个弧形所在的椭圆
		+ `startAngle` 是弧形的起始角度
		+ `sweepAngle` 是弧形划过的角度
		+ `useCenter` 表示是否连接到圆心，如果不连接到圆心，就是弧形，如果连接到圆心，就是扇形
- `Canvas.drawPath()` 组合图形

### Paint

+ `Paint.setStyle(Style style)`  设置绘制模式
+ `Paint.setColor(int color)`  设置颜色
+ `Paint.setStrokeWidth(float width)`  设置线条宽度
+ `Paint.setTextSize(float textSize)`  设置文字大小
+ `Paint.setAntiAlias(boolean aa)`  设置抗锯齿开关

### Path

> 如果只画一个圆，没必要用 Path，直接用 drawCircle() 就行了。drawPath() 一般是在绘制组合图形时才会用到的
 
#### Path.addXxx
> 添加子图形


`path.addCircle(300, 300, 200, Path.Direction.CW)`


#### Path.xxxTo
> 用于画线（直线或曲线）

`lineTo(float x, float y) / rLineTo(float x, float y) `画直线:
![ICkWGR.png](https://z3.ax1x.com/2021/11/01/ICkWGR.png)
```java
paint.setStyle(Style.STROKE);
path.lineTo(100, 100); // 由当前位置 (0, 0) 向 (100, 100) 画一条直线
path.rLineTo(100, 0); // 由当前位置 (100, 100) 向正右方 100 像素的位置画一条直线
```

![ICASL8.png](https://z3.ax1x.com/2021/11/01/ICASL8.png)]
```java
paint.setStyle(Style.STROKE);
path.lineTo(100, 100); // 画斜线
path.moveTo(200, 100); // 我移~~
path.lineTo(200, 0); // 画竖线
```
`arcTo()` 和 `addArc()`画弧线:

| true | false|
|:--------:| :-------------:|
| ![ICEuct.png](https://z3.ax1x.com/2021/11/01/ICEuct.png) | ![ICEinK.png](https://z3.ax1x.com/2021/11/01/ICEinK.png)   |

```java
paint.setStyle(Style.STROKE);
path.lineTo(100, 100);
// true 直接连线连到弧形起点（有痕迹）
// false 强制移动到弧形起点（无痕迹）
path.arcTo(100, 100, 300, 300, -90, 90, true/false); 
// 等同上面
paint.setStyle(Style.STROKE);
path.lineTo(100, 100);
path.addArc(100, 100, 300, 300, -90, 90); // addArc() 只是一个直接使用了 forceMoveTo = true 的简化版 arcTo()
```

`close() `封闭当前子图形:


| no close | close |
|:--------:| :-------------:|
| ![ICVaPH.png](https://z3.ax1x.com/2021/11/01/ICVaPH.png) | ![ICVgIg.png](https://z3.ax1x.com/2021/11/01/ICVgIg.png)|

```java
paint.setStyle(Style.STROKE);
path.moveTo(100, 100);
path.lineTo(200, 100);
path.lineTo(150, 150);
path.close(); 
```

#### Path.setFillType

> 设置填充方式

![ICFucV.png](https://z3.ax1x.com/2021/11/01/ICFucV.png)

| EVEN_ODD:even-odd rule （奇偶原则） | WINDING: non-zero winding rule （非零环绕数原则） |
|:--------:| :-------------:|
| ![ICFKXT.png](https://z3.ax1x.com/2021/11/01/ICFKXT.png) | ![ICFJhR.png](https://z3.ax1x.com/2021/11/01/ICFJhR.png) |