---
title: 源码学习 - DataBinding
tags:
  - 源码解析
updated: 1638199600000
categories: Android
date: 2021-11-29 23:27:07
---

### Sameple

首先,我们来看看DataBinding使用:
1. 给layout文件套娃:
```xml
<?xml version="1.0" encoding="utf-8"?>
<layout>
    <FrameLayout>
        <androidx.viewpager.widget.ViewPager
            android:id="@+id/home_pager"/>
        <com.flyco.tablayout.SlidingTabLayout
            android:id="@+id/home_tab"/>
    </FrameLayout>
</layout>
```
2. 在Fragment或者View中获取binding对象
```kotlin
binding = DataBindingUtil.inflate(inflater,layoutRes,container,false)
```
超,就是这么简单,来从`inflate`开始解析
<!-- more -->
### DataBindingUtil.inflate梳理

```java
    public static <T extends ViewDataBinding> T inflate(
            @NonNull LayoutInflater inflater, int layoutId, @Nullable ViewGroup parent,
            boolean attachToParent, @Nullable DataBindingComponent bindingComponent) {
        // 检查康康是不是子layout附加到父layout,一般是false,因为如果为ture会导致子layout的layoutparams 失效
        final boolean useChildren = parent != null && attachToParent;
        // 上面一般为false,那这里一般也是零蛋啊
        final int startChildren = useChildren ? parent.getChildCount() : 0;
        // 解析xml生成View,放空大脑就是这样,深究的话,我只能甩一篇郭神的文章了:https://blog.csdn.net/guolin_blog/article/details/12921889
        // 什么具体inflate是怎么样,不听不听王八念经
        final View view = inflater.inflate(layoutId, parent, attachToParent);
        // 嘿嘿,这里开始binding
        if (useChildren) {
            return bindToAddedViews(bindingComponent, parent, startChildren, layoutId);
        } else {
            return bind(bindingComponent, view, layoutId);
        }
    }
    ...
    private static DataBinderMapper sMapper = new DataBinderMapperImpl();
    static <T extends ViewDataBinding> T bind(DataBindingComponent bindingComponent, View root,int layoutId) {
        // 不理了,这里就拿到binding了,可以直接引用View了,停止思考
        return (T) sMapper.getDataBinder(bindingComponent, root, layoutId);
    }
```
问题来了,它是怎么取得我要的DataBinding对象且又是怎么实现View绑定的?

1. 好,来康康,这个`DataBinderMapperImpl`,这是编译阶段下kapt自动生成的:

```java
package androidx.databinding;
// 好家伙,这里直接继承MergedDataBinderMapper 
public class DataBinderMapperImpl extends MergedDataBinderMapper {
  DataBinderMapperImpl() {
    // 添加kapt下生成当前Module的DataBinderMapperImpl
    addMapper(new com.xxx.xxx.DataBinderMapperImpl());
  }
}
...
// 合并其他Mapper的DataBindingMapper 
public class MergedDataBinderMapper extends DataBinderMapper {
    private Set<Class<? extends DataBinderMapper>> mExistingMappers = new HashSet<>();
    // 添加Mapper,如果已经存在则忽略
    public void addMapper(DataBinderMapper mapper) {
        Class<? extends DataBinderMapper> mapperClass = mapper.getClass();
        if (mExistingMappers.add(mapperClass)) {
            mMappers.add(mapper);
            final List<DataBinderMapper> dependencies = mapper.collectDependencies();
            for(DataBinderMapper dependency : dependencies) {
                addMapper(dependency);
            }
        }
    }
}
```

2. `MergedDataBinderMapper.getDataBinder`:

```java
    @Override
    public ViewDataBinding getDataBinder(DataBindingComponent bindingComponent, View view,
            int layoutId) {
        // 遍历之前添加的mMappers    
        for(DataBinderMapper mapper : mMappers) {
            // 好了,这里获取对应的Mapper的DataBinder,超,想想之前添加了哪个Mapper
            // 没错,不正好对上tmd的上面1的com.xxx.xxx.DataBinderMapperImpl()
            ViewDataBinding result = mapper.getDataBinder(bindingComponent, view, layoutId);
            if (result != null) return result;
        }
        if (loadFeatures()) return getDataBinder(bindingComponent, view, layoutId);
        return null;
    }
```

3. 让我们来康康kapt生成的`com.xxx.xxx.DataBinderMapperImpl()`到底有什么

```java
public class DataBinderMapperImpl extends DataBinderMapper {
  // layout id 索引
  private static final int LAYOUT_FRAGMENTHOME = 40;
  private static final SparseIntArray INTERNAL_LAYOUT_ID_LOOKUP = new SparseIntArray(1);
  static {
    INTERNAL_LAYOUT_ID_LOOKUP.put(com.xmiles.callshow.R.layout.fragment_home, LAYOUT_FRAGMENTHOME);
  }
  @Override
  public ViewDataBinding getDataBinder(DataBindingComponent component, View view, int layoutId) {
    // 拿到索引的id
    int localizedLayoutId = INTERNAL_LAYOUT_ID_LOOKUP.get(layoutId);
    if(localizedLayoutId > 0) {
      final Object tag = view.getTag();
      // 判断tag
      if(tag == null)throw new RuntimeException("view must have a tag");
      // 这里如果布局数量超50,为避免一个方法体过大,会额外新建另外一个函数去处理,这里为好展示,整个类我就删除了很多其他布局参数
      int methodIndex = (localizedLayoutId - 1) / 50;
      // 跟据对应componet生成对应的ViewDetaBinding
      switch(methodIndex) {
        case 0: {
          return internalGetViewDataBinding0(component, view, localizedLayoutId, tag);
        }
        case 1: {
          return internalGetViewDataBinding1(component, view, localizedLayoutId, tag);
        }
      }
    }
    return null;
  }
  // GetViewDataBinding
  private final ViewDataBinding internalGetViewDataBinding0(DataBindingComponent component,View view, int internalId, Object tag) {
    switch(internalId) {
      case  LAYOUT_FRAGMENTHOME: {
        if ("layout/fragment_home_0".equals(tag)) {
          return new FragmentHomeBindingImpl(component, view);
        }
        throw new IllegalArgumentException("The tag for fragment_home is invalid. Received: " + tag);
      }
    }
    return null;
  }
}
```

### Refer
[Android架构组件之DataBinding源码解析 - 简书 (jianshu.com)](https://www.jianshu.com/p/4be20cc58f17)
[DataBinding源码分析 - 简书 (jianshu.com)](https://www.jianshu.com/p/4e9a1ab05bb5)