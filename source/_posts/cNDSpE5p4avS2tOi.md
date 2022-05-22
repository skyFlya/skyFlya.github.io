---
title: 自定义View(11) - 滚动速度和滚动计算
tags:
  - 图形绘制
categories: Android
updated: 1635948547000
date: 2021-11-03 22:09:36
---

### VelocityTracker

> 跟踪手指在滑动过程中的速度，包括水平和竖直方向的速度
<!-- more -->
```java
class VelocityTrackerTestView(context: Context?, attrs: AttributeSet?) : View(context, attrs) {
    //1、创建实例
    private var mVelocityTracker = VelocityTracker.obtain()
    @SuppressLint("ClickableViewAccessibility")
    override fun onTouchEvent(event: MotionEvent): Boolean {
        //2、重置
        if (event.actionMasked == MotionEvent.ACTION_DOWN) {
            mVelocityTracker.clear()
        }
        //3、开始追踪
        mVelocityTracker.addMovement(event)
 
        when (event.actionMasked) {
            MotionEvent.ACTION_DOWN -> {
                //...
            }
            MotionEvent.ACTION_MOVE -> {
                //o...
            }
            MotionEvent.ACTION_UP -> {
                //速度 = （ 终点位置(px) - 起点位置(px) ）/ 时间段(ms)
                //4、设置时间段
                mVelocityTracker.computeCurrentVelocity(1000)
                //5、获取x方向、y方向的速度
                //其中getXVelocity、getYVelocity方法的参数是pointerId，用于多指触控。不考虑多指时，可以不用传参数
                var xVelocity = mVelocityTracker.getXVelocity(0)
                var yVelocity = mVelocityTracker.getYVelocity(0)
                //...
            }
        }
        return true
    }
 
    override fun onDetachedFromWindow() {
        //6、当不需要使用时，重置并回收内存
        mVelocityTracker.clear()
        mVelocityTracker.recycle()
        super.onDetachedFromWindow()
    }
}
```
VelocityTracker 一般用来判断当前是否达到一定的滑动速度来触发 Fling 的效果，这个滑动速度我们可以自己设置，也可以通过系统提供的来获取

```java
    private var mViewConfiguration : ViewConfiguration = ViewConfiguration.get(context)
    private var mMaxFlingVelocity = 0
    //触发fling的速度
    private var mMinFlingVelocity = 0
    
    init {
        mMaxFlingVelocity = mViewConfiguration.scaledMaximumFlingVelocity
        mMinFlingVelocity = mViewConfiguration.scaledMinimumFlingVelocity
    }
```

### Scroller

在 View 类里面，有两个和滚动相关的类 scrollTo() 和 scrollBy。这两个方法可以实现 View 内容的移动，比如说一个 TextView，如果使用 scrollTo()，那么移动的是里面的文字而不是位置，scrollBy() 也是一样的。那么为什么是移动，不是滚动呢？这是因为这两个方法都是瞬间完成，而不是带有滚动过程的滚动，所以说如果要实现效果比较好的滚动还是需要 Scroller

常用API:

|API|简介|
|:--:|:--:|
| computeScrollOffset() | 判断当前的滑动动作是否完成的 |
| getCurrX()、getCurrY() | 获取当前滑动的坐标值 |
| getFinalX()、getFinalY() | 获取最终滑动停止时的坐标 |
| isFinished() | 用来判断当前滚动是否结束 |
| startScroll(int startX, int startY, int dx, int dy) | 用来开始滚动，这个是很重要的一个触发computeScroll()的方法，调用这个方法之后，我们就可以在computeScroll里面获取滚动的信息，然后完成我们的需要。这个还有一个带有滚动持续时间的重载函数，可以根据需求自由使用。特别要注意这四个参数，startX和startY是开始的坐标位置，正数左上，负数右下，dx、dy同理，当在computeScroll()获取getCurrX()的时候，变化范围就与这里地设置有关。|


### OverScroller

> 对超出滑动边界的情况的处理