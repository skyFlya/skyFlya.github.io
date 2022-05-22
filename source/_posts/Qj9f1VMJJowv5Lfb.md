---
title: 自定义View(6) - measure 测量过程
tags:
  - 图形绘制
categories: Android
updated: 1635947955000
date: 2021-11-03 22:00:58
---

![QJQEcR.jpg](https://s2.ax1x.com/2019/12/06/QJQEcR.jpg)
<!-- more -->
### MeasureSpec

> MeasureSpec 代表测量规格，是一个 32 位的 int 值，高 2 位代表 SpecMode（测量模式），低 30 位代表 SpecSize（测量大小）

![IF7OM9.png](https://z3.ax1x.com/2021/11/02/IF7OM9.png)

MeasureSpec 通过将 SpecMode 和 SpecSize 打包成一个 int 值来避免过多的内存分配，并提供了打包和解包的方法

```java
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        int widthMode = MeasureSpec.getMode(widthMeasureSpec);
        int widthSize = MeasureSpec.getSize(widthMeasureSpec);
        int heightMode = MeasureSpec.getMode(heightMeasureSpec);
        int heightSize = MeasureSpec.getSize(heightMeasureSpec);
        ...
    }
```

#### SpecMode

![IFbTc4.png](https://z3.ax1x.com/2021/11/02/IFbTc4.png)


### MeasureSpec值计算

> View 的 MeasureSpec 值是由 View 的布局参数和父容器 的 MeasureSpec 值计算

![IAVp1e.png](https://z3.ax1x.com/2021/11/03/IAVp1e.png)

```java
    /**
     * 源码分析：getChildMeasureSpec（）
     * 作用：根据父视图的MeasureSpec & 布局参数LayoutParams，计算单个子View的MeasureSpec
     * 注：子view的大小由父view的MeasureSpec值 和 子view的LayoutParams属性 共同决定
     * 参数说明
     * @param spec           父view的详细测量值 (MeasureSpec)
     * @param padding        view当前尺寸的的内边距和外边距(padding, margin)
     * @param childDimension 子视图的布局参数（宽 / 高）
     **/
    public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
        //父view的测量模式
        int specMode = MeasureSpec.getMode(spec);
        //父view的大小
        int specSize = MeasureSpec.getSize(spec);
        //通过父view计算出的子view大小 = 父大小-边距（父要求的大小，但子view不一定用这个值）   
        int size = Math.max(0, specSize - padding);
 
        //子view想要的实际大小和模式（需要计算）  
        int resultSize = 0;
        int resultMode = 0;
 
        //通过父view的MeasureSpec和子view的LayoutParams确定子view的大小  
        switch (specMode) {
            //当父view的模式为EXACITY时，父view强加给子view确切的值
            //一般是父view设置为match_parent或者固定值的ViewGroup 
            case MeasureSpec.EXACTLY:
                // 当子view的LayoutParams>0，即有确切的值  
                if (childDimension >= 0) {
                    //子view大小为子自身所赋的值，模式大小为EXACTLY  
                    resultSize = childDimension;
                    resultMode = MeasureSpec.EXACTLY;
                    
                    //当子view的LayoutParams为MATCH_PARENT时(-1)  
                } else if (childDimension == LayoutParams.MATCH_PARENT) {
                    //子view大小为父view大小，模式为EXACTLY  
                    resultSize = size;
                    resultMode = MeasureSpec.EXACTLY;

                    // 当子view的LayoutParams为WRAP_CONTENT时(-2)      
                } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                    //子view决定自己的大小，但最大不能超过父view，模式为AT_MOST  
                    resultSize = size;
                    resultMode = MeasureSpec.AT_MOST;
                }
                break;
 
            // 当父view的模式为AT_MOST时，父view强加给子view一个最大的值。（一般是父view设置为wrap_content）  
            case MeasureSpec.AT_MOST:
                // 道理同上  
                if (childDimension >= 0) {
                    resultSize = childDimension;
                    resultMode = MeasureSpec.EXACTLY;
                } else if (childDimension == LayoutParams.MATCH_PARENT) {
                    resultSize = size;
                    resultMode = MeasureSpec.AT_MOST;
                } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                    resultSize = size;
                    resultMode = MeasureSpec.AT_MOST;
                }
                break;
 
            // 当父view的模式为UNSPECIFIED时，父容器不对view有任何限制，要多大给多大
            // 多见于ListView、GridView  
            case MeasureSpec.UNSPECIFIED:
                if (childDimension >= 0) {
                    // 子view大小为子自身所赋的值  
                    resultSize = childDimension;
                    resultMode = MeasureSpec.EXACTLY;
                } else if (childDimension == LayoutParams.MATCH_PARENT) {
                    // 因为父view为UNSPECIFIED，所以MATCH_PARENT的话子类大小为0  
                    resultSize = 0;
                    resultMode = MeasureSpec.UNSPECIFIED;
                } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                    // 因为父view为UNSPECIFIED，所以WRAP_CONTENT的话子类大小为0  
                    resultSize = 0;
                    resultMode = MeasureSpec.UNSPECIFIED;
                }
                break;
        }
        return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
    }
```

![IAemLj.png](https://z3.ax1x.com/2021/11/03/IAemLj.png)

### Measure 过程
![IAuMTA.png](https://z3.ax1x.com/2021/11/03/IAuMTA.png)


布局过程自定义的方式
1.  重写  `onMeasure()`  来修改已有的  `View`  的尺寸；
2.  重写  `onMeasure()`  来全新定制自定义  `View`  的尺寸；
3.  重写  `onMeasure()`  和  `onLayout()`  来全新定制自定义  `ViewGroup`  的内部布局。

#### View的measure

View 的 measure 过程由其`measure()` 方法完成：
```java
    /**
    * 源码分析：measure（）
    * 定义：Measure过程的入口；属于View.java类 & final类型，即子类不能重写此方法
    * 作用：基本测量逻辑的判断
    **/ 
    public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
        ...
        // 判断是否需要重新测量           
        if (forceLayout || needsLayout) {
            ...
            // 判断是否有缓存
            int cacheIndex = forceLayout ? -1 : mMeasureCache.indexOfKey(key);
            if (cacheIndex < 0 || sIgnoreMeasureCache) {
                // 开始测量
                onMeasure(widthMeasureSpec, heightMeasureSpec);
                mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
            } else {
                long value = mMeasureCache.valueAt(cacheIndex);
                // Casting a long to int drops the high 32 bits, no mask needed
                setMeasuredDimensionRaw((int) (value >> 32), (int) value);
                mPrivateFlags3 |= PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
            }
            ...
        }
        ...
    }  
```

`setMeasuredDimension()`方法会设置 View 的宽/高的测量值，因此我们只需要看`getDefaultSize()` 方法即可

![IAGYZT.png](https://z3.ax1x.com/2021/11/03/IAGYZT.png)

```java
   /**
    * onMeasure（）要的做是事情
    * 1. 根据View宽/高的测量规格计算View的宽/高值：getDefaultSize()
    * 2. 存储测量后的View宽 / 高：setMeasuredDimension()
    **/
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
    }
    
    /**
     * @param size        提供的默认大小
     * @param measureSpec 宽/高的测量规格
     */
    public static int getDefaultSize(int size, int measureSpec) {
        // 设置默认大小
        int result = size;
        // 获取宽/高测量规格的模式 & 测量大小
        int specMode = MeasureSpec.getMode(measureSpec);
        int specSize = MeasureSpec.getSize(measureSpec);
        switch (specMode) {
          // 模式为UNSPECIFIED时，使用提供的默认大小 = 参数Size
          case MeasureSpec.UNSPECIFIED:
            result = size;
            break;
          // 模式为AT_MOST,EXACTLY时，使用View测量后的宽/高值 = measureSpec中的Size  
          case MeasureSpec.AT_MOST:
          case MeasureSpec.EXACTLY:
            result = specSize;
            break;
        }
        // 返回View的宽/高值
        return result;
    }  
```
当模式是 UNSPECIFIED 时，使用的是提供的默认大小:
- 若 View 无设置背景，那么 View 的宽度 = mMinWidth。mMinWidth为 android:minWidth属性所指定的值，默认为 0
- 若 View设置了背景，View 的宽度为 mMinWidth 和 mBackground.getMinimumWidth()中的最大值。

```java
    protected int getSuggestedMinimumWidth() {
        return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
    }
```
|过程| 行为作用 |
|:--:|:--:|
| ![IAJ9mV.png](https://z3.ax1x.com/2021/11/03/IAJ9mV.png) | ![IAJ1te.png](https://z3.ax1x.com/2021/11/03/IAJ1te.png) |


#### ViewGroup的measure

> ViewGroup是个抽象类,不同ViewGroup的onMeasure的实现都个不相同,除了完成自己的 measure 过程以外，还会遍历去调用所有子元素的 measure 方法，各个子元素再递归去执行这个过程

1. 遍历所有子 View & 测量：measureChildren()
2. 合并所有子 View 的尺寸大小，最终得到 ViewGroup 的测量值（需自身实现）
3. 存储测量后 View 宽/高的值：调用 setMeasuredDimension()  

```java
  @Override
  protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        // 定义存放测量后的View宽/高的变量
        int widthMeasure ;
        int heightMeasure ;
        // 1. 遍历所有子 View & 测量(measureChildren())
        measureChildren(widthMeasureSpec, heightMeasureSpec)；
        // 2. 合并所有子View的尺寸大小，最终得到ViewGroup父视图的测量值
        // 需自身实现
        measureMerge();
        // 3. 存储测量后View宽/高的值：调用setMeasuredDimension()
        // 类似单一View的过程，此处不作过多描述
        setMeasuredDimension(widthMeasure,  heightMeasure);
  }
```

`measureChildren()`遍历子 View 并且调用 measureChild() 进行下一步测量
```java
    protected void measureChildren(int widthMeasureSpec, int heightMeasureSpec) {
        // 要求该视图的所有子视图度量自己，同时考虑该视图的MeasureSpec要求及其填充。我们跳过了处于GONE状态的子节点
        final int size = mChildrenCount;
        final View[] children = mChildren;
        for (int i = 0; i < size; ++i) {
            final View child = children[i];
            if ((child.mViewFlags & VISIBILITY_MASK) != GONE) {
                measureChild(child, widthMeasureSpec, heightMeasureSpec);
            }
        }
    }
```

|过程| 行为作用 |
|:--:|:--:|
|![IAokqI.png](https://z3.ax1x.com/2021/11/03/IAokqI.png)|![IAo1Ln.png](https://z3.ax1x.com/2021/11/03/IAo1Ln.png)|


##### LinearLayout measure分析

```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    // 根据不同的布局属性进行不同的计算
    if (mOrientation == VERTICAL) {
        measureVertical(widthMeasureSpec, heightMeasureSpec);
    } else {
        measureHorizontal(widthMeasureSpec, heightMeasureSpec);
    }
}

void measureVertical(int widthMeasureSpec, int heightMeasureSpec) {
    // 获取垂直方向上的子View个数
    final int count = getVirtualChildCount();
 
    // 遍历子View获取其高度，并记录下子View中最高的高度数值
    for (int i = 0; i < count; ++i) {
        final View child = getVirtualChildAt(i);
 
        // 子View不可见，直接跳过该View的measure过程，getChildrenSkipCount()返回值恒为0
        // 注：若view的可见属性设置为VIEW.INVISIBLE，还是会计算该view大小
        if (child.getVisibility() == View.GONE) {
           i += getChildrenSkipCount(child, i);
           continue;
        }
 
        // 记录子View是否有weight属性设置，用于后面判断是否需要二次measure
        totalWeight += lp.weight;
 
        if (heightMode == MeasureSpec.EXACTLY && lp.height == 0 && lp.weight > 0) {
            // 如果LinearLayout的specMode为EXACTLY且子View设置了weight属性，在这里会跳过子View的measure过程
            // 同时标记skippedMeasure属性为true，后面会根据该属性决定是否进行第二次measure
            // 若LinearLayout的子View设置了weight，会进行两次measure计算，比较耗时
            // 这就是为什么LinearLayout的子View需要使用weight属性时候，最好替换成RelativeLayout布局
            final int totalLength = mTotalLength;
            mTotalLength = Math.max(totalLength, totalLength + lp.topMargin + lp.bottomMargin);
            skippedMeasure = true;
        } else {
            int oldHeight = Integer.MIN_VALUE;
            // 步骤1：遍历所有子View & 测量：measureChildren（）
            // 注：该方法内部，最终会调用measureChildren（），从而 遍历所有子View & 测量
            measureChildBeforeLayout(child, i, widthMeasureSpec, 0, heightMeasureSpec, totalWeight == 0 ? mTotalLength : 0);
                   ...
            //步骤2：合并所有子View的尺寸大小,最终得到ViewGroup父视图的测量值（自身实现）
            final int childHeight = child.getMeasuredHeight();
 
            // 1. mTotalLength用于存储LinearLayout在竖直方向的高度
            final int totalLength = mTotalLength;
 
            // 2. 每测量一个子View的高度， mTotalLength就会增加
            mTotalLength = Math.max(totalLength, totalLength + childHeight + lp.topMargin +
                           lp.bottomMargin + getNextLocationOffset(child));
        }
    }
 
    // 3. 记录LinearLayout占用的总高度
    // 即除了子View的高度，还有本身的padding属性值
    mTotalLength += mPaddingTop + mPaddingBottom;
    int heightSize = mTotalLength;
    int heightSizeAndState = resolveSizeAndState(heightSize, heightMeasureSpec, 0);
 
    //步骤3：存储测量后View宽/高的值：调用setMeasuredDimension()
    setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState), heightSizeAndState);
    ...
}
```

### 获取View的宽高

> Activity 启动时，在 onCreate()、onStart()、onResume() 中均无法正确的得到某个 View 的宽高信息，这是因为 View 的 measure 过程和 Activity 的生命周期方法不是同步执行的

- onWindowFocusChanged()
	- 需要注意的是，onWindowFocusChanged() 会被调用多次，当 Activity 的窗口得到焦点和失去焦点时均会被调用一次。
- View.post(runnable)
- ViewTreeObserver
- 手动调用 View 的 measure 方法

### Refer

[Android 知识体系学习目录_lerendan的博客-CSDN博客](https://blog.csdn.net/u010289802/article/details/80183142)