---
title: 自定义View(3) - Text相关
tags:
  - 图形绘制
categories: Android
updated: 1635947827000
date: 2021-11-03 21:57:48
---

### 文字绘制(Canvas)

#### 绘制方式

##### drawText

![IiheJA.png](https://z3.ax1x.com/2021/11/02/IiheJA.png)
<!-- more -->
```java
String text = "Hello HenCoder";
...
canvas.drawText(text, 200, 100, paint);
```
##### drawTextRun

##### drawTextOnPath
> 沿着一条 Path 来绘制文字

![IihNzq.png](https://z3.ax1x.com/2021/11/02/IihNzq.png)
`drawTextOnPath()` 使用的 `Path` ，拐弯处全用圆角，别用尖角
```java
canvas.drawPath(path, paint); // 把 Path 也绘制出来，理解起来更方便
canvas.drawTextOnPath("Hello HeCoder", path, 0, 0, paint);
```

##### StaticLayout
> 进行多行文字的绘制

- View 的边缘自动折行
- 在换行符 `\n` 处换行

![Iih7Yd.md.png](https://z3.ax1x.com/2021/11/02/Iih7Yd.md.png)

```java
String text1 = "Lorem Ipsum is simply dummy text of the printing and typesetting industry.";
StaticLayout staticLayout1 = new StaticLayout(text1, paint, 600,
        Layout.Alignment.ALIGN_NORMAL, 1, 0, true);
String text2 = "a\nbc\ndefghi\njklm\nnopqrst\nuvwx\nyz";
StaticLayout staticLayout2 = new StaticLayout(text2, paint, 600,
        Layout.Alignment.ALIGN_NORMAL, 1, 0, true);

...

canvas.save();
canvas.translate(50, 100);
staticLayout1.draw(canvas);
canvas.translate(0, 200);
staticLayout2.draw(canvas);
canvas.restore();
```

### 文字绘制辅助(Paint)

#### 样式设置

- `setTextSize(float textSize)`
- `setTypeface(Typeface typeface)`
- `setFakeBoldText(boolean fakeBoldText)`是否使用伪粗体
- `setStrikeThruText(boolean strikeThruText)`是否加删除线
- `setUnderlineText(boolean underlineText)`
- `setTextSkewX(float skewX)` 文字横切角度
- `setTextScaleX(float scaleX)`
- `setLetterSpacing(float letterSpacing)` 字符间距
- `setFontFeatureSettings(String settings)`
- `setTextAlign(Paint.Align align)`
- `setTextLocale(Locale locale)`语言区域
- `setHinting(int mode)`字体微调
- `setSubpixelText(boolean subpixelText)`是否开启像素级的抗锯齿



#### 测量

- `getFontSpacing()` 获取推荐的行距
	- ![IiboRg.png](https://z3.ax1x.com/2021/11/02/IiboRg.png)
- `getFontMetrics()`
	- ![IibtM9.png](https://z3.ax1x.com/2021/11/02/IibtM9.png)
- `getTextBounds()` 获取文字范围
- `measureText()`测量文字的宽度并返回
- `getTextWidths()`获取字符串中每个字符的宽度
- `breakText()` 和 `measureText()` 的区别是， breakText() 是在给出宽度上限的前提下测量文字的宽度。如果文字的宽度超出了上限，那么在临近超限的位置截断文字
- 光标相关
	- `getRunAdvance()`计算出某个字符处光标的 `x` 坐标
		- ![Iibnrn.png](https://z3.ax1x.com/2021/11/02/Iibnrn.png)
		- ![IibMV0.md.png](https://z3.ax1x.com/2021/11/02/IibMV0.md.png)
	- `getOffsetForAdvance` 计算第几个字符最接近这个坐标
- `hasGlyph()`检查指定的字符串中是否是一个单独的字形
	- ![IiH9tU.png](https://z3.ax1x.com/2021/11/02/IiH9tU.png)