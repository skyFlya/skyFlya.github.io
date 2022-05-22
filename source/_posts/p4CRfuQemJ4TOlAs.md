---
title: 自定义View(8) - draw 绘制过程
tags:
  - 图形绘制
categories: Android
updated: 1635948186000
date: 2021-11-03 22:03:38
---

> Android 里面的绘制都是按顺序的，先绘制的内容会被后绘制的盖住

![IES7KP.png](https://z3.ax1x.com/2021/11/03/IES7KP.png)
<!-- more -->

### draw 过程解析

 一个完整的绘制过程会依次绘制以下几个内容：
 ![IEp3Ie.png](https://z3.ax1x.com/2021/11/03/IEp3Ie.png)

- `drawBackground()`绘制背景
	- 这个方法是 private 的，不能重写，你如果要设置背景，只能用自带的 API 去设置
- `onDraw()`绘制主体
	- 这个方法在 View 和 ViewGroup 里都是空实现，因此自定义时需要复写
- `dispatchDraw()`绘制子 View
	- 在于单一 View 中无子 View，故在 View 中此方法默认为空实
	- 在 ViewGroup中系统已经复写好此方法对其子视图进行绘制因此我们不需要复写
- `onDrawForeground()`滑动边缘渐变和滑动条以及前景

ViewGroup中的`dispatchDraw()`:
```java
    protected void dispatchDraw(Canvas canvas) {
        boolean usingRenderNodeProperties = canvas.isRecordingFor(mRenderNode);
        final int childrenCount = mChildrenCount;
        final View[] children = mChildren;
        int flags = mGroupFlags;
        // 动画处理
        if ((flags & FLAG_RUN_ANIMATION) != 0 && canAnimate()) {
            final boolean buildCache = !isHardwareAccelerated();
            for (int i = 0; i < childrenCount; i++) {
                final View child = children[i];
                if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE) {
                    final LayoutParams params = child.getLayoutParams();
                    attachLayoutAnimationParameters(child, params, i, childrenCount);
                    bindLayoutAnimation(child);
                }
            }
            // xxx
            controller.start();
        }
        // 间距处理
        int clipSaveCount = 0;
        final boolean clipToPadding = (flags & CLIP_TO_PADDING_MASK) == CLIP_TO_PADDING_MASK;
        if (clipToPadding) {
            clipSaveCount = canvas.save(Canvas.CLIP_SAVE_FLAG);
            canvas.clipRect(mScrollX + mPaddingLeft, mScrollY + mPaddingTop,
                    mScrollX + mRight - mLeft - mPaddingRight,
                    mScrollY + mBottom - mTop - mPaddingBottom);
        }

        // 遍历子View
        for (int i = 0; i < childrenCount; i++) {
                if ((transientChild.mViewFlags & VISIBILITY_MASK) == VISIBLE ||transientChild.getAnimation() != null) {
                    // 2. 绘制子View视图    
                    more |= drawChild(canvas, transientChild, drawingTime);
                }
        }
    }

```

### draw顺序

####  onDraw() 

> 自定义绘制最基本的形态：继承 View 类，在 onDraw() 中完全自定义它的绘制

定义 View 时，绘制代码写在 super.onDraw() 的上面还是下面都无所谓,但基于已有控件的自定义绘制，就不能不考虑 `super.onDraw()` 了，你需要根据自己的需求，判断出你绘制的内容需要盖住控件原有的内容还是需要被控件原有的内容盖住，从而确定你的绘制代码是应该写在 super.onDraw() 的上面还是下面。

|上面| 下面 |
|:--:|:--:|
| ![IEFrgH.png](https://z3.ax1x.com/2021/11/03/IEFrgH.png)  | ![IEFHrn.png](https://z3.ax1x.com/2021/11/03/IEFHrn.png) |

#### dispatchDraw()

>  如果想在一个ViewGroup中按上面的做法在下面添加绘制内容则是不行的,因为在绘制过程中每一个 ViewGroup 会先调用自己的 onDraw() 来绘制完自己的主体之后再去绘制它的子 View,会覆盖其主体


![IEEpH1.png](https://z3.ax1x.com/2021/11/03/IEEpH1.png)

```java
public class SpottedLinearLayout extends LinearLayout {
    ...
    // 把 onDraw() 换成了 dispatchDraw()
    protected void dispatchDraw(Canvas canvas) {
       super.dispatchDraw(canvas);
       ... // 绘制斑点
    }
}
```

虽然 View 和 ViewGroup 都有 dispatchDraw() 方法，不过由于 View 是没有子 View 的，所以一般来说 dispatchDraw() 这个方法只对 ViewGroup（以及它的子类）有意义。

#### onDrawForeground()

> 前景前后处理

在 super.onDrawForeground() 的上面
```java
public class AppImageView extends ImageView {
    ...
    public void onDrawForeground(Canvas canvas) {
       ... // 绘制「New」标签
       super.onDrawForeground(canvas);
    }
}
```
![IEVgSS.png](https://z3.ax1x.com/2021/11/03/IEVgSS.png)

在 super.onDrawForeground() 的下面:

```java
public class AppImageView extends ImageView {
    ...
    public void onDrawForeground(Canvas canvas) {
       super.onDrawForeground(canvas);
       ... // 绘制「New」标签
    }
}
```
![IEZV0A.png](https://z3.ax1x.com/2021/11/03/IEZV0A.png)

#### draw()

### 总结

![IEZ1XQ.png](https://z3.ax1x.com/2021/11/03/IEZ1XQ.png)