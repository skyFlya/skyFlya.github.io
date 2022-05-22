---
title: 安卓优化 - 包体积
tags:
  - 性能优化
categories: Android
updated: 1635684466000
date: 2021-10-31 20:48:08
---


[Android APP 终极瘦身指南](https://cloud.tencent.com/developer/article/1425318)

- 图片转WebP
- 去掉不必要的so库
- 开启shrinkResources去除无用资源
- 开启minifyEnabled混淆代码
- 删除无用的语言资源
- 使用微信资源压缩打包工具
- 避免重复库,以及避免不同版本的库
- AndroidManifest中 -> android:extractNativeLibs="true"