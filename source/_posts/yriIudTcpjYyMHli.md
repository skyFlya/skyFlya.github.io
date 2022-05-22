---
title: Android黑科技 - 插件化
tags:
  - 黑科技
updated: 1646663918000
categories: Android
thumbnail: 'https://s1.ax1x.com/2022/03/07/b60yFg.md.png'
date: 2022-03-07 22:46:11
---
插件化技术最初源于免安装运行apk的想法，这个免安装的apk可以理解为插件。支持插件化的app可以在运行时加载和运行插件，这样便可以将app中一些不常用的功能模块做成插件，一方面减小了安装包的大小，另一方面可以实现app功能的动态扩展。

想要实现插件化，主要是解决下面三个问题：
-   插件中代码的加载和与主工程的互相调用
-   插件中资源的加载和与主工程的互相访问
-   四大组件生命周期的管理
<!--more-->
### 插件化发展

![byn21s.png](https://s1.ax1x.com/2022/03/07/byn21s.png)


**第一代**：dynamic-load-apk最早使用ProxyActivity这种静态代理技术，由ProxyActivity去控制插件中PluginActivity的生命周期。该种方式缺点明显，插件中的activity必须继承PluginActivity，开发时要小心处理context。而DroidPlugin通过Hook系统服务的方式启动插件中的Activity，使得开发插件的过程和开发普通的app没有什么区别，但是由于hook过多系统服务，异常复杂且不够稳定。

**第二代**：为了同时达到插件开发的低侵入性（像开发普通app一样开发插件）和框架的稳定性，在实现原理上都是趋近于选择尽量少的hook，并通过在manifest中预埋一些组件实现对四大组件的插件化。另外各个框架根据其设计思想都做了不同程度的扩展，其中Small更是做成了一个跨平台，组件化的开发框架。

**第三代**：VirtualApp比较厉害，能够完全模拟app的运行环境，能够实现app的免安装运行和双开技术。Atlas是阿里今年开源出来的一个结合组件化和热修复技术的一个app基础框架，其广泛的应用与阿里系的各个app，其号称是一个容器化框架


### 插件化方案

#### 静态方案-通过ProxyActivity统一加载插件中的所有Activity，以that框架为代表

> ProxyActivity代理的方式最早是由dynamic-load-apk提出的，其思想很简单，在主工程中放一个ProxyActivy，启动插件中的Activity时会先启动ProxyActivity，在ProxyActivity中创建插件Activity，并同步生命周期

![byKLex.png](https://s1.ax1x.com/2022/03/07/byKLex.png)

##### 流程:
1.  首先需要通过统一的入口（如图中的PluginManager）启动插件Activity，其内部会将启动的插件Activity信息保存下来，并将intent替换为启动ProxyActivity的intent。
2.  ProxyActivity根据插件的信息拿到该插件的ClassLoader和Resource，通过反射创建PluginActivity并调用其onCreate方法。
3.  PluginActivty调用的setContentView被重写了，会去调用ProxyActivty的setContentView。由于ProxyActivity重写了getResource返回的是插件的Resource，所以setContentView能够访问到插件中的资源。同样findViewById也是调用ProxyActivity的。
4.  ProxyActivity中的其他生命周期回调函数中调用相应PluginActivity的生命周期

##### 实现
模拟Activty基本实现:
```java
public interface ProxyActivityInterface {

    //生命周期的activity

    public void attach(Activity proxyActivity);


    public void onCreate(Bundle savedInstanceState);

    public void onStart();

    public void onResume();

    public void onPause();

    public void onStop();

    public void onDestroy();

    public void onSaveInstanceState(Bundle outState);

    public boolean onTouchEvent(MotionEvent event);

    public void onBackPressed();
}
```
插件BaseActivty类:
```java
// 这是插件的基类，所有的activity都要继承这个类，
public class BaseActivity extends Activity implements ProxyActivityInterface {
    public Activity that;//这里的that 指的是我们的宿主app，因为插件是没有安装的 是没有上下文的

    @Override
    public void attach(Activity proxyActivity) {
        that = proxyActivity;
    }

    @Override
    public void setContentView(View view) {
        //可以看到,最终调用主App(宿主)的activity 函数
        if (that != null) {
            that.setContentView(view);
        } else {
            super.setContentView(view);
        }
    }

    @Override
    public void setContentView(int layoutResID) {
        that.setContentView(layoutResID);
    }

    @Override
    public View findViewById(int id) {
        return that.findViewById(id);
    }

    @Override
    public Intent getIntent() {
        if (that != null) {
            return that.getIntent();
        }
        return super.getIntent();
    }

    @Override
    public ClassLoader getClassLoader() {
        return that.getClassLoader();
    }

    @NonNull
    @Override
    public LayoutInflater getLayoutInflater() {
        return that.getLayoutInflater();
    }

    @Override
    public ApplicationInfo getApplicationInfo() {
        return that.getApplicationInfo();
    }

    @Override
    public Window getWindow() {
        return that.getWindow();
    }


    @Override
    public WindowManager getWindowManager() {
        return that.getWindowManager();
    }

    @Override
    public void startActivity(Intent intent) {
//        ProxyActivity --->className
        Intent m = new Intent();
        m.putExtra("ClassName", intent.getComponent().getClassName());
        that.startActivity(m);
    }

    @Override
    public void onCreate(Bundle savedInstanceState) {}

    @Override
    public void onStart() {}

    @Override
    public void onResume() {}

    @Override
    public void onPause() {}

    @Override
    public void onStop() {}

    @Override
    public void onDestroy() {}

    @Override
    public void onSaveInstanceState(Bundle outState) {}
```
插件ProxyActivity(): 主要通过这个去 New 一个Activity 来模拟原生Activity管理

> 插件内部的跳转其实也就是在开同一个activity

```java
public class ProxyActivity extends AppCompatActivity {

    private ProxyActivityInterface pluginObj;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        //在这里拿到了真实跳转的activity 拿出来 再去启动真实的activity

        String className = getIntent().getStringExtra("ClassName");

        //通过反射在去启动一个真实的activity 拿到Class对象
        try {
            Class<?> plugClass = getClassLoader().loadClass(className);
            Constructor<?> pluginConstructor = plugClass.getConstructor(new Class[]{});
            //因为插件的activity实现了我们的标准
            pluginObj = (ProxyActivityInterface) pluginConstructor.newInstance(new Object[]{});
            pluginObj.attach(this);//注入上下文
            pluginObj.onCreate(new Bundle());//一定要调用onCreate 
        } catch (Exception e) {
            if (e.getClass().getSimpleName() .equals("ClassCastException")){
                //我这里是直接拿到异常判断的 ，也可的 拿到上面的plugClass对象判断有没有实现我们的接口
                finish();
                Toast.makeText(this,"非法页面",Toast.LENGTH_LONG).show();
                return;
            }
            e.printStackTrace();
        }
    }
]
    //为什么要重写这个呢 因为这个是插件内部startactivity调用的 将真正要开启的activity的类名穿过来
    //然后取出来，启动我们的占坑的activity 在我们真正要启动的赛进去
    @Override
    public void startActivity(Intent intent) {
        String className1=intent.getStringExtra("ClassName");
        Intent intent1 = new Intent(this, ProxyActivity.class);
        intent1.putExtra("ClassName", className1);
        super.startActivity(intent1);
    }

    //重写classLoader
    @Override
    public ClassLoader getClassLoader() {
        return HookManager.getInstance().getClassLoader();
    }

    //重写Resource
    @Override
    public Resources getResources() {
        return HookManager.getInstance().getResource();
    }

    @Override
    protected void onStart() {
        super.onStart();
        pluginObj.onStart();
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        pluginObj.onDestroy();
    }

    @Override
    protected void onPause() {
        super.onPause();
        pluginObj.onPause();
    }
}
```
插件apk和资源加载处理-Hookmanager:
```java
public class HookManager {
    private static final HookManager ourInstance = new HookManager();
    private Resources resources;
    private DexClassLoader loader;
    public PackageInfo packageInfo;

    public static HookManager getInstance() {
        return ourInstance;
    }

    private HookManager() {}

    // 复制插件到缓存目录,方便加载
    public void loadPlugin(Activity activity) {
        // 假如这里是从网络获取的插件 我们直接从sd卡获取 然后读取到我们的cache目录
        String pluginName = "plugin.apk";
        File filesDir = activity.getDir("plugin", activity.MODE_PRIVATE);
        String filePath = new File(filesDir, pluginName).getAbsolutePath();
        File file = new File(filePath);
        if (file.exists()) {
            file.delete();
        }
        FileInputStream is = null;
        FileOutputStream os = null;
        //读取的目录
        try {
            is = new FileInputStream(new File(Environment.getExternalStorageDirectory(), pluginName));
            //要输入的目录
            os = new FileOutputStream(filePath);
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        }
        try {
            int len = 0;
            byte[] buffer = new byte[1024];
            while ((len = is.read(buffer)) != -1) {
                os.write(buffer, 0, len);
            }
            File f = new File(filePath);
            if (f.exists()) {
                Toast.makeText(activity, "dex overwrite", Toast.LENGTH_SHORT).show();
            }
            loadPathToPlugin(activity);
        } catch (IOException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        } finally {
            try {
                os.close();
                is.close();
            } catch (IOException e) {
                // TODO Auto-generated catch block
                e.printStackTrace();
            }
        }


    }

    private void loadPathToPlugin(Activity activity) {
        File filesDir = activity.getDir("plugin", activity.MODE_PRIVATE);
        String name = "plugin.apk";
        String path = new File(filesDir, name).getAbsolutePath();

        //然后我们开始加载我们的apk 使用DexClassLoader
        File dexOutDir = activity.getDir("dex", activity.MODE_PRIVATE);
        loader = new DexClassLoader(path, dexOutDir.getAbsolutePath(), null, activity.getClassLoader());

        //通过PackAgemanager 来获取插件的第一个activity是哪一个
        PackageManager packageManager = activity.getPackageManager();
        packageInfo = packageManager.getPackageArchiveInfo(path, PackageManager.GET_ACTIVITIES);


        //然后开始加载我们的资源 肯定要使用Resource 但是它是AssetManager创建出来的 就是AssertManager 有一个addAssertPath 这个方法 但是私有的 所有使用反射
        Class<?> assetManagerClass = AssetManager.class;
        try {
            AssetManager assertManagerObj = (AssetManager) assetManagerClass.newInstance();
            Method addAssetPathMethod = assetManagerClass.getMethod("addAssetPath", String.class);
            addAssetPathMethod.setAccessible(true);
            addAssetPathMethod.invoke(assertManagerObj, path);
            //在创建一个Resource
            resources = new Resources(assertManagerObj, activity.getResources().getDisplayMetrics(), activity.getResources().getConfiguration());
        } catch (Exception e) {
            e.printStackTrace();
        }

    }
    //对外提供插件的classLoader
    public ClassLoader getClassLoader() {
        return loader;
    }

    //插件中的Resource
    public Resources getResource() {
        return resources;
    }
}
```
现在主App可以实现插件化功能了
```java
// 加载对应插件
HookManager.getInstance().loadPlugin(this)
// 通过跳到ProxyActivity 来模拟实现对应插件Activity
Intent intent = new Intent(this, ProxyActivity.class);//这里就是一个占坑的activity
//这里是拿到我们加载的插件的第一个activity的全类名
intent.putExtra("ClassName",HookManager.getInstance().packageInfo.activities[0].name);
startActivity(intent);
```
通过这种代理模拟方式也可以实现 Service、BroadcastReceiver、ContentProvider组件的代理

##### 问题

注意事项：
-   ProxyActivity中需要重写getResouces，getAssets，getClassLoader方法返回插件的相应对象。生命周期函数以及和用户交互相关函数，如onResume，onStop，onBackPressedon，KeyUponWindow，FocusChanged等需要转发给插件。
-   PluginActivity中所有调用context的相关的方法，如setContentView，getLayoutInflater，getSystemService等都需要调用ProxyActivity的相应方法。

缺点
-   插件中的Activity必须继承PluginActivity，开发侵入性强。
-   如果想支持Activity的singleTask，singleInstance等launchMode时，需要自己管理Activity栈，实现起来很繁琐。
-   插件中需要小心处理Context，容易出错。
-   如果想把之前的模块改造成插件需要很多额外的工作。

该方式虽然能够很好的实现启动插件Activity的目的，但是由于开发式侵入性很强，dynamic-load-apk之后的插件化方案很少继续使用该方式，而是通过hook系统启动Activity的过程，让启动插件中的Activity像启动主工程的Activity一样简单。



#### 动态替换方案：提供对Android底层的各种类进行Hook，来实现加载插件中的四大组件，以DroidPlugin框架为代表

实现方案:
- Hook Instrument实现
- Hook Handler实现

代码Demo:[Onion99/AndroidComponentPlugin: Android上简单实现四大组件的插件化](https://github.com/Onion99/AndroidComponentPlugin)
参考 : [Android Hook告诉你 如何启动未注册的Activity](https://developer.aliyun.com/article/873723?spm=a2c6h.12883283.0.0.67d543078VqvHa&scm=20140722.ID_873723.P_121.MO_938-ST_5186-V_1-ID_873723-OR_rec)

### Refer

[Android插件化(一))](https://www.jianshu.com/p/71585d744076)
[深入理解Android插件化技术 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/33017826)
[Android插件化实现方案_安卓插件化](https://blog.csdn.net/n_fly/article/details/113865650)