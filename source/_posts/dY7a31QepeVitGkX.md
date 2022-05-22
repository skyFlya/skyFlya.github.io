---
title: 自定义View(10) - GestureDetector
tags:
  - 图形绘制
updated: 1635948469000
categories: Android
date: 2021-11-03 22:08:39
---

### GestureDetector

| 监听器 | 简介 |
|--|--|
| OnContextClickListener | 这个很容易让人联想到ContextMenu，然而它和ContextMenu并没有什么关系，它是在Android6.0(API 23)才添加的一个选项，是用于检测外部设备上的按钮是否按下的，例如蓝牙触控笔上的按钮，一般情况下，忽略即可。 |
| OnDoubleTapListener  | 双击事件，有三个回调类型：双击(DoubleTap)、单击确认(SingleTapConfirmed) 和 双击事件回调(DoubleTapEvent) |
| OnGestureListener | 手势检测，主要有以下类型事件：按下(Down)、 一扔(Fling)、长按(LongPress)、滚动(Scroll)、触摸反馈(ShowPress) 和 单击抬起(SingleTapUp) |
| SimpleOnGestureListener  | 这个是上述三个接口的空实现，一般情况下使用这个比较多，也比较方便 |
<!-- more -->
```java
// 1.创建一个监听回调
SimpleOnGestureListener listener = new SimpleOnGestureListener() {
    @Override public boolean onDoubleTap(MotionEvent e) {
        Toast.makeText(MainActivity.this, "双击666", Toast.LENGTH_SHORT).show();
        return super.onDoubleTap(e);
    }
};
 
// 2.创建一个检测器
final GestureDetector detector = new GestureDetector(this, listener);
 
// 3.给监听器设置数据源
view.setOnTouchListener(new View.OnTouchListener() {
    @Override public boolean onTouch(View v, MotionEvent event) {
        return detector.onTouchEvent(event);
    }
});
```

### ScaleGestureDetector

> Android 缩放手势检测 ScaleGestureDetector，在大多数的情况下缩放手势都不是单独存在的，需要配合其它的手势来使用

#### sample

```java
public class ScaleGestureDemoView extends View {
    private static final String TAG = "ScaleGestureDemoView";
 
    private ScaleGestureDetector mScaleGestureDetector;
 
    public ScaleGestureDemoView(Context context) {
        super(context);
    }
 
    public ScaleGestureDemoView(Context context, @Nullable AttributeSet attrs) {
        super(context, attrs);
        initScaleGestureDetector();
    }
 
    private void initScaleGestureDetector() {
        mScaleGestureDetector = new ScaleGestureDetector(getContext(), 
                new ScaleGestureDetector.SimpleOnScaleGestureListener() {
            @Override
            public boolean onScaleBegin(ScaleGestureDetector detector) {
                return true;
            }
 
            @Override
            public boolean onScale(ScaleGestureDetector detector) {
                Log.i(TAG, "focusX = " + detector.getFocusX());       // 缩放中心，x坐标
                Log.i(TAG, "focusY = " + detector.getFocusY());       // 缩放中心y坐标
                Log.i(TAG, "scale = " + detector.getScaleFactor());   // 缩放因子
                return true;
            }
 
            @Override
            public void onScaleEnd(ScaleGestureDetector detector) {
            }
        });
    }
 
    @Override
    public boolean onTouchEvent(MotionEvent event) {
        mScaleGestureDetector.onTouchEvent(event);
        return true;
    }
}
```

