---
title: 源码学习 - LiveData
tags:
  - 源码解析
updated: 1646214225000
categories: Android
thumbnail: 'https://s4.ax1x.com/2022/03/02/b8oz5R.md.png'
date: 2022-03-02 17:47:39
---

> [`LiveData`](https://developer.android.google.cn/reference/androidx/lifecycle/LiveData) 是一种可观察的数据存储器类。与常规的可观察类不同，LiveData 具有生命周期感知能力，意指它遵循其他应用组件（如 Activity、Fragment 或 Service）的生命周期。这种感知能力可确保 LiveData 仅更新处于活跃生命周期状态的应用组件观察者

<!--more-->

### 优点

- 确保界面符合数据状态
	+ LiveData 遵循观察者模式。当底层数据发生变化时，LiveData 会通知 Observer 对象。您可以整合代码以在这些 Observer 对象中更新界面。这样一来，您无需在每次应用数据发生变化时更新界面，因为观察者会替您完成更新。
- 不会发生内存泄漏
	+ 观察者会绑定到 Lifecycle 对象，并在其关联的生命周期遭到销毁后进行自我清理。
- 不会因 Activity 停止而导致崩溃
	+ 如果观察者的生命周期处于非活跃状态（如返回栈中的 Activity），则它不会接收任何 LiveData 事件。
- 不再需要手动处理生命周期
	+ 界面组件只是观察相关数据，不会停止或恢复观察。LiveData 将自动管理所有这些操作，因为它在观察时可以感知相关的生命周期状态变化。
- 数据始终保持最新状态
	+ 如果生命周期变为非活跃状态，它会在再次变为活跃状态时接收最新的数据。例如，曾经在后台的 Activity 会在返回前台后立即接收最新的数据。
- 适当的配置更改
	+ 如果由于配置更改（如设备旋转）而重新创建了 Activity 或 Fragment，它会立即接收最新的可用数据。
- 共享资源
	+ 您可以使用单例模式扩展 LiveData 对象以封装系统服务，以便在应用中共享它们。LiveData 对象连接到系统服务一次，然后需要相应资源的任何观察者只需观察 LiveData 对象。如需了解详情，请参阅扩展 LiveData。


### 使用
```kotlin
class DetailVideModel : ViewModel {
	  val getTopTabLiveData by lazy {  MutableLiveData<Boolean>() }
}
class MediaDetailsActivity : BaseActivity(){
	  viewModel.getTopTabLiveData.observe(this){
	       // do some thing    
	  }
}
```
![I8k2xH.png](https://z3.ax1x.com/2021/11/08/I8k2xH.png =100x100)
### observer()做了什么

> 将指定事件跟生命周期绑定

```java
public abstract class LiveData<T> {
    /* 在给定所有者的生命周期内将给定的观察者添加到观察者列表中。 */
    @MainThread
    public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T> observer) {
        assertMainThread("observe");
        // 判断生命周期为销毁,则忽略
        if (owner.getLifecycle().getCurrentState() == DESTROYED) return;
        // 实例一个新的生命周期绑定观察者
        LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
        // 判断是否之前已经赋值,来防止有煞笔重复调用,专治代码水土不服
        ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
        // 注意看LifecycleBoundObserver.isAttachedTo()
        // 如果之前已经赋值但又有煞笔在不同LifecycleOwner(生命周期管理者)中调用,话不多说,直接给crash,google就是牛逼
        if (existing != null && !existing.isAttachedTo(owner)) {
            throw new IllegalArgumentException("Cannot add the same observer"
                    + " with different lifecycles");
        }
        // 如果已经赋值且上面煞笔处理没问题,就不再做处理了
        if (existing != null) {
            return;
        }
        // 看到了没有,核心啊,精髓啊,给当前LifecycleOwner加入Observer(观察者),在生命周期各个阶段响应事件
        owner.getLifecycle().addObserver(wrapper);
    }
 }   
```

看看mObservers.putIfAbsent:
```java
/* LinkedList，它伪装成一个Map并支持在迭代期间进行修改。它不是线程安全的 */
public class SafeIterableMap<K, V> implements Iterable<Map.Entry<K, V>> {

    /* 如果指定的键尚未与值关联，则将其与给定值关联,返回与指定键关联的前一个值，如果该键没有映射，则null */
    public V putIfAbsent(@NonNull K key, @NonNull V v) {
        Entry<K, V> entry = get(key);
        // 如果给定的值已经存在,则返回值
        if (entry != null) return entry.mValue;
        // 否则存进去,真似厉害啊
        put(key, v);
        // 返回代表之前不存在的null值,这就是理解有木有 
        return null;
    }
 }   
```

看看LifecycleBoundObserver :
```java
    class LifecycleBoundObserver extends ObserverWrapper implements LifecycleEventObserver {
        @NonNull
        // 生命周期管理者
        final LifecycleOwner mOwner;
        // 看到这里没,直接赋值了,有什么好说的
        LifecycleBoundObserver(@NonNull LifecycleOwner owner, Observer<? super T> observer) {
            super(observer);
            mOwner = owner;
        }

        // 有点意思,这里判断是否处在活跃状态,就可以他妈的做正确的生命周期回调响应
        @Override
        boolean shouldBeActive() {
            return mOwner.getLifecycle().getCurrentState().isAtLeast(STARTED);
        }

        @Override
        public void onStateChanged(@NonNull LifecycleOwner source,
                @NonNull Lifecycle.Event event) {
            Lifecycle.State currentState = mOwner.getLifecycle().getCurrentState();
            // 若生命周期已为销毁状态, 则移除 observer, 避免内存泄露
            if (currentState == DESTROYED) {
                removeObserver(mObserver);
                return;
            }
            Lifecycle.State prevState = null;
            while (prevState != currentState) {
                prevState = currentState;
                activeStateChanged(shouldBeActive());
                currentState = mOwner.getLifecycle().getCurrentState();
            }
        }
        // 判断两个生命周期管理者是否为同一个
        @Override
        boolean isAttachedTo(LifecycleOwner owner) {
            return mOwner == owner;
        }

        @Override
        void detachObserver() {
            mOwner.getLifecycle().removeObserver(this);
        }
    }
```

好了到这LiveData.observe()已经观察完毕了,到看看LiveData.postValue()了,
别说,google还是有点讲究的,看看这个官方提示:
```java
liveData.postValue("a");// 异步传值
liveData.setValue("b"); //同步传值
// 结果: 值“b”将首先设置，然后主线程将用值“a”覆盖它。
// 如果在此主线程执行postValue()之前多次调用postValue()，则只会调度最后一个值
```

### postValue()做了什么

> 他妈的,还能做什么,就是传值啊,这还用说的文章到此结束了,不会吧,不会吧,异步传值得注意处理多线程下的共享变量问题呢

![I8mrdA.png](https://z3.ax1x.com/2021/11/08/I8mrdA.png =100x100)

```java
...
protected void postValue(T value) {
   boolean postTask;
   synchronized (mDataLock) {
     // 判断之前是否已经赋值过了
     postTask = mPendingData == NOT_SET;
     // 给mPendingData赋值 
     mPendingData = value;
   }
   // 看到没有,他妈的,之前已经赋值就return了,好气哦,干嘛这样呢
   // 嘿,看下mPostValueRunnable没有,他就是防止煞笔多线程下多次赋相同值做的
   if (!postTask) return;
   // 把传值线程放到主线程执行
   ArchTaskExecutor.getInstance().postToMainThread(mPostValueRunnable);
}
...
// 未赋值状态
static final Object NOT_SET = new Object()
// 默认就是未赋值状态
volatile Object mPendingData = NOT_SET
// 传值线程
private final Runnable mPostValueRunnable = new Runnable() {
  @SuppressWarnings("unchecked")
  @Override
  public void run() {
    // 看到没有,真正传值都是通过newValue来传的好不好
    Object newValue;
    synchronized (mDataLock) {
      newValue = mPendingData;
      // 嘿嘿,这里回归默认未赋值状态了
      mPendingData = NOT_SET;
    }
    // 最终还是调到setValue啊,果然啊,条条道路通setValue啊
    setValue((T) newValue);
  }
};
```

### setValue()

> > 有什么好说的,其他的一枪秒了,我才是传输数据的核心代码好不好,屁,dispatchingValue才是

```java
// 设置传输数据。 如果有活跃的ObserverWrapper(观察者)，值将被分发给他们。    
protected void setValue(T value) {
  assertMainThread("setValue");
  mVersion++;
  mData = value;
  dispatchingValue(null);
}
@SuppressWarnings("WeakerAccess") /* synthetic access */
void dispatchingValue(@Nullable ObserverWrapper initiator) {
        // 判断事件分发是否已经在执行,是则打断
        if (mDispatchingValue) {
            mDispatchInvalidated = true;
            return;
        }
        // 事件分发开始
        mDispatchingValue = true;
        do {
            mDispatchInvalidated = false;
            if (initiator != null) {
                considerNotify(initiator);
                initiator = null;
            } else {
                for (Iterator<Map.Entry<Observer<? super T>, ObserverWrapper>> iterator =
                        mObservers.iteratorWithAdditions(); iterator.hasNext(); ) {
                    considerNotify(iterator.next().getValue());
                    if (mDispatchInvalidated) break;
                }
            }
        } while (mDispatchInvalidated);
        // 事件分发结束
        mDispatchingValue = false;
}
...
private void considerNotify(ObserverWrapper observer) {
        // 如果生命周期管理者已经销毁,则忽略
        if (!observer.mActive) {
            return;
        }
        // 在分发之前检查下最先状态,如果不在活跃阶段,则不改变状态
        if (!observer.shouldBeActive()) {
            observer.activeStateChanged(false);
            return;
        }
        // 判断version , 因为从上面来看,每次赋值version都会变得
        // 他妈的的,这样就不会陷入多次分发,保证只取最先的传值
        if (observer.mLastVersion >= mVersion) {
            return;
        }
        observer.mLastVersion = mVersion;
        // 执行每个生命周期管理者的观察者的事件分发
        observer.mObserver.onChanged((T) mData);
}
```

get到没有?
![I8YNWQ.png](https://z3.ax1x.com/2021/11/08/I8YNWQ.png)]