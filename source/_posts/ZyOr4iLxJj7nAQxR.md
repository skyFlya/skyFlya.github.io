---
title: 自定义View(7) - layout 布局过程
tags:
  - 图形绘制
categories: Android
updated: 1635948086000
date: 2021-11-03 22:02:46
---

![viewasds](https://s2.ax1x.com/2019/12/06/QJQEcR.jpg)
<!-- more -->
### layout 类型

![IA7tv4.png](https://z3.ax1x.com/2021/11/03/IA7tv4.png)


### View 的 layout 过程

```java
    // 指定视图及其所有子视图的大小和位置 这是布局机制的第二阶段
    public void layout(int l, int t, int r, int b) {
        // 判断是否measure,没有的话再measure一遍
        if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
            onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);
            mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
        }
        // 当前视图的四个顶点
        int oldL = mLeft;
        int oldT = mTop;
        int oldB = mBottom;
        int oldR = mRight;
        // 判断视图大小或者位置是否发生改变
        boolean changed = isLayoutModeOptical(mParent) ?
                setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);
        // 发生改变        
        if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
            onLayout(changed, l, t, r, b);
            // 对于单一View的laytou过程：由于单一View是没有子View的，故onLayout（）是一个空实现->>分析3
            // 对于ViewGroup的laytou过程：由于确定位置与具体布局有关，所以onLayout（）在ViewGroup为1个抽象方法，需重写实现

            ...
        }
        ...
    }
```

![IAqjZd.png](https://z3.ax1x.com/2021/11/03/IAqjZd.png)

### ViewGroup 的 layout 过程

> ViewGroup 的 layout 过程确定位置与具体的布局有关，所以在 ViewGroup 中是一个抽象方法，需要重写实现

复写` onLayout()`步骤:
1. 遍历所有子 View
2. 根据自身需求计算当前子 View 的四个位置值（需自身实现）
3. 根据上述 4 个位置的计算值，设置子 View 的 4 个顶点：调用子 View 的 layout 方法，即确定了子 View 在父容器里的位置

```java
  // ViewGroup的onLayout实现的大致思路
  protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
     // 参数说明
     // changed 当前View的大小和位置改变了 
     // left 左部位置  top 顶部位置  right 右部位置  bottom 底部位置
     // 1. 遍历子View：循环所有子View
     for (int i=0; i<getChildCount(); i++) {
           View child = getChildAt(i);   
           // 2. 计算当前子View的四个位置值
           // 2.1 位置的计算逻辑需自己实现，也是自定义View的关键
           calculate();
           // 2.2 对计算后的位置值进行赋值
           int mLeft  = Left
           int mTop  = Top
           int mRight = Right
           int mBottom = Bottom
 
         // 3. 根据上述4个位置的计算值设置子View的4个顶点：调用子view的layout() & 传递计算过的参数
         // 即确定了子View在父容器的位置
         child.layout(mLeft, mTop, mRight, mBottom);
         // 该过程类似于单一View的layout过程中的layout()和onLayout()
     }
  }
```

![IAL1L4.png](https://z3.ax1x.com/2021/11/03/IAL1L4.png)

#### ViewGroup 子类（LinearLayout）的 layout 过程分析

```java
  @Override
  protected void onLayout(boolean changed, int l, int t, int r, int b) {
      // 根据自身方向属性，而选择不同的处理方式
      if (mOrientation == VERTICAL) {
          layoutVertical(l, t, r, b);
      } else {
          layoutHorizontal(l, t, r, b);
      }
  }
```
根据 LinearLayout 的方向（vertical、horizontal）进入不同的布局过程，这里我们只选垂直方向的布局过程，即layoutVertical()

```java
void layoutVertical(int left, int top, int right, int bottom) {
    // 子View的数量
    final int count = getVirtualChildCount();
    // 1. 遍历子View
    for (int i = 0; i < count; i++) {
        final View child = getVirtualChildAt(i);
        if (child == null) {
            childTop += measureNullChild(i);
        } else if (child.getVisibility() != GONE) {
            // 2. 计算子View的测量宽 / 高值
            final int childWidth = child.getMeasuredWidth();
            final int childHeight = child.getMeasuredHeight();
 
            // 3. 确定自身子View的位置
            // 即：递归调用子View的setChildFrame()，实际上是调用了子View的layout() ->>分析2
            setChildFrame(child, childLeft, childTop + getLocationOffset(child), childWidth, childHeight);
 
            // childTop逐渐增大，即后面的子元素会被放置在靠下的位置
            // 这符合垂直方向的LinearLayout的特性
            childTop += childHeight + lp.bottomMargin + getNextLocationOffset(child);
 
            i += getChildrenSkipCount(child, i);
        }
    }
}
 
private void setChildFrame( View child, int left, int top, int width, int height){
    // setChildFrame（）仅仅只是调用了子View的layout（）而已
    child.layout(left, top, left ++ width, top + height);
    // 在子View的layout（）又通过调用setFrame（）确定View的四个顶点
    // 即确定了子View的位置
    // 如此不断循环确定所有子View的位置，最终确定ViewGroup的位置
}
```

### getMeasureWidth 和 getWidth 区别

> 某些情况下，View 需要多次 measure 才能确定自己的测量宽高，在前几次的测量过程中，其得出的测量宽高有可能和最终宽高不一致.，但最终来说，测量宽高还是和最终宽高相同。

![IAjrmF.png](https://z3.ax1x.com/2021/11/03/IAjrmF.png)