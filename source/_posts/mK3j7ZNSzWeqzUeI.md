---
title: Cocos和Android对接文档
tags:
  - CoCosCreator
  - Android
categories: CoCosCreator
thumbnail: 'https://s4.ax1x.com/2022/03/02/b8oz5R.md.png'
date: 2022-05-25 15:39:13
---

> CoCos构建后利用Android Studio发布apk包流程

<!--more-->
# 一、构建apk安装包
1.下载安装Android Studio开发环境，拉取对应的Android项目代码，切换到指定Git分支。
2.构建Cocos Creator项目，将构建完成的【assets、jsb-adapter、src、main.js、project.json】（一般是在Cocos项目【build/jsb-default】目录中）复制到Android项目的【game/src/main/assets】目录。
![bWKCWV.png](https://s1.ax1x.com/2022/05/25/XkCB60.png)
![bWKCWV.png](https://s1.ax1x.com/2022/05/25/XkC67F.png)
![bWKCWV.png](https://s1.ax1x.com/2022/05/25/XkCRh9.png)
3.构建apk安装包前，先点击Android Studio右上角的【Sync Project With Gradle Files】，再点击Android Studio工具栏的【Build -- Clean Project】，等编译完成后，点击Android Studio右侧【Gradle -- BoomBoom -- app -- Task -- channel】中的【channelDebug】或者【channelRelease】构建【测试】或者【正式】apk安装包，构建完成的apk文件在Android项目下的【app/channel/debug(releases)】中。
![bWKCWV.png](https://s1.ax1x.com/2022/05/25/XkCLhd.png)
![bWKCWV.png](https://s1.ax1x.com/2022/05/25/XkPSnf.png)
![bWKCWV.png](https://s1.ax1x.com/2022/05/25/XkPPAg.png)
4.如果需要真机运行代码，需要用USB连接线插入手机到电脑，打开手机的开发者选项，开启USB调试，点击Android Studio的【Run ‘app’】即可运行到手机。
![bWKCWV.png](https://s1.ax1x.com/2022/05/25/XkPFhj.png)
