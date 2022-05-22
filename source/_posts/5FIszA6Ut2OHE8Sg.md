---
title: 安卓优化 - 布局优化
tags:
  - 性能优化
categories: Android
updated: 1635684594000
date: 2021-10-31 20:50:14
---

### 布局耗时统计

- 手动埋点
- AOP/ArtHook	
	+ 切入Activity的setContentView

### 布局加载优化

- 代码写布局
	+  Java代码写布局
	+  Anko
	+  JetpackCompose
- X2C
- Litho

<!-- more -->
### 视图绘制优化

- 优化布局层级以及嵌套
	+ 使用ConstraintLayout
- 避免过度绘制,自定义View避免多次调用 onDraw,onMeasure
- 其他
	+  ViewStub: 延迟初始化
	+  onDraw,onMeasure中避免创建大对象,耗时操作


#### Include

> 提高布局复用性

login.xml
```xml
    <include
        android:layout_width="match_parent"
        android:layout_height="40dp"
        layout="@layout/titlebar" />
```

include所在的layout的布局有给其设置id, 而include标签里面又给自己的根容器设置id,最好两个id都相同,否则findview时拿到空对象

#### Merge

> 帮助include标签排除多余的一层ViewGroup容器，减少view hierarchy的结构，提升UI渲染的性能

titlebar.xml
```xml
<merge xmlns:android="http://schemas.android.com/apk/res/android">
     <Button 
        android:layout_width="match_parent"  
        android:layout_height="wrap_content"  
        android:layout_marginLeft="20dp"  
        android:layout_marginRight="20dp"  
        android:text="标题" />  
</merge>
```

- 因为merge标签并不是View,所以在通过LayoutInflate.inflate()方法渲染的时候,第二个参数必须指定一个父容器(parent),且第三个参数(attachToRoot)必须为true
- merge标签必须使用在根布局，并且ViewStub标签中的layout布局不能使用merge标签


#### ViewStub

> 延迟绘制View

layout.xml
```xml
    <ViewStub
        android:id="@+id/viewstub"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout="@layout/info" />
```

activity.java
```java
        ViewStub stub = ((ViewStub) findViewById(R.id.viewstub));
        if(stub!=null){
            View stubView = stub.inflate();
            EditText editText = (EditText) stubView.findViewById(R.id.edit_password);  
        }
```

- ViewStub标签不支持merge标签
- ViewStub的inflate只能被调用一次,第二次调用会抛出异常
- 虽然ViewStub是不占用任何空间的，但是每个布局都必须要指定layout_width和layout_height属性，否则运行就会报错