---
title: 源码学习 - ViewModel
tags:
  - 源码解析
categories: Android
updated: 1638199831000
thumbnail: 'https://s4.ax1x.com/2022/03/02/b3GOSg.md.png'
date: 2021-11-29 23:30:53
---
> ViewModel 类旨在以注重生命周期的方式存储和管理界面相关的数据。ViewModel 类让数据可在发生屏幕旋转等配置更改后继续留存,将视图和数据进行了分离解耦，为视图层提供数据

<!--more-->
### 配置变更保存数据的方式

- onSaveInstance(Bundle)
- ViewModel

### ViewModel优势

- ViewModel是将数据存到内存中，而onSaveInstance()是通过Bundle将序列化数据存在磁盘中
- ViewModel可以存储任何形式的数据，且大小不限制(不超过App分配的内存即可)，onSaveInstance()中只能存储可序列化的数据，且大小一般不超过1M（IPC通信数据限制）

### ViewModel生命周期

![I8L0jU.png](https://z3.ax1x.com/2021/11/08/I8L0jU.png)

ViewModel对象存在的时间范围是获取 ViewModel 时传递给 ViewModelProvider 的Lifecycle。ViewModel将一直留在内存中，直到限定其存在时间范围的Lifecycle永久消失：
对于activity，是在activity销毁时；
对于fragment，是在 fragment分离时


### 解析

#### ViewModel是怎么样实例化的?

```kotlin
abstract class BindingActivity<T : ViewDataBinding> constructor(
    @LayoutRes private val layoutId: Int
) : AppCompatActivity(), HasAndroidInjector {
    
    protected open fun <T : ViewModel?> getActivityScopeViewModel(modelClass: Class<T>): T {
        // 实例化对应Scope的ViewModelProvider
        if (!::activityProvider.isInitialized) {
            activityProvider = ViewModelProvider(requireActivity())
        }
        // 获取ViewModelProvider获取Viewmodel
        return activityProvider.get(modelClass)
    }
}
```

#### ViewModelProvider又是啥?

> ViewModel的辅助程序类，该类负责为界面准备数据。在配置更改期间会自动保留 ViewModel 对象，以便它们存储的数据立即可供下一个 activity 或 fragment 实例使用

```java
/* 为Fragment/Activity提供ViewModels的实用程序类 */
public class ViewModelProvider {
    // 构造方法
    // 1.创建用来存储ViewModel的ViewModelStore
    // 2. 创建用于实例化新ViewModel的Factory
    public ViewModelProvider(@NonNull ViewModelStoreOwner owner) {
        this(owner.getViewModelStore(), owner instanceof HasDefaultViewModelProviderFactory
                ? ((HasDefaultViewModelProviderFactory) owner).getDefaultViewModelProviderFactory()
                : NewInstanceFactory.getInstance());
    }
    // 构造方法
    public ViewModelProvider(@NonNull ViewModelStoreOwner owner, @NonNull Factory factory) {
        this(owner.getViewModelStore(), factory);
    }
    // 构造方法
    public ViewModelProvider(@NonNull ViewModelStore store, @NonNull Factory factory) {
        mFactory = factory;
        mViewModelStore = store;
    }
  
}
```

ViewModelProvider 关键参数组成:
-   ViewModelStoreOwner：ViewModel存储器拥有者，用来提供ViewModelStore
-   ViewModelStore：ViewModel存储器，用来存储ViewModel
-   ViewModelProviderFactory：创建ViewModel的工厂


首先尝试通过ViewModelStore.get(key)获取ViewModel，如果不为空直接返回该实例；如果为空，通过Factory.create创建ViewModel并保存到ViewModelStore中。先来看Factory是如何创建ViewModel的，ViewModelProvider构造函数中，如果没有传入Factory，那么会使用NewInstanceFactory

#### ViewModelProvider中的Owner和Factory怎么来的?

```java
public class ComponentActivity extends androidx.core.app.ComponentActivity implements
        ViewModelStoreOwner,
        HasDefaultViewModelProviderFactory{
}
```

##### ViewModelProviderFactory

> Activity,Fragment默认实现了HasDefaultViewModelProviderFactory接口,实现自己创建ViewModel的ViewModelProviderFactory

![IGEOT1.png](https://z3.ax1x.com/2021/11/08/IGEOT1.png)

再来看看`ComponentActivity.getViewModelStore()`:
```java
public class ComponentActivity extends androidx.core.app.ComponentActivity implements
        ViewModelStoreOwner,
        HasDefaultViewModelProviderFactory{
            
    /* 返回未向ViewModelProvider构造函数提供自定义Factory时应使用的默认ViewModelProvider.Factory  */
    public ViewModelProvider.Factory getDefaultViewModelProviderFactory() {
        if (getApplication() == null) {
            throw new IllegalStateException("Your activity is not yet attached to the "
                    + "Application instance. You can't request ViewModel before onCreate call.");
        }
        // 妈的,就是这么简单,直接实例化
        if (mDefaultFactory == null) {
            mDefaultFactory = new SavedStateViewModelFactory(
                    getApplication(),
                    this,
                    getIntent() != null ? getIntent().getExtras() : null);
        }
        return mDefaultFactory;
    }
}    
```

##### ViewModelStore

>  Activity,Fragment也默认实现了这个接口,以来获取跟当前生命周期相关的ViewModelStore,看到没有,有图有真相

![I8xIk4.png](https://z3.ax1x.com/2021/11/08/I8xIk4.png)

看看`ComponentActivity.getViewModelStore()`:

```java
    public ViewModelStore getViewModelStore() {
        if (getApplication() == null) {
            throw new IllegalStateException("Your activity is not yet attached to the "
                    + "Application instance. You can't request ViewModel before onCreate call.");
        }
        // 叉,这里照我猜想肯定是实现ViewModelStore的单例
        ensureViewModelStore();
        // 返回与此Activity关联的ViewModelStore
        return mViewModelStore;
    }
    
    ```
    void ensureViewModelStore() {
        if (mViewModelStore == null) {
            // 检索先前由onRetainNonConfigurationInstance()返回的配置变更后的缓存配置
            NonConfigurationInstances nc = (NonConfigurationInstances) getLastNonConfigurationInstance();
            // 如果缓存配置不为空,则取缓存配置的viewModelStore
            if (nc != null) {
                mViewModelStore = nc.viewModelStore;
            }
            // 否则自己实例一个
            if (mViewModelStore == null) {
                mViewModelStore = new ViewModelStore();
            }
        }
    }    
```

**通过getLastNonConfigurationInstance()我们可以在知道 **
ViewModelStore的存取都是间接在ActivityThread中进行并保存在ActivityClientRecord中。在Activity配置变化时，ViewModelStore可以在Activity销毁时得以保存并在重建时重新从lastNonConfigurationInstances中获取，又因为ViewModelStore提供了ViewModel，所以ViewModel也可以在Activity配置变化时得以保存，这也是为什么ViewModel的生命周期比Activity生命周期长的原因了。

#### 最后ViewModelProvider是如何get到 Viewmodel的?

```java
    // 第一步:小get
    public <T extends ViewModel> T get(@NonNull Class<T> modelClass) {
        String canonicalName = modelClass.getCanonicalName();
        // 好家伙,看到没有,如果这里判断是局部类或者匿名类,就直接给crash了,谷歌就是牛逼
        if (canonicalName == null) {
            throw new IllegalArgumentException("Local and anonymous classes can not be ViewModels");
        }
        // 首先构造了一个key，直接调用下面的get(key,modelClass)
        return get(DEFAULT_KEY + ":" + canonicalName, modelClass);
    }
    // 第二步:大get
    public <T extends ViewModel> T get(@NonNull String key, @NonNull Class<T> modelClass) {
        ViewModel viewModel = mViewModelStore.get(key);
        // 尝试从ViewModelStore中获取ViewModel
        if (modelClass.isInstance(viewModel)) {
            if (mFactory instanceof OnRequeryFactory) {
                ((OnRequeryFactory) mFactory).onRequery(viewModel);
            }
            // viewModel不为空直接返回该实例
            return (T) viewModel;
        }
        // 然后如果为null,则通过具体工厂类去实例化ViewModel
        if (mFactory instanceof KeyedFactory) {
            viewModel = ((KeyedFactory) mFactory).create(key, modelClass);
        } else {
            viewModel = mFactory.create(modelClass);
        }
        // 嘿嘿,放进缓存
        mViewModelStore.put(key, viewModel);
        return (T) viewModel;
    } 
    // ViewModel的实例化
    public <T extends ViewModel> T create(@NonNull String key, @NonNull Class<T> modelClass) {
        boolean isAndroidViewModel = AndroidViewModel.class.isAssignableFrom(modelClass);
        Constructor<T> constructor;
        // 通过反射,去找到当前ViewModel的构造函数
        if (isAndroidViewModel && mApplication != null) {
            constructor = findMatchingConstructor(modelClass, ANDROID_VIEWMODEL_SIGNATURE);
        } else {
            constructor = findMatchingConstructor(modelClass, VIEWMODEL_SIGNATURE);
        }
        if (constructor == null) {
            return mFactory.create(modelClass);
        }
        SavedStateHandleController controller = SavedStateHandleController.create(
                mSavedStateRegistry, mLifecycle, key, mDefaultArgs);
        // 嘿嘿,调用构造函数,实例化        
        try {
            T viewmodel;
            if (isAndroidViewModel && mApplication != null) {
                viewmodel = constructor.newInstance(mApplication, controller.getHandle());
            } else {
                viewmodel = constructor.newInstance(controller.getHandle());
            }
            viewmodel.setTagIfAbsent(TAG_SAVED_STATE_HANDLE_CONTROLLER, controller);
            return viewmodel;
        } catch (IllegalAccessException e) {
            throw new RuntimeException("Failed to access " + modelClass, e);
        } catch (InstantiationException e) {
            throw new RuntimeException("A " + modelClass + " cannot be instantiated.", e);
        } catch (InvocationTargetException e) {
            throw new RuntimeException("An exception happened in constructor of "
                    + modelClass, e.getCause());
        }
    }   
```




#### ViewModelStore 是是如何存储ViewModel的?

嘿嘿,存储ViewModel的ViewModelStore,牛逼啊,用一个HashMap来缓存看到有木有:

![viewmodelcreate.png](https://img-blog.csdnimg.cn/img_convert/61bfe8240cd5d1648037da8bdc77ea8e.png)

```java
public class ViewModelStore {
    private final HashMap<String, ViewModel> mMap = new HashMap<>();

    final void put(String key, ViewModel viewModel) {
        ViewModel oldViewModel = mMap.put(key, viewModel);
        if (oldViewModel != null) oldViewModel.onCleared();
    }

    final ViewModel get(String key) {
        return mMap.get(key);
    }

    Set<String> keys() {
        return new HashSet<>(mMap.keySet());
    }

    public final void clear() {
        for (ViewModel vm : mMap.values()) {
            vm.clear();
        }
        mMap.clear();
    }
}
```
![IG8UgA.png](https://z3.ax1x.com/2021/11/08/IG8UgA.png)




### ViewModel意义

![IGd3ff.png](https://z3.ax1x.com/2021/11/08/IGd3ff.png)

![IGwfKg.png](https://z3.ax1x.com/2021/11/08/IGwfKg.png)