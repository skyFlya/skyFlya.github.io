---
title: 自定义View(4) - Canvas相关
tags:
  - 图形绘制
categories: Android
updated: 1635947893000
date: 2021-11-03 21:58:53
---

### 范围裁切

#### clipRect
![IFKTER.png](https://z3.ax1x.com/2021/11/02/IFKTER.png)

```java
canvas.clipRect(left, top, right, bottom);
canvas.drawBitmap(bitmap, x, y, paint);
// 加上 Canvas.save() 和 Canvas.restore() 来及时恢复绘制范围
canvas.save();
canvas.clipRect(left, top, right, bottom);
canvas.drawBitmap(bitmap, x, y, paint);
canvas.restore();
```
<!-- more -->
#### clipPath
![IFM9UI.md.png](https://z3.ax1x.com/2021/11/02/IFM9UI.md.png)

```java
canvas.save();
canvas.clipPath(path1);
canvas.drawBitmap(bitmap, point1.x, point1.y, paint);
canvas.restore();

canvas.save();
canvas.clipPath(path2);
canvas.drawBitmap(bitmap, point2.x, point2.y, paint);
canvas.restore();
```

### 几何变换
#### 二维变换

> 处理常见的二维变换

- `Canvas.translate()` 平移
- `Canvas.rotate()` 旋转
- `Canvas.scale()` 缩放
- `Canvas.skew()` 错切

#### Matrix变换
> 用来处理不常见的二维变换

![IFn6qs.png](https://z3.ax1x.com/2021/11/02/IFn6qs.png)

- `Canvas.setMatrix(matrix)`
	- 用 `Matrix` 直接替换 `Canvas` 当前的变换矩阵，即抛弃 `Canvas` 当前的变换
- `Canvas.concat(matrix)`
	- 用 `Canvas` 当前的变换矩阵和 `Matrix` 相乘，即基于 `Canvas` 当前的变换，叠加上 `Matrix` 中的变换

```java
Matrix matrix = new Matrix();
float pointsSrc = {left, top, right, top, left, bottom, right, bottom};
float pointsDst = {left - 10, top + 50, right + 120, top - 90, left + 20, bottom + 30, right + 20, bottom + 60};
...
matrix.reset();
matrix.setPolyToPoly(pointsSrc, 0, pointsDst, 0, 4);
canvas.save();
canvas.concat(matrix);
canvas.drawBitmap(bitmap, x, y, paint);
canvas.restore();
```

#### Camera

> 处理三维旋转

##### Camera.rotate*

![IFZoMd.md.png](https://z3.ax1x.com/2021/11/02/IFZoMd.md.png)

```java
 Camera camera =  new  Camera();
 Point point1 = new Point(200, 100);
 Point point2 = new Point(600, 200);
 
 @Override
 protected void onDraw(Canvas canvas) {
    canvas.save();
    camera.save(); // 保存 Camera 的状态
    camera.rotateX(30); // 旋转 Camera 的三维空间
    camera.applyToCanvas(canvas); // 把旋转投影到 Canvas
    camera.restore(); // 恢复 Camera 的状态
    canvas.drawBitmap(bitmap, point1.x, point1.y, paint);
    canvas.restore();
 }
```

居中处理:
![IFesfS.png](https://z3.ax1x.com/2021/11/02/IFesfS.png)

```java
        int bitmapWidth = bitmap.getWidth();
        int bitmapHeight = bitmap.getHeight();
        int center1X = point1.x + bitmapWidth / 2;
        int center1Y = point1.y + bitmapHeight / 2;
        int center2X = point2.x + bitmapWidth / 2;
        int center2Y = point2.y + bitmapHeight / 2;

        camera.save();
        matrix.reset();
        camera.rotateX(30);
        camera.getMatrix(matrix);
        camera.restore();
        matrix.preTranslate(-center1X, -center1Y);
        matrix.postTranslate(center1X, center1Y);
        canvas.save();
        canvas.concat(matrix);
        canvas.drawBitmap(bitmap, point1.x, point1.y, paint);
        canvas.restore();
```
##### Camera.translate
##### Camera.setLocation
> 设置虚拟相机的位置