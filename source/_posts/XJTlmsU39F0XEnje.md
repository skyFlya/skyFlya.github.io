---
title: Android - 多渠道打包
categories: Android
updated: 1647136200000
thumbnail: 'https://s1.ax1x.com/2022/03/14/bL6CLQ.md.png'
date: 2022-03-13 09:49:49
tags:
---

> 本质就是给APK添加特定的标签信息

<!--more-->
### 普通多渠道打包方案

#### Android Gradle Plugin

1. 首先，在AndroidManifest.xml中添加渠道信息占位符：
```xml
<meta-data 
android:name="InstallChannel" android:value="${InstallChannel}" />
```

2. 通过Gradle Plugin提供的`productFlavors`标签，添加渠道信息：

```groovy
productFlavors{
    "YingYongBao"{
        manifestPlaceholders = [InstallChannel : "YingYongBao"]
    }
    "360"{
        manifestPlaceholders = [InstallChannel : "360"]
    }
}
```

#### ApkTool 

> ApkTool是一个逆向分析工具，可以把APK解开，添加代码后，重新打包成APK

-   复制一份新的APK
-   通过ApkTool工具，解压APK（apktool d origin.apk）
-   删除已有签名信息
-   添加渠道信息（可以在APK的任何文件添加渠道信息）
-   通过ApkTool工具，重新打包生成新APK（apktool b newApkDir）
-   重新签名

[Apktool - A tool for reverse engineering 3rd party, closed, binary Android apps](https://ibotpeaches.github.io/Apktool/)
  
### 添加comments多渠道打包

> 利用的是Zip文件“可以添加comment（摘要）”的数据结构特点，在文件的末尾写入任意数据，而不用重新解压zip文件（apk文件就是zip文件格式）

开源实现:[seven456/MultiChannelPackageTool: Android Multi channel package tool （安卓多渠道打包工具）](https://github.com/seven456/MultiChannelPackageTool)

但由于Android7.0之后新增了V2签名，该签名会校验整个APK的数据摘要，导致上述渠道打包方案失效。所以如果想继续使用上述方案，需要关闭Gradle Plugin中的V2签名选项，禁用V2签名。



###  美团的Android多渠道打包

>  通过可扩展的APK Signature Scheme v2 Block,添加渠道信息

开源实现: [Meituan-Dianping/walle: Android Signature V2 Scheme签名下的新一代渠道包打包神器](https://github.com/Meituan-Dianping/walle)

原理:[新一代开源Android渠道包生成工具Walle - 美团技术团队 (meituan.com)](https://tech.meituan.com/2017/01/13/android-apk-v2-signature-scheme.html)


### 豌豆荚Android多渠道打包

> 把一个Android应用包当作zip文件包进行解压，然后发现在签名生成的目录下添加一个空文件不需要重新签名。利用这个机制，该文件的文件名就是渠道名。但由于v2签名的校验机制,添加了一个非空文件就会破坏签名校验，需要重新签名。

<!--more-->
提前构建一个没有渠道的APK，输入原始APK文件和渠道号，产出包含渠道号标记的APK.

1.使用AAPT添加渠道文件：

```shell
aapt add base.apk asserts/channel.txt
```

2.使用zipalign工具将APK进行字节对齐

```shell
zipalign -f 4 base.apk  app_zipalign.apk}
```

3.使用apksigner对apk重新签名

```shell
java -jar apksigner sign --v1-signer-name CERT -ks ${mStore} --ks-key-alias ${mAlias}
```

### APK打包流程
![bWKCWV.png](https://s1.ax1x.com/2022/03/09/bWKCWV.png)

-   打包资源文件，生成 R.java 文件
    -   aapt 工具（aapt.exe） -> AndroidManifest.xml 和 布局文件 XMl 都会编译 -> R.java -> AndroidManifest.xml 会被 aapt 编译成二进制
    -   res 目录下资源 -> 编译，变成二进制文件，生成 resource id -> 最后生成 resouce.arsc（文件索引表）
-   处理 aidl 文件，生成相应的 Java 文件
    -   aidl 工具（aidl.exe）
-   编译项目源代码，生成 class 文件
-   转换所有 class 文件，生成 classes.dex 文件
    -   dx.bat
-   打包生成 APK 文件
    -   apkbuilder 工具打包到最终的 .apk 文件中
-   对APK文件进行签名
-   对签名后的 APK 文件进行对齐处理（正式包）
-   对 APK 进行对齐处理，用到的工具是 zipalign

### APK 签名

> 了解 HTTPS 通信的同学都知道，在消息通信时，必须至少解决两个问题：一是确保消息来源的真实性，二是确保消息不会被第三方篡改。
同理，在安装 apk 时，同样也需要确保 apk 来源的真实性，以及 apk 没有被第三方篡改


签名机制主要有两种用途：
-  使用特殊的 key 签名可以获取到一些不同的权限
-  验证数据保证不被篡改，防止应用被恶意的第三方覆盖

签名工具
-  jarsigner：jdk 自带的签名工具，对 jar 进行签名。使用 keystore 文件进行签名，生成的签名文件默认使用 keystore 的别名命名。
-  apksigner：Android sdk 提供的专门用于 Android 应用的签名工具。使用 pk8、x509.pem 文件进行签名。 pk8 是私钥文件，x509.pem 是含有公钥的文件。生成的签名文件统一使用“CERT”命名。

#### V1 签名

>基于 JAR 签名

##### 签名

![bWMbDg.png](https://s1.ax1x.com/2022/03/09/bWMbDg.png)

#####  校验

![bWQS2V.png](https://s1.ax1x.com/2022/03/09/bWQS2V.png)

-   检查 APK 中包含的所有文件，对应的摘要值与 MANIFEST.MF 文件中记录的值一致。
-   使用证书文件（RSA 文件）检验签名文件（SF 文件）没有被修改过。
-   使用签名文件（SF 文件）检验 MF 文件没有被修改过。

##### v1 弊端

-   签名检验速度慢：对所有文件进行摘要绩，如果 Android 机器差，安装速度慢。
-   完整性保障不够：META-INF 目录用来存放签名，但可以随意添加文件。

####  v2 签名

> 一种全文件签名方案，能够发现对 APK 的受保护部分进行的所有更改，从而有助于加快验证速度并增强完整性保证。

- 验证归档中的所有字节，而不是单个 ZIP 条目，因此，在签署后无法再运行 ZIPalign（必须在签名之前执行）

- v2 会在原先 APK 块中增加了一个Signing Block（签名块），新的块存储了签名、摘要、签名算法、证书链和额外属性等信息。最终的签名APK有四块：头文件区、V2签名块、中央目录、尾部。

各版本签名组成:
![bWl9JI.png](https://s1.ax1x.com/2022/03/09/bWl9JI.png)


#### v3 签名

在v2的基础上加入了证书的旋转校验，即可以在一次的升级安装中使用新的证书，新的私钥来签名APK。当然这个新的证书是需要老证书来保证的，类似一个证书链。


#### 签名版本区别
| 版本 | 简介                                                         |
| :----: | ------------------------------------------------------------ |
| v1   | 基于JAR 签名,签名以文件的形式存在于apk包中，这个版本的apk包就是一个标准的zip包 |
| v2   | 在 Android 7.0 引入,签名信息被塞到了apk文件本身中，这时apk已经不符合一个标准的zip压缩包的文件结构 |
| v3   | 在 Android 9.0 引入,添加了一种更新证书的方式，这部分更新证书的数据同样被放在了签名信息中 |

### Refer

[Android多渠道打包（VasDolly实现原理） - 简书 (jianshu.com)](https://www.jianshu.com/p/5baea0e1cd1e)
[详解Android v1、v2、v3签名(小结）_](https://www.jb51.net/article/174939.htm)
[Android P v3签名新特性](https://blog.csdn.net/bobby_fu/article/details/103843038)
[SoleilNotes/v1、v2、v3签名区别](https://github.com/yoyiyi/SoleilNotes/blob/master/Android/v1%E3%80%81v2%E3%80%81v3%E7%AD%BE%E5%90%8D%E5%8C%BA%E5%88%AB.md)



