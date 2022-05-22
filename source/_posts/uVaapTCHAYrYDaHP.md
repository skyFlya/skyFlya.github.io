---
title: 源码学习 - Lifecycle
tags:
  - 源码解析
updated: 1646127156000
categories: Android
thumbnail: 'https://s4.ax1x.com/2022/03/02/b3lHxA.md.png'
date: 2022-03-01 17:33:23
---

> Jetpack Lifecycle 提供了可用于构建生命周期感知型组件的类和接口,从而根据 Activity 或 Fragment 的当前生命周期状态自动调整其行为,记住我们要解析的是Jet Pack Lifecycle ,而不是 原有Activity/Fragment的生命周期流程

<!-- more -->

![space](https://s4.ax1x.com/2022/03/01/blNKTx.png)


### 使用

#### 通过 DefaultLifecycleObserver 实现

> 类可以通过实现 DefaultLifecycleObserver 并替换相应的方法（如 onCreate 和 onStart 等）来监控组件的生命周期状态。然后，您可以通过调用 Lifecycle 类的 addObserver() 方法并传递观察器的实例来添加观察器

```kotlin
// 构建生命周期感知型组件
class MyObserver : DefaultLifecycleObserver {
    override fun onResume(owner: LifecycleOwner) {
        connect() // 在页面 onResume 时连接
    }

    override fun onPause(owner: LifecycleOwner) {
        disconnect() // 在页面 onStop 时连接
    }
}
// 添加生命周期观察器
class MyActivity : AppCompatActivity() {
    override fun onCreate(...) {
	    myLifecycleOwner.getLifecycle().addObserver(MyObserver())
    }
}
```

#### 通过注解实现Licecycle Observer

```kotlin
// 1
interface ILifecycleObserver : LifecycleObserver {

    @OnLifecycleEvent(Lifecycle.Event.ON_CREATE)
    fun onCreate(owner: LifecycleOwner)

    @OnLifecycleEvent(Lifecycle.Event.ON_START)
    fun onStart(owner: LifecycleOwner)

    @OnLifecycleEvent(Lifecycle.Event.ON_RESUME)
    fun onResume(owner: LifecycleOwner)

    @OnLifecycleEvent(Lifecycle.Event.ON_PAUSE)
    fun onPause(owner: LifecycleOwner)

    @OnLifecycleEvent(Lifecycle.Event.ON_STOP)
    fun onStop(owner: LifecycleOwner)

    @OnLifecycleEvent(Lifecycle.Event.ON_DESTROY)
    fun onDestroy(owner: LifecycleOwner)

    @OnLifecycleEvent(Lifecycle.Event.ON_ANY)
    fun onLifecycleChanged(owner: LifecycleOwner, event: Lifecycle.Event)

}
// 2
class MyObserver : ILifecycleObserver {

    override fun onCreate(owner: LifecycleOwner) {}

    override fun onStart(owner: LifecycleOwner) {}

    override fun onResume(owner: LifecycleOwner) {
        connect() // 在页面 onResume 时连接
    }

    override fun onPause(owner: LifecycleOwner) {
        disconnect() // 在页面 onStop 时连接
    }

    override fun onStop(owner: LifecycleOwner) {}

    override fun onDestroy(owner: LifecycleOwner) {}
    
    override fun onLifecycleChanged(owner: LifecycleOwner, event: Lifecycle.Event) {}
}
```

### 解析
#### Class组成

1. 首先我们得知道Activity/Fragment是如何实现LifecycleOwner的

LifecycleOwner:
```java
public interface LifecycleOwner {
    /**
     * Returns the Lifecycle of the provider.
     */
    @NonNull
    Lifecycle getLifecycle();
}
```

可以看到Fragment 和Activity都实现了相关的LifecycleOwner接口
```java
public class Fragment implements ComponentCallbacks, OnCreateContextMenuListener, LifecycleOwner,
        ViewModelStoreOwner, HasDefaultViewModelProviderFactory, SavedStateRegistryOwner,
        ActivityResultCaller {

    LifecycleRegistry mLifecycleRegistry;

    private void initLifecycle() {
        mLifecycleRegistry = new LifecycleRegistry(this);
    }
    
    @Override
    public Lifecycle getLifecycle() {
        return mLifecycleRegistry;
    }    
}
        
public class ComponentActivity extends androidx.core.app.ComponentActivity implements
        ContextAware,
        LifecycleOwner,
        ViewModelStoreOwner,
        HasDefaultViewModelProviderFactory,
        SavedStateRegistryOwner,
        OnBackPressedDispatcherOwner,
        ActivityResultRegistryOwner,
        ActivityResultCaller {} 
```


2. 明确LifecycleRegistry主要作用

```java
/* 定义一个具有 Android 生命周期的对象 */
public abstract class Lifecycle {
    @MainThread
    public abstract void addObserver(@NonNull LifecycleObserver observer);
    @MainThread
    public abstract void removeObserver(@NonNull LifecycleObserver observer);
    // 生命周期事件,表明已被分发的生命周期事件,映射到Activity和Fragment中的回调事件 
    public enum Event {
        ON_CREATE,ON_START,ON_RESUME,ON_PAUSE,ON_STOP,ON_DESTROY,ON_ANY;
    }
    // 生命周期状态,表明组件的当前状态
    public enum State {
        DESTROYED,INITIALIZED,CREATED,STARTED,RESUMED;
    }
}


/* 可以处理多个Observer的Lifecycle实现,Fragment和Activity使用 */
public class LifecycleRegistry extends Lifecycle {

    private LifecycleRegistry(@NonNull LifecycleOwner provider, boolean enforceMainThread) {
        mLifecycleOwner = new WeakReference<>(provider);
        mState = INITIALIZED;
        mEnforceMainThread = enforceMainThread;
    }
}
``` 

3. 到这里我们就可以大致这么认为了, LifecycleOwner 通过获取当前对应的 LifecycleRegistry  管理多个LicecycleObserver,然后在生命周期状态发生变化时,处理不同状态事件的分放

#### Activity 是如何实现Jetpack Lifecycle事件分放的?

```kotlin
package androidx.core.app;

public class ComponentActivity extends Activity implements LifecycleOwner,KeyEventDispatcher.Component {

    private LifecycleRegistry mLifecycleRegistry = new LifecycleRegistry(this); //1
    
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        ReportFragment.injectIfNeededIn(this); // 3
    }
    
    @Override
    public Lifecycle getLifecycle() {
        return mLifecycleRegistry; // 2
    }

}
```

注意看ReportFragment:
```java
public class ReportFragment extends Fragment {
    private static final String REPORT_FRAGMENT_TAG = "androidx.lifecycle"
            + ".LifecycleDispatcher.report_fragment_tag";

    public static void injectIfNeededIn(Activity activity) {
        if (Build.VERSION.SDK_INT >= 29) {
         activity.registerActivityLifecycleCallbacks(new LifecycleCallbacks());
        }
       
        android.app.FragmentManager manager = activity.getFragmentManager();
        if (manager.findFragmentByTag(REPORT_FRAGMENT_TAG) == null) {
            manager.beginTransaction().add(new ReportFragment(), REPORT_FRAGMENT_TAG).commit();
            // Hopefully, we are the first to make a transaction.
            manager.executePendingTransactions();
        }
    }

    @SuppressWarnings("deprecation")
    static void dispatch(@NonNull Activity activity, @NonNull Lifecycle.Event event) {
        //已经被标注为@Deprecated
        if (activity instanceof LifecycleRegistryOwner) {
            ((LifecycleRegistryOwner) activity).getLifecycle().handleLifecycleEvent(event);
            return;
        }

        if (activity instanceof LifecycleOwner) {
            Lifecycle lifecycle = ((LifecycleOwner) activity).getLifecycle();
            if (lifecycle instanceof LifecycleRegistry) {
                ((LifecycleRegistry) lifecycle).handleLifecycleEvent(event);
            }
        }
    }

    static ReportFragment get(Activity activity) {
        return (ReportFragment) activity.getFragmentManager().findFragmentByTag(
                REPORT_FRAGMENT_TAG);
    }

    @Override
    public void onActivityCreated(Bundle savedInstanceState) {
        super.onActivityCreated(savedInstanceState);
        dispatchCreate(mProcessListener);
        dispatch(Lifecycle.Event.ON_CREATE);
    }

    @Override
    public void onStart() {
        super.onStart();
        dispatchStart(mProcessListener);
        dispatch(Lifecycle.Event.ON_START);
    }

    @Override
    public void onResume() {
        super.onResume();
        dispatchResume(mProcessListener);
        dispatch(Lifecycle.Event.ON_RESUME);
    }

    @Override
    public void onPause() {
        super.onPause();
        dispatch(Lifecycle.Event.ON_PAUSE);
    }

    @Override
    public void onStop() {
        super.onStop();
        dispatch(Lifecycle.Event.ON_STOP);
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        dispatch(Lifecycle.Event.ON_DESTROY);
        // just want to be sure that we won't leak reference to an activity
        mProcessListener = null;
    }

    private void dispatch(@NonNull Lifecycle.Event event) {
        if (Build.VERSION.SDK_INT < 29) {
            dispatch(getActivity(), event);
        }
    }

    //API29及以上直接使用Application.ActivityLifecycleCallbacks来监听生命周期
    static class LifecycleCallbacks implements Application.ActivityLifecycleCallbacks {
        @Override
        public void onActivityCreated(@NonNull Activity activity,
                @Nullable Bundle bundle) {
        }

        @Override
        public void onActivityPostCreated(@NonNull Activity activity,
                @Nullable Bundle savedInstanceState) {
            dispatch(activity, Lifecycle.Event.ON_CREATE);
        }

        @Override
        public void onActivityStarted(@NonNull Activity activity) {
        }

        @Override
        public void onActivityPostStarted(@NonNull Activity activity) {
            dispatch(activity, Lifecycle.Event.ON_START);
        }

        @Override
        public void onActivityResumed(@NonNull Activity activity) {
        }

        @Override
        public void onActivityPostResumed(@NonNull Activity activity) {
            dispatch(activity, Lifecycle.Event.ON_RESUME);
        }

        @Override
        public void onActivityPrePaused(@NonNull Activity activity) {
            dispatch(activity, Lifecycle.Event.ON_PAUSE);
        }

        @Override
        public void onActivityPaused(@NonNull Activity activity) {
        }

        @Override
        public void onActivityPreStopped(@NonNull Activity activity) {
            dispatch(activity, Lifecycle.Event.ON_STOP);
        }

        @Override
        public void onActivityStopped(@NonNull Activity activity) {
        }

        @Override
        public void onActivitySaveInstanceState(@NonNull Activity activity,
                @NonNull Bundle bundle) {
        }

        @Override
        public void onActivityPreDestroyed(@NonNull Activity activity) {
            dispatch(activity, Lifecycle.Event.ON_DESTROY);
        }

        @Override
        public void onActivityDestroyed(@NonNull Activity activity) {
        }
    }
}
```

通过一个透明的Fragment来分发生命周期事件，这样对于Activity来说是无侵入的。分成两部分逻辑：
- 当API>=29时，直接使用Application.ActivityLifecycleCallbacks来分发生命周期事件
- 而当API<29时，在Fragment的生命周期回调中进行了事件分发。
但殊途同归，两者最终都会走到`dispatch(Activity activity, Lifecycle.Event event)`方法


#### LifecycleObserver 是如何被响应的?

```java
/* 1 */
public class ReportFragment{
    // 事件分发
    static void dispatch(@NonNull Activity activity, @NonNull Lifecycle.Event event) {
        if (activity instanceof LifecycleRegistryOwner) {
            ((LifecycleRegistryOwner) activity).getLifecycle().handleLifecycleEvent(event);
            return;
        }

        if (activity instanceof LifecycleOwner) {
            Lifecycle lifecycle = ((LifecycleOwner) activity).getLifecycle();
            if (lifecycle instanceof LifecycleRegistry) {
                ((LifecycleRegistry) lifecycle).handleLifecycleEvent(event);
            }
        }
    }
}
/* 2 */
public class LifecycleRegistry extends Lifecycle {
    // 2.1-处理生命周期事件
    public void handleLifecycleEvent(@NonNull Lifecycle.Event event) {
        enforceMainThreadIfNeeded("handleLifecycleEvent");
        moveToState(event.getTargetState());
    }
    // 2.2-判断生命周期状态
    private void moveToState(State next) {
        if (mState == next) {
            return;
        }
        mState = next;
        if (mHandlingEvent || mAddingObserverCounter != 0) {
            mNewEventOccurred = true;
            // we will figure out what to do on upper level.
            return;
        }
        mHandlingEvent = true;
        sync();
        mHandlingEvent = false;
    }
    // 2-3 同步状态至LifecycleObserver
    private void sync() {
        LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
        if (lifecycleOwner == null) {
            throw new IllegalStateException("LifecycleOwner of this LifecycleRegistry is already"
                    + "garbage collected. It is too late to change lifecycle state.");
        }
        while (!isSynced()) {
            mNewEventOccurred = false;
            // no need to check eldest for nullability, because isSynced does it for us.
            if (mState.compareTo(mObserverMap.eldest().getValue().mState) < 0) {
                backwardPass(lifecycleOwner);
            }
            Entry<LifecycleObserver, ObserverWithState> newest = mObserverMap.newest();
            if (!mNewEventOccurred && newest != null
                    && mState.compareTo(newest.getValue().mState) > 0) {
                forwardPass(lifecycleOwner);
            }
        }
        mNewEventOccurred = false;
    }
    
    private void forwardPass(LifecycleOwner lifecycleOwner) {
        Iterator<Entry<LifecycleObserver, ObserverWithState>> ascendingIterator =
                mObserverMap.iteratorWithAdditions();
        while (ascendingIterator.hasNext() && !mNewEventOccurred) {
            Entry<LifecycleObserver, ObserverWithState> entry = ascendingIterator.next();
            ObserverWithState observer = entry.getValue();
            while ((observer.mState.compareTo(mState) < 0 && !mNewEventOccurred
                    && mObserverMap.contains(entry.getKey()))) {
                pushParentState(observer.mState);
                final Event event = Event.upFrom(observer.mState);
                if (event == null) {
                    throw new IllegalStateException("no event up from " + observer.mState);
                }
                observer.dispatchEvent(lifecycleOwner, event);
                popParentState();
            }
        }
    }

    private void backwardPass(LifecycleOwner lifecycleOwner) {
        Iterator<Entry<LifecycleObserver, ObserverWithState>> descendingIterator =
                mObserverMap.descendingIterator();
        while (descendingIterator.hasNext() && !mNewEventOccurred) {
            Entry<LifecycleObserver, ObserverWithState> entry = descendingIterator.next();
            ObserverWithState observer = entry.getValue();
            while ((observer.mState.compareTo(mState) > 0 && !mNewEventOccurred
                    && mObserverMap.contains(entry.getKey()))) {
                Event event = Event.downFrom(observer.mState);
                if (event == null) {
                    throw new IllegalStateException("no event down from " + observer.mState);
                }
                pushParentState(event.getTargetState());
                observer.dispatchEvent(lifecycleOwner, event);
                popParentState();
            }
        }
    }
        
}
```


### Refer

[Analysis of the basic use and rationale of Android Jetpack component Lifecycle - Moment For Technology (mo4tech.com)](https://www.mo4tech.com/analysis-of-the-basic-use-and-rationale-of-android-jetpack-component-lifecycle.html)

[Android Jetpack系列之Lifecycle_小马快跑的博客-CSDN博客_android中lifecycle](https://blog.csdn.net/u013700502/article/details/118469311)
