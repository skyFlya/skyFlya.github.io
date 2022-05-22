---
title: Android黑科技 - 热修复
tags:
  - 黑科技
updated: 1646663557000
categories: Android
thumbnail: 'https://s1.ax1x.com/2022/03/07/b6wtK0.md.png'
date: 2022-03-07 22:38:05
---

> 热修复本质就是将错误的代码替换成正确的代码,但这里的替换不是改写原有的代码,而是提供一份新的正确的代码,让应用运行时绕过错误的代码,从而执行正确的代码

![bDtcIx.png](https://s1.ax1x.com/2022/03/06/bDtcIx.png)
<!--more-->
基础知识:
- [热修复与插件化基础 - dex与class](https://fullstackaction.com/pages/ccf9c0/#%E4%B8%80%E3%80%81dex-class-%E6%B5%85%E6%9E%90)
- [热修复与插件化基础 - Java与Android虚拟机](https://fullstackaction.com/pages/1084c4/)
- [热修复与插件化基础 - Java与Android的类加载器](https://fullstackaction.com/pages/a5eb80/)

## 实现方案

### Native层替换方案
 
 > 底层替换，修改java方法在native层的函数指针，指向修复后的方法以达到修复目的

Android/Java代码的最小组织方式是方法（Method，实际上，每一个dex文件最多可以包含65536（0xffff）个方法），每个方法在ART虚拟机中都有一个ArtMethod结构体指针与之对应，ArtMethod结构体中包含了Java方法的所有信息，包括执行入口、访问权限、所属类和代码执行地址等等。换句话说，虚拟机就是通过ArtMethod结构体来操纵Java方法的。ArtMethod结构如下：

```c++
class ArtMethod FINAL {
...
protected:
  GcRoot<mirror::Class> declaring_class_;

  std::atomic<std::uint32_t> access_flags_;

  // Offset to the CodeItem.
  uint32_t dex_code_item_offset_;

  // Index into method_ids of the dex file associated with this method.
  uint32_t dex_method_index_;

  uint16_t method_index_;

  uint16_t hotness_count_;

  struct PtrSizedFields {
    // Depending on the method type, the data is
    //   - native method: pointer to the JNI function registered to this method
    //                    or a function to resolve the JNI function,
    //   - conflict method: ImtConflictTable,
    //   - abstract/interface method: the single-implementation if any,
    //   - proxy method: the original interface method or constructor,
    //   - other methods: the profiling data.
    void* data_;

    // Method dispatch from quick compiled code invokes this pointer which may cause bridging into
    // the interpreter.
    void* entry_point_from_quick_compiled_code_;
  } 
  ptr_sized_fields_;
...
｝
```

其中有一个关键指针，它是方法的执行入口：
`entry_point_from_quick_compiled_code_`
也就是说，这个指针指向方法体编译后对应的汇编指令。那么，如果我们能hook这个指针，由原来指向有bug的方法，变成指向正确的方法，就达到了修复的目的。这就是native层替换方案的核心原理。具体实现方案可以是改变指针指向（AndFix），也可以直接替换整个结构体（Sophix）。

需要注意的是，底层替换方案虽然是即使生效的，但是因为不会加载新类，而是直接修改原类，所以修改的代码不能增加新的方法，否则会造成索引数与方法数不匹配，无法通过索引找到正确方法，字段同理

### 类加载方案

> 加载一个类的时候，都会去循环dexElements数组取出里面的dex文件，然后从dex文件中找目标类，只要目标类找到，则直接退出循环，也就是后面的dex文件就没有被取到的机会。将热修复的类放在dexElements[]的最前面，这样加载类时 **会优先加载到要修复的类**以达到修复目的

![bDduwj.png](https://s1.ax1x.com/2022/03/06/bDduwj.png)

基于jvm的java应用是通过ClassLoader来加载应用中的class的，Android对JVM优化过，使用的是ART(以前是Dalvik)，class文件会被打包进dex文件中，底层虚拟机有所不同，那么它们的类加载器也会有所区别，在Android中，要加载dex文件中的class文件就需要用到 PathClassLoader 或 DexClassLoader 这两个Android专用的类加载器。


![bsEKiR.png](https://s1.ax1x.com/2022/03/07/bsEKiR.png)


#### 原理

![bsdsXT.png](https://s1.ax1x.com/2022/03/07/bsdsXT.png)

#### 生成修复Dex
  
1.将出bug的类修改正确，然后执行打包流程  [![bsNkSf.png](https://s1.ax1x.com/2022/03/07/bsNkSf.png)](https://imgtu.com/i/bsNkSf)
  
2.此时取出工程目录下的/build/intermediates/javac/debug/classes/包路径/文件夹下对应的class文件 
  
3.在复制这个class文件时，需要把它所在的完整包目录一起复制，然后在命令行下cd到该目录，执行`dx --dex --output=patch.dex 包名路径/需要修复的类文件`,此时会在当前目录下生成patch.dex文件  [![bsNQf0.png](https://s1.ax1x.com/2022/03/07/bsNQf0.png)](https://imgtu.com/i/bsNQf0)
  
4.然后将patch.dex文件当成补丁包放入资源文件夹raw下即可。

指令为：dx --dex --output=patch.dex com/xxx/xxx/fixbug.class -> 生成patch.dex文件  
  
也可以写成：dx --dex --output=patch.jar com/xxx/xxx/fixbug.class -> 生成patch.jar文件  
  
ClassLoader可以加载.dex文件，或者.zip、.jar、.apk中包含的.dex文件

#### 实践
 Application:
```java
/**
 * 之所以要在本类中做补丁包的安装是因为怕如果在后面的流程中做安装会造成有些带bug的类如果已经被系统加载的话，后续补丁包安装之后
 * 补丁包中的类得不到执行，因为类加载有缓存机制，系统会将加载过的类做一份内存缓存。
 */
public class MyApplication extends Application {
    @Override
    public void onCreate() {
        super.onCreate();
        // 此处为加载补丁包，其实真实应用场景是从服务端下载补丁文件，因为本工程为示例所以直接将其放在raw资源文件目录下去读取，省去了下载过程
        File patchFile = new File(getCacheDir().getAbsolutePath() + File.separator + "patch.dex");
        try {
            // 为了高版本Android访问外部存储需要分区等问题，因为此处仅做示例讲解所以就直接将包拷贝到私有目录中
            // 拷贝到私有目录中还有一个好处是避免在补丁包安装过程中包文件被删除造成安装失败
            copyFile(patchFile);
            // 安装补丁包
            HotFix.installPatch(this, patchFile);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private void copyFile(File dest) throws IOException {
        InputStream input = null;
        OutputStream output = null;
        try {
            // 由于本项目是测试功能，所以这里直接将补丁包放在工程内，真实开发环境中应该是放在服务器然后下载下来
            input = getResources().openRawResource(R.raw.patch);
            output = new FileOutputStream(dest);
            byte[] buf = new byte[1024];
            int bytesRead;
            while ((bytesRead = input.read(buf)) != -1) {
                output.write(buf, 0, bytesRead);
            }
        } finally {
            input.close();
            output.close();
        }
    }
}
```

HotFix.installPatch:
```java
public class HotFix {
    public static void installPatch(Application application, File patch) {
        if (patch == null || !patch.exists()) {
            return;
        }
        try {
            /**
             * 1.获取当前的类加载器z
             */
            ClassLoader classLoader = application.getClassLoader();

            /**
             * 2.获取到dexElements属性以便后续向前追加patch.dex
             */
            Field pathListField = ReflectUtils.findField(classLoader, "pathList");
            Object pathList = pathListField.get(classLoader);
            Field dexElementsField = ReflectUtils.findField(pathList, "dexElements");
            Object[] dexElements = (Object[]) dexElementsField.get(pathList);

            /**
             * 3.通过反射调用DexPathList类中的makePathElements()方法将patch.dex最终转换为Element[]数组，
             * DexPathList一系列方法都是用来将补丁包转换为Element[]数组的，如makePathElements，makeDexElements..
             * 具体的API根据真实API的版本不同方法参数等可能会有出入，所以这里在使用过程中实际上应该通过判断去兼容各个版本，
             * 此处因为是示例所以没做兼容
             */
            List<File> files = new ArrayList<>();
            files.add(patch);
            Method method = ReflectUtils.findMethod(pathList, "makePathElements", List.class, File.class, List.class);
            ArrayList<IOException> suppressedExceptions = new ArrayList<>();
            Object[] patchElements = (Object[]) method.invoke(pathList, files, application.getCacheDir(), suppressedExceptions);

            /**
             * 4.合并patchElements+dexElements,将补丁包的.dex文件插入数组最前面，后续在加载类的时候会优先从第一个开始遍历查找类
             */
            Object[] newElements = (Object[]) Array.newInstance(dexElements.getClass().getComponentType(), dexElements.length + patchElements.length);
            System.arraycopy(patchElements, 0, newElements, 0, patchElements.length);
            System.arraycopy(dexElements, 0, newElements, patchElements.length, dexElements.length);

            /**
             * 5.将新数组置换掉BaseDexClassLoader -> pathList -> dexElements属性，至此工作完成
             */
            dexElementsField.set(pathList, newElements);

        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

public class ReflectUtils {
    /* 找到反射属性 **/
    public static Field findField(Object instance, String name) {
        Class<?> clz = instance.getClass();
        Field field = null;
        while (clz != Object.class) {
            try {
                field = clz.getDeclaredField(name);
                field.setAccessible(true);
                return field;
            } catch (NoSuchFieldException e) {
                e.printStackTrace();
            }
            //向父类寻找属性
            clz = clz.getSuperclass();
        }
        return field;
    }
    /* 找到反射函数 **/
    public static Method findMethod(Object instance, String name, Class<?>... parameterTypes) {
        Class<?> clz = instance.getClass();
        Method method = null;
        while (clz != Object.class) {
            try {
                method = clz.getDeclaredMethod(name, parameterTypes);
                method.setAccessible(true);
                return method;
            } catch (NoSuchMethodException e) {
                e.printStackTrace();
            }
            //向父类寻找属性
            clz = clz.getSuperclass();
        }
        return method;
    }
    /* 找到反射所有函数数组 **/
    public static Method[] findAllMethods(Object instance) {
        Class<?> clz = instance.getClass();

        return clz.getDeclaredMethods();
    }
}
```

实际上，类替换方案的核心思想就是：将修改后的patch（包含bug类文件）打包成dex文件，然后hook ClassLoader加载流程，将这个dex文件插入到Element数组的第一个元素。因为加载类是依次进行的，所以虚拟机从第一个Element找到类后，就不会再加载bug类了。

类加载方案也有缺点，因为类加载后无法卸载，所以类加载方案必须重启App，让bug类重新加载后才能生效

### Instant Run方案

> Instant Run 方案的核心思想是——插桩，在编译时通过插桩在每一个方法中插入代码，修改代码逻辑，在需要时绕过错误方法，调用patch类的正确方法


首先，在编译时Instant Run为每个类插入IncrementalChange变量：
`IncrementalChange  $change;`

为每一个方法添加类似如下代码：
```java
public void onCreate(Bundle savedInstanceState) {
        IncrementalChange var2 = $change;
        //$change不为null，表示该类有修改，需要重定向
        if(var2 != null) {
            //通过access$dispatch方法跳转到patch类的正确方法
            var2.access$dispatch("onCreate.(Landroid/os/Bundle;)V", new Object[]{this, savedInstanceState});
        } else {
            super.onCreate(savedInstanceState);
            this.setContentView(2130968601);
            this.tv = (TextView)this.findViewById(2131492944);
        }
    }
```

如上代码，当一个类被修改后，Instant Run会为这个类新建一个类，命名为xxx&override，且实现IncrementalChange接口，并且赋值给原类的$change变量。
```java
public class MainActivity$override implements IncrementalChange {
｝
```
此时，在运行时原类中每个方法的var2 != null，通过accessdispatch（参数是方法名和原参数）定位到patch类MainActivityoverride中修改后的方法。

Instant Run是google在AS2.0时用来实现“热部署”的，同时也为“热修复”提供了一个绝佳的思路。美团的Robust就是基于此。

### SO库修复

#### 接口调用替换
sdk提供接口替换System默认加载so库的接口
```kotlin
SOPatchManger.loadLibrary(String libName)
//代替
System.loadLibrary(String libName)
```
SOPatchManger.loadLibrary接口加载so库的时候优先尝试去加载sdk指定目录下补丁的so。若不存在，则再去加载安装apk目录下的so库

优点：不需要对不同sdk版本进行兼容，所以sdk版本都是System.loadLibrary这个接口
缺点：需要侵入业务代码，替换掉System默认加载so库的接口

#### 反射注入

采取类似类修复反射注入方式，只要把补丁so库的路径插入到nativeLibraryDirectories数组的最前面，就能够达到加载so库的时候是补丁so库而不是原来so库的目录，从而达到修复。

```
public String findLibrary(String libraryName) {
        String fileName = System.mapLibraryName(libraryName);

        for (NativeLibraryElement element : nativeLibraryPathElements) {
            String path = element.findNativeLibrary(fileName);

            if (path != null) {
                return path;
            }
        }

        return null;
    }
```
优点：不需侵入用户接口调用
缺点：需要做版本兼容控制，兼容性较差

## 热修复技术方案选型

![bDwH8x.png](https://s1.ax1x.com/2022/03/06/bDwH8x.png)

## 热修复和插件化区别

> 插件化和热修复的原理，都是动态加载 dex／apk 中的类／资源，让宿主正常的加载和运行插件（补丁）中的内容

- 插件化目标是想把需要实现的模块或功能当做一个独立的提取出来，减少宿主的规模。重在解决组件的生命周期，以及资源的问题

- 热修复目标在修复已有的问题。重在解决替换已有的有问题的类／方法／资源等

## Deemo

[jiangzhengnan/Syringe: 📌 插件化注入工程(热修复+插件化)](https://github.com/jiangzhengnan/Syringe)

## Refer

[热修复——深入浅出原理与实现](https://www.jianshu.com/p/cb1f0702d59f)
[热修复 - 西贝雪](https://www.cnblogs.com/not2/p/11392733.html)
[Android热修复技术,你会怎么选？](https://zhuanlan.zhihu.com/p/109169752)
[BigSweet/hotFixALL: 整理andfix,thinker,robust热修复使用方法和原理](https://github.com/BigSweet/hotFixALL)
[SoleilNotes/热修复.md at master · yoyiyi/SoleilNotes](https://github.com/yoyiyi/SoleilNotes/blob/master/Android/%E7%83%AD%E4%BF%AE%E5%A4%8D.md)