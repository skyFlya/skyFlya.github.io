---
title: 自定义View(2) - Paint相关
tags:
  - 图形绘制
categories: Android
updated: 1635776315000
date: 2021-11-01 22:19:10
---
### Color

#### 基本颜色

setColor(int color):
```java
paint.setColor(Color.parseColor("#009688"));
canvas.drawRect(30, 30, 230, 180, paint);
```
<!-- more -->
 setARGB(int a, int r, int g, int b)
```java
paint.setARGB(100, 0, 0, 0);
canvas.drawLine(0, 0, 200, 200, paint);
```

#### Shader

##### LinearGradient(线性渐变)
![ICK7e1.png](https://z3.ax1x.com/2021/11/01/ICK7e1.png)
```java
Shader shader = new LinearGradient(100, 100, 500, 500, Color.parseColor("#E91E63"),
        Color.parseColor("#2196F3"), Shader.TileMode.CLAMP);
paint.setShader(shader);
...
canvas.drawCircle(300, 300, 200, paint);
```

| CLAMP | MIRROR | REPEAT|
|:--------:| :-------------:| :-------------:|
| ![ICuIVP.png](https://z3.ax1x.com/2021/11/01/ICuIVP.png)| ![ICKWJU.png](https://z3.ax1x.com/2021/11/01/ICKWJU.png)| ![ICK5QJ.png](https://z3.ax1x.com/2021/11/01/ICK5QJ.png)|

##### RadialGradient(辐射渐变)
![IClQRU.png](https://z3.ax1x.com/2021/11/01/IClQRU.png)

```java
Shader shader = new RadialGradient(300, 300, 200, Color.parseColor("#E91E63"),
        Color.parseColor("#2196F3"), Shader.TileMode.CLAMP);
paint.setShader(shader);
...
canvas.drawCircle(300, 300, 200, paint);  
```

| CLAMP | MIRROR | REPEAT|
|:--------:| :-------------:| :-------------:|
| ![IClfW8.png](https://z3.ax1x.com/2021/11/01/IClfW8.png)| ![IC1SOJ.png](https://z3.ax1x.com/2021/11/01/IC1SOJ.png)| ![IC1SOJ.png](https://z3.ax1x.com/2021/11/01/IC1SOJ.png)|

##### SweepGradient(扫描渐变)

![IC3N8K.png](https://z3.ax1x.com/2021/11/01/IC3N8K.png)]

```java
Shader shader = new SweepGradient(300, 300, Color.parseColor("#E91E63"),
        Color.parseColor("#2196F3"));
paint.setShader(shader);
...
canvas.drawCircle(300, 300, 200, paint);
```

##### BitmapShader

![IC8dJ0.png](https://z3.ax1x.com/2021/11/01/IC8dJ0.png)

```java
Bitmap bitmap = BitmapFactory.decodeResource(getResources(), R.drawable.batman);
Shader shader = new BitmapShader(bitmap, Shader.TileMode.CLAMP, Shader.TileMode.CLAMP);
paint.setShader(shader);
...
canvas.drawCircle(300, 300, 200, paint);
```

| CLAMP | MIRROR | REPEAT|
|:--------:| :-------------:| :-------------:|
| ![ICGhAs.png](https://z3.ax1x.com/2021/11/01/ICGhAs.png)| ![ICG7cT.png](https://z3.ax1x.com/2021/11/01/ICG7cT.png)| ![ICGLB4.png](https://z3.ax1x.com/2021/11/01/ICGLB4.png)|


##### ComposeShader 混合着色器

> 所谓混合，就是把两个 Shader 一起使用。


![ICUmdg.png](https://z3.ax1x.com/2021/11/01/ICUmdg.png)

```java
// 第一个 Shader：头像的 Bitmap
Bitmap bitmap1 = BitmapFactory.decodeResource(getResources(), R.drawable.batman);
Shader shader1 = new BitmapShader(bitmap1, Shader.TileMode.CLAMP, Shader.TileMode.CLAMP);
// 第二个 Shader：从上到下的线性渐变（由透明到黑色）
Bitmap bitmap2 = BitmapFactory.decodeResource(getResources(), R.drawable.batman_logo);
Shader shader2 = new BitmapShader(bitmap2, Shader.TileMode.CLAMP, Shader.TileMode.CLAMP);
// ComposeShader：结合两个 Shader
Shader shader = new ComposeShader(shader1, shader2, PorterDuff.Mode.SRC_OVER);
paint.setShader(shader);
...

canvas.drawCircle(300, 300, 300, paint);
```


#### PorterDuff




### PathEffect

#### Style

`Paint.setStrokeWidth`:线条宽度
![ICvqQe.png](https://z3.ax1x.com/2021/11/01/ICvqQe.png)
```java
paint.setStyle(Paint.Style.STROKE);
paint.setStrokeWidth(1);
canvas.drawCircle(150, 125, 100, paint);
paint.setStrokeWidth(5);
canvas.drawCircle(400, 125, 100, paint);
paint.setStrokeWidth(40);
canvas.drawCircle(650, 125, 100, paint);
```

`Paint.setStrokeCap`:线条形状

![ICxkLj.png](https://z3.ax1x.com/2021/11/01/ICxkLj.png)

`Paint.setStrokeJoin`:拐角形状

![ICxQlF.png](https://z3.ax1x.com/2021/11/01/ICxQlF.png)

`Paint.setStrokeMiter`:拐角边缘

![image](https://z3.ax1x.com/2021/11/01/ICzenH.png)

#### Filter

`PathEffect.setDither()`:图像抖动
![ICL31H.png](https://z3.ax1x.com/2021/11/01/ICL31H.png)

`PathEffect.setFilterBitmap()`:双线性过滤
![ICvZ8O.png](https://z3.ax1x.com/2021/11/01/ICvZ8O.png)

#### Effect

| Name| Effect  |
|:--------:| :-------------:|
| CornerPathEffect| ![ICgRkq.png](https://z3.ax1x.com/2021/11/01/ICgRkq.png) |
| DiscretePathEffect| ![ICgGOH.png](https://z3.ax1x.com/2021/11/01/ICgGOH.png) |
| DashPathEffect| ![ICcXQg.png](https://z3.ax1x.com/2021/11/01/ICcXQg.png) |
| PathDashPathEffect| ![IC6T8U.png](https://z3.ax1x.com/2021/11/01/IC6T8U.png)|
| SumPathEffect| ![ICyBpF.png](https://z3.ax1x.com/2021/11/01/ICyBpF.png)|
| ComposePathEffect| ![ICy8yj.png](https://z3.ax1x.com/2021/11/01/ICy8yj.png)|



#### ShadowLayer
> 绘制层下方的阴影效果

![ICstPO.png](https://z3.ax1x.com/2021/11/01/ICstPO.png)

```java
paint.setShadowLayer(10, 0, 0, Color.RED);
canvas.drawText(text, 80, 300, paint);
```

#### MaskFilter
> 绘制层上方的附加效果

BlurMaskFilter(模糊效果):

![ICrhE6.md.png](https://z3.ax1x.com/2021/11/01/ICrhE6.md.png)

```java
paint.setMaskFilter(new BlurMaskFilter(50, BlurMaskFilter.Blur.NORMAL));
canvas.drawBitmap(bitmap, 100, 100, paint);
```

EmbossMaskFilter(浮雕效果):
![ICr0EV.png](https://z3.ax1x.com/2021/11/01/ICr0EV.png)
```java
// 数组:光源方向,光强度,炫光系数,应用光线范围 
paint.setMaskFilter(new EmbossMaskFilter(new float[]{0, 1, 1}, 0.2f, 8, 10));
canvas.drawBitmap(bitmap, 100, 100, paint);
```

#### getPath

`PathEffect.getFillPath()`获取图形Path

![ICDOtU.md.png](https://z3.ax1x.com/2021/11/01/ICDOtU.md.png)

`PathEffect.getTextPath()`获取图形Path

![ICDAQs.png](https://z3.ax1x.com/2021/11/01/ICDAQs.png)

### Refer

[HenCoder Android 开发进阶: 自定义 View 1-2 Paint 详解 (rengwuxian.com)](https://rengwuxian.com/ui-1-2/)