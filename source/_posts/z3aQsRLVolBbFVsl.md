---
title: 自定义View(9) - 事件分发
tags:
  - 图形绘制
categories: Android
updated: 1635948234000
date: 2021-11-03 22:07:31
---

> 事件分发的过程其实也就是事件传递过程。事件传递的顺序由Acticity传递到根View，这个根View 通常是一个ViewGroup(ViewGroup本身也是View的子类)，然后再传递给ViewGroup下的子View, 若事件在自上而下的传递过程中一直没有被消费，则事件会反向向上传递，此时父ViewGroup可以对事件进行消费，若仍然没有被消费的话，最后会回到Activity的onTouchEvent

所以很多时候如果有冲突的话,最先消费的是最底部的子View
![IE3gqf.png](https://z3.ax1x.com/2021/11/03/IE3gqf.png)
<!-- more -->
### 事件分发Method

|方法|作用|调用时刻|
|:--:|:--:|:--:|
|dispatchTouchEvent()|用来进行事件传递 | 如果事件能够传递给当前 View，那么此方法一定会被调用|
|onInterceptTouchEvent()|用来是否拦截事件 | 如果当前 View 拦截了某个事件，那么在同一个事件序列当中，此方法不会被再次调用|
|onTouchEvent()|处理事件|在 dispatchTouchEvent()方法中调用|


### 事件分发顺序

Activity&Window
```java
public boolean dispatchTouchEvent(MotionEvent ev) {
    if (ev.getAction() == MotionEvent.ACTION_DOWN) {
        onUserInteraction();
    }
    // 当事件没有被任何子View 消费时,即这里为false时,最终执行自己的nTouchEvent
    if (getWindow().superDispatchTouchEvent(ev)) {
        return true;
    }
    return onTouchEvent(ev);
}
```
DecorView(FrameLayout 的子类)
```java
public class PhoneWindow extends Window implements MenuBuilder.Callback {
	@Override
	public boolean superDispatchTouchEvent(MotionEvent event) {
	    return mDecor.superDispatchTouchEvent(event);
	}
 
	private final class DecorView extends FrameLayout implements RootViewSurfaceTaker{
	    //FrameLayout 并没有重写dispatchTouchEvent方法，所以事件开始交由 ViewGroup 的 dispatchTouchEvent 开始分发了
	    public boolean superDispatchTouchEvent(MotionEvent event) {
	        return super.dispatchTouchEvent(event);
	    }
	}
}
```
ViewGroup

- 判断事件是够需要被 ViewGroup 拦截
- 遍历所有子View，逐个分发事件
- 将事件交给ViewGroup自己或者目标子View处理

```java
@Override
public boolean dispatchTouchEvent(MotionEvent ev) {
 
    boolean handled = false;
    if (onFilterTouchEventForSecurity(ev)) {
        final int action = ev.getAction();
    
        // 先检验事件是否需要被ViewGroup拦截
        final boolean intercepted;
        if (actionMasked == MotionEvent.ACTION_DOWN || mFirstTouchTarget != null) {
            // 校验是否给mGroupFlags设置了FLAG_DISALLOW_INTERCEPT标志位
            final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
            if (!disallowIntercept) {
                // 走onInterceptTouchEvent判断是否拦截事件
                intercepted = onInterceptTouchEvent(ev);
            } else {
                intercepted = false;
            }
        } else {
            intercepted = true;
        }
    
        final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
        if (!canceled && !intercepted) {
            // 注意ACTION_DOWN等事件才会走遍历所有子View的流程
            if (actionMasked == MotionEvent.ACTION_DOWN
                    || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                    || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
            
                // 开始遍历所有子View开始逐个分发事件
                final int childrenCount = mChildrenCount;
                if (childrenCount != 0) {
                    for (int i = childrenCount - 1; i >= 0; i--) {
                        // 判断触摸点是否在这个View的内部
                        final View child = children[i];
                        if (!canViewReceivePointerEvents(child)
                                || !isTransformedTouchPointInView(x, y, child, null)) {
                            continue;
                        }
                    
                        // 事件被子View消费，退出循环，不再继续分发给其他子View
                        if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                        
                            // addTouchTarget内部将mFirstTouchTarget设置为child，即不为null
                            newTouchTarget = addTouchTarget(child, idBitsToAssign);
                            alreadyDispatchedToNewTouchTarget = true;
                            break;
                        }
                    }
                }
            }
        }
        // 事件未被任何子View消费，自己处理
        if (mFirstTouchTarget == null) {
            // No touch targets so treat this as an ordinary view.
            handled = dispatchTransformedTouchEvent(ev, canceled, null,
                    TouchTarget.ALL_POINTER_IDS);
        } else {
            // 将MotionEvent.ACTION_DOWN后续事件分发给mFirstTouchTarget指向的View
            TouchTarget predecessor = null;
            TouchTarget target = mFirstTouchTarget;
            while (target != null) {
                final TouchTarget next = target.next;
                // 如果已经在上面的遍历过程中传递过事件，跳过本次传递
                if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                    handled = true;
                } else {
                    final boolean cancelChild = resetCancelNextUpFlag(target.child)
                            || intercepted;
                    if (dispatchTransformedTouchEvent(ev, cancelChild,
                            target.child, target.pointerIdBits)) {
                        handled = true;
                    }
                
                }
                predecessor = target;
                target = next;
            }
        }
        // Update list of touch targets for pointer up or cancel, if needed.
        if (canceled
                || actionMasked == MotionEvent.ACTION_UP
                || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
            resetTouchState();
        } else if (split && actionMasked == MotionEvent.ACTION_POINTER_UP) {
            final int actionIndex = ev.getActionIndex();
            final int idBitsToRemove = 1 << ev.getPointerId(actionIndex);
            removePointersFromTouchTargets(idBitsToRemove);
        }
    }
    return handled;
}
 
private void resetTouchState() {
    clearTouchTargets();
    resetCancelNextUpFlag(this);
    mGroupFlags &= ~FLAG_DISALLOW_INTERCEPT;
}
 
private void clearTouchTargets() {
    TouchTarget target = mFirstTouchTarget;
    if (target != null) {
        do {
            TouchTarget next = target.next;
            target.recycle();
            target = next;
        } while (target != null);
        mFirstTouchTarget = null;
    }
}
 
private TouchTarget addTouchTarget(View child, int pointerIdBits) {
    TouchTarget target = TouchTarget.obtain(child, pointerIdBits);
    target.next = mFirstTouchTarget;
    mFirstTouchTarget = target;
    return target;
}
 
private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
                                              View child, int desiredPointerIdBits) {
    final boolean handled;
    // 注意传参child为null时，调用的是自己的dispatchTouchEvent
    if (child == null) {
        handled = super.dispatchTouchEvent(event);
    } else {
        handled = child.dispatchTouchEvent(transformedEvent);
    }
    return handled;
}
 
public boolean onInterceptTouchEvent(MotionEvent ev) {
    // 默认不拦截事件
    return false;
}
```

View:
```java
public boolean dispatchTouchEvent(MotionEvent event) {
        if (onFilterTouchEventForSecurity(event)) {
        	// 判断事件是否先交给ouTouch方法处理
            if (mOnTouchListener != null && (mViewFlags & ENABLED_MASK) == ENABLED &&
                    mOnTouchListener.onTouch(this, event)) {
                return true;
            }
            // onTouch未消费事件，传给onTouchEvent
            if (onTouchEvent(event)) {
                return true;
            }
        }
        return false;
}
```

### 综上

-   View 事件分发的本质是递归。
-   递归的本质是，任务的下发和结果的上报。
-   View 事件分发设计成递归，是为了配合 View 的排版规则，形成符合用户直觉的触控体验。
-   View 事件分发的对象是一个 MotionEvent。
-   一次用户触控操作包含多个 MotionEvent（例如从 ACTION_DOWN 到 ACTION_UP ），也即会走多次事件分发流程。
-   一次 View 事件分发流程包含 “递” 流程和 “归” 流程，“递” 流程可以因 ViewGroup 的拦截而提前步入 “归” 流程。
-   child 可以通过 getParent.requestDisallowInterceptTouchEvent 阻止父容器的拦截。因而需要差异化地配置阈值，来确保 child 执行 getParent.requestDisallowInterceptTouchEvent 优先于父容器 onInterceptTouchEvent 返回 true（不然都先被拦截了，child 哪有机会阻止？）
-   在“归”流程中，唯有当前层级的 super.dispatchTouchEvent 返回了 true，才认定被消费，被消费前，下级都有干活，只是结果不 OK。被消费后，上级都不需要干活，直接向上传达消费者的功。

### 事件冲突解决

- 从父View着手, 重写onInterceptTouchEvent方法，在父View需要拦截的时候拦截，不要的时候返回false
- 从子View着手, 重写子 View的dispatchTouchEvent方法，在Action_down 动作中通过方法 requestDisallowInterceptTouchEvent（true） 先请求 父 View不要拦截事件，这样保证子 View 能够接受到 Action_move 事件，再在 Action_move 动作中根据自己的逻辑是否要拦截事件，不需要拦截事件的话再交给 父 View 处理

### Refer

[学习 View 事件分发，就像外地人上了黑车！ - 掘金 (juejin.cn)](https://juejin.cn/post/6844903894103883789)
[View 事件分发机制，看这一篇就够了 - 掘金 (juejin.cn)](https://juejin.cn/post/6965649194744807461)
[Android 手把手进阶自定义View（十）- 事件分发机制解析_lerendan的博客-CSDN博客](https://blog.csdn.net/u010289802/article/details/86169939)