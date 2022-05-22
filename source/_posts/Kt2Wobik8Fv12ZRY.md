---
title: Android AOP - ASM+Transform
tags:
  - AOP
categories: Android
updated: 1647049800000
thumbnail: 'https://s1.ax1x.com/2022/03/14/bLyIxO.md.png'
date: 2022-03-12 09:45:36
---

<!--more-->
![bhoW8O.png](https://s1.ax1x.com/2022/03/10/bhoW8O.png)

### Transform

> Android Gradle Plugin 从 1.5.0 开始支持 Transform API，以允许第三方插件在经过编译的 .class 文件转换为 .dex 文件之前对其进行操纵。

普通编译过程：
![bINI8f.png](https://s1.ax1x.com/2022/03/11/bINI8f.png)
可以看到Android 构建流程是一套流水线的工作机制，每一个的构建单元接收上一个构建单元的输出，作为输入，再将产品进行输出，`com.android.build`库提供了`Transform`的机制，而这个机制是Android 构建系统为了给外部提供一个可以加入自定义构建单元，如拦截某个构建单元的输出，或者加入一些输出等。而这些Transform是在java源码编译完成之后，package之前进行的。
![bIZBy8.md.png](https://s1.ax1x.com/2022/03/11/bIZBy8.md.png)

在Android Gradle构建系统中，可以通过`AppExtension `将`transform`注册到构建系统中:
```java
public class CustomPlugin implements Plugin<Project> {
    public void apply(Project project) {
        AppExtension appExtension = (AppExtension)project.getProperties().get("android");
        // 给App编译过程注册transform
        appExtension.registerTransform(new CustomTransform(), Collections.EMPTY_LIST);
    }
}
```

### ASM

> [ASM](https://asm.ow2.io/)是一种通用**Java字节码**操作和分析框架。它可以用来修改现有的类，也可以直接以二进制形式动态生成类。

![bI0iSx.png](https://s1.ax1x.com/2022/03/11/bI0iSx.png)


ASM设计了两种API类型解析文件结构：
- Tree API
- 基于Visitor API(visitor pattern)，

让我们以不同的方式处理下吧🤤🤤🤤

#### Tree API

Tree API将class的结构读取到内存，构建一个树形结构，然后需要处理Method、Field等元素时，到树形结构中定位到某个元素，进行操作，然后把操作再写入新的class文件。


```kotlin
class TestPlugin : Plugin<Project> {
    override fun apply(target: Project) {
        println("Test Plugin : target = [${target}]")
        val appExtension: AppExtension = target.extensions.getByType()
        appExtension.registerTransform(TestTransform())
    }
}

class TestTransform : BaseTransform(){

    companion object{
        private var isPrint = true
    }
    
    private var isChange = false

    override fun modifyClass(byteArray: ByteArray): ByteArray {
        val classReader = ClassReader(byteArray)
        val classNode = ClassNode()
        // 通过传递 ClassReader 解析对应 ClassFile 生成 ClassNode
        classReader.accept(classNode, ClassReader.EXPAND_FRAMES)
        // 打印测试
        if(isPrint){
            isPrint = false
            Log.log(classNode.toString())
            for (methodNode in classNode.methods) {
                Log.log("${classNode.name} ------>>> $methodNode")
            }
        }
        // 如果文件对classNode有修改的情况下，需要这样处理
        if(isChange){
            val classWriter = ClassWriter(ClassWriter.COMPUTE_MAXS)
            classNode.accept(classWriter)
            return classWriter.toByteArray()
        }
        // 返回修改内容
        return byteArray
    }

    override fun getInputTypes(): MutableSet<QualifiedContent.ContentType> {
        return TransformManager.CONTENT_CLASS
    }

    override fun getScopes(): MutableSet<in QualifiedContent.Scope> {
        return mutableSetOf(
            QualifiedContent.Scope.PROJECT
        )
    }
}
```

可以看到编译的类正被我们打印出来：
![bIq3id.png](https://s1.ax1x.com/2022/03/11/bIq3id.png)


#### Visitor API 
Visitor API则将通过接口的方式，分离读class和写class的逻辑，一般通过一个ClassReader负责读取class字节码，然后ClassReader通过一个ClassVisitor接口，将字节码的每个细节按顺序通过接口的方式，传递给ClassVisitor（你会发现ClassVisitor中有多个visitXXXX接口），这个过程就像ClassReader带着ClassVisitor游览了class字节码的每一个指令。

```kotlin
class TestPlugin : Plugin<Project> {
    override fun apply(target: Project) {
        println("Test Plugin : target = [${target}]")
        val appExtension: AppExtension = target.extensions.getByType()
        appExtension.registerTransform(TestVisitorTransform())
    }
}
class TestClassVisitor(visitor: ClassVisitor):ClassVisitor(Opcodes.ASM4,visitor){

}
class TestVisitorTransform : BaseTransform(){

    companion object{
        private var isPrint = true
    }

    private var isChange = false

    override fun modifyClass(byteArray: ByteArray): ByteArray {
        // 解析class文件
        val classReader = ClassReader(byteArray)
        // 将class文件内容写入到ClassWriter中
        val classWriter = ClassWriter(classReader,ClassWriter.COMPUTE_MAXS)
        // 赋予对应的ClassVisitor读写能力
        val classVisitor = TestClassVisitor(classWriter)
        classReader.accept(classVisitor, ClassReader.EXPAND_FRAMES)
        // 打印测试
        if(isPrint){
            isPrint = false
            Log.log(classVisitor.toString())
        }
        // 如果文件对classNode有修改的情况下，需要这样处理
        if(isChange){
            return classWriter.toByteArray()
        }
        // 返回修改内容
        return byteArray
    }

    override fun getInputTypes(): MutableSet<QualifiedContent.ContentType> {
        return TransformManager.CONTENT_CLASS
    }

    override fun getScopes(): MutableSet<in QualifiedContent.Scope> {
        return mutableSetOf(
            QualifiedContent.Scope.PROJECT
        )
    }
}
```
同样可以看到编译的类正被我们打印出来：
![bIvebV.png](https://s1.ax1x.com/2022/03/11/bIvebV.png)

- ClassReader：负责对 .class 文件进行读取
- ClassVisitor： 负责访问 .class 文件中各个元素
- ClassWriter： 负责对 .class 文件进行写入，将字节码输出为 byte 数组。


### 字节码了解


[Android Studio 辅助类转字节码插件：ASM Bytecode Viewer Support Kotlin](https://plugins.jetbrains.com/plugin/14860-asm-bytecode-viewer-support-kotlin)
![preview](https://pic1.zhimg.com/v2-fdd5caeb51dbfb819539611fa9ccbcd4_r.jpg)

[字节码结构分析 - 掘金 (juejin.cn)](https://juejin.cn/post/6944517233674551304#heading-5)
[Java ByteCode - 简书 (jianshu.com)](https://www.jianshu.com/p/92a75a18cbc1)
[史上最通俗易懂的ASM教程 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/94498015)

### 自定义gradle插件

> Transform和Asm的结合使用需要用到Gradle插件

自定义插件方式：详看[Developing Custom Gradle Plugins](https://docs.gradle.org/current/userguide/custom_plugins.html#example_a_build_for_a_custom_plugin)
- Build script
- `buildSrc` 项目
- 独立项目（Standalone project）


#### Build script 

> 直接在项目的build.gradle中添加groovy脚本代码并引用。这样插件在构建脚本之外不可见，只能在此模块中使用脚本插件。

1. 简单编写脚本
build.gradle:
```groovy
class GreetingPlugin implements Plugin<Project> {
    void apply(Project project) {
        project.task('hello') {
            doLast {
                println 'Hello from the GreetingPlugin'
            }
        }
    }
}

// Apply the plugin
apply plugin: GreetingPlugin
```
跟上面一样，只不过采用新版kts风格
build.gradle.kts
```kotlin
class GreetingPlugin : Plugin<Project> {
    override fun apply(project: Project) {
        project.task("hello") {
            doLast {
                println("Hello from the GreetingPlugin")
            }
        }
    }
}
// Apply the plugin
apply<GreetingPlugin>()
```
2. 运行
```bash
> gradle -q hello
Hello from the GreetingPlugin
```

#### buildSrc 项目

>将插件的源代码放在rootProjectDir/buildSrc/src/main/groovy目录中，Gradle将负责编译和测试插件，并使其在构建脚本的类路径中可用。该插件每个构建脚本都是可见的。但是它在构建外部不可见即在当前工程的各个模块都可见，但是项目之外不可见

buildSrc是android中一个保留名，是一个专门用来做gradle插件的module，所以这个module的名字必须是buildSrc，此模块下面有一个固定的目录结构src/main/groovy，这个是用来存放真正的脚本文件的。其他的自定义类可以放在这个目录下，也可以放在自建的其他目录下。


1. 创建`buildSrc`Module,可以直接创建目录
	-  在工程根目录下创建目录`buildSrc`
	-  在buildSrc下创建目录结构 `src/main/groovy`或者`src/main/java`
	-  在buildSrc根目录下创建` build.gradle`或者` build.gradle.kts`

build.gradle.kts:

```kotlin
import org.jetbrains.kotlin.gradle.tasks.KotlinCompile

plugins {
    `kotlin-dsl`
    kotlin("jvm") version "1.4.32"
}

val compileKotlin: KotlinCompile by tasks
val compileTestKotlin: KotlinCompile by tasks
compileKotlin.kotlinOptions {
    jvmTarget = "1.8"
}
compileTestKotlin.kotlinOptions {
    jvmTarget = "1.8"
}

repositories {
    google()
    mavenCentral()
}

dependencies {
    implementation("com.android.tools.build:gradle:4.1.1")
    compileOnly("commons-io:commons-io:2.6")
    compileOnly("commons-codec:commons-codec:1.15")
    compileOnly("org.ow2.asm:asm-commons:9.2")
    compileOnly("org.ow2.asm:asm-tree:9.2")
}
```

build.gradle
```groovy
apply plugin: 'groovy'
 
dependencies {
    // gradle插件必须的引用
    implementation gradleApi()
    implementation localGroovy()
    
    implementation 'com.android.tools.build:gradle:4.1.1'
    // asm依赖
    implementation 'org.ow2.asm:asm:9.2'
    implementation 'org.ow2.asm:asm-util:9.2'
    implementation 'org.ow2.asm:asm-commons:9.2'
}
 
repositories {
    mavenCentral()
    jcenter()
    google()
}
 
// 指定编译的编码 不然有中文的话会出现  ’编码GBK的不可映射字符‘
tasks.withType(JavaCompile) {
    options.encoding = "UTF-8"
    println('使用utf8编译')
}
```

2. 创建插件入口TestPlugin
```kotlin
package com.onion.plugin.plugin

class TestPlugin: Plugin<Project> {
    override fun apply(target: Project) {
        println("Test Plugin : target = [${target}]")
    }
}
```
3. 在App Module 中引入,然后Project Build
```groovy
import com.onion.plugin.plugin.TestPlugin
apply plugin: TestPlugin // 插桩测试
```

#### 插件发布额外学习

-   [Android Gradle 插件开发入门指南（一）](https://juejin.cn/post/6887581345384497165 "https://juejin.cn/post/6887581345384497165")，讲解Gradle Plugin开发的完整流程
-   [Android Gradle 插件开发入门指南（二）](https://juejin.cn/post/6887583351348133895 "https://juejin.cn/post/6887583351348133895")，针对Android的Gradle Plugin开发实践
-   [Android Gradle 插件开发入门指南（三）](https://juejin.cn/post/6890544619856068616 "https://juejin.cn/post/6890544619856068616")，如何将插件发布到jcenter




### Refer

[一起玩转Android项目中的字节码 | Quinn Note (quinnchen.cn)](http://quinnchen.cn/2018/09/13/2018-09-13-asm-transform/)
[Java ByteCode](https://www.jianshu.com/p/92a75a18cbc1)
[ASM + Transform 在android中的使用](https://blog.csdn.net/qq_23992393/article/details/103696976)
[ASM](https://www.jianshu.com/p/a1e6b3abd789)
[白话 Android AOP (一) ](https://juejin.cn/post/6893917892061413389#heading-3)
[Android gradle Transform 分析](https://mp.weixin.qq.com/s/YFi6-DrV22X_VVfFbKHNEg)