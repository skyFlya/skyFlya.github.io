---
title: 如何在 VideoPlayer 上面放置 UI 控件
tags:
  - CoCosCreator
  - UI组件
categories: CoCosCreator
thumbnail: 'https://s1.ax1x.com/2022/03/07/b6aO6P.md.png'
date: 2022-05-25 15:39:16
---

> CoCos中在视频层上方放置ui

<!--more-->

最近使用Cocos Creator开发项目，有个需求是在视频上面添加按钮交互。由于视频调用的是系统原生控件，并且层级放置在了Cocos引擎层级之上，没有和游戏一起混合渲染，所以单纯在cocos这边改动 zIndex是没有效果的，必须要和对应平台打交道了。


# 为了实现视频上面放置UI，大概有三种思路：
1.将视频层级放到cocos层级之下，这样cocos的UI就可以显示在视频之上了。
2.需要放置到视频层级上的UI不使用cocos内置控件，调用对应平台控件放置在视频控件之上。
3.获取视频数据，渲染到cocos控件上。

第1、2种方式实现的最终效果一致。缺点是视频和cocos是分层显示的，所以视频只能放置在cocos控件的最下层。
第三种方式可以自由调整视频层级，但我还没有实现细节的思路，不做探讨。本文基于 Cocos Creator 2.0.7，其他版本引擎运行效果需自行测试。

**我选择用第一种方式，简单粗暴**
首先要做的是把屏幕背景色设为透明，这一步骤在iOS、Android、Web上都需要做。
打开Creator把场景挂载的 Camera 组件上的 backgroundColor 的R、G、B、A全部设为 0 ，用于清除背景色。
然后在各个平台上调整视频层级到引擎层级下面。

# iOS端：
 // 先添加videoView再添加cocosView
    UIViewController* rvC = [[UIViewController alloc]init];
    // 视频view
    UIView *videoView = [[UIView alloc] initWithFrame:bounds];
    videoView.tag = 10; // 通过tag找到view
    [rvC.view addSubview:videoView];
    // 引擎view
    UIView *cocosView = _viewController.view;
    cocosView.frame = bounds;//
    cocosView.backgroundColor = [UIColor clearColor];
    [rvC.view addSubview:cocosView];
    
    // Set RootViewController to window
    if ( [[UIDevice currentDevice].systemVersion floatValue] < 6.0)
    {
        // warning: addSubView doesn't work on iOS6
         [window addSubview: rvC.view]; //新的rootView
    }
    else
    {
        // use this method on ios6
        [window setRootViewController:rvC]; //新的rootView
    }
              
具体如图：
![bWKCWV.png](https://s1.ax1x.com/2022/05/25/XFzwqg.png)
接着修改VideoPlayer的默认层级。在引擎文件 VideoPlayer-ios.mm 中找到[eaglviewaddSubview:self.moviePlayer.view]; 方法，注释掉添加到默认层级的代码。通过tag找到我们之前新建的视频view，并添加视频到上面。加入如下代码：
[[[eaglview superview] viewWithTag:10] addSubview:self.moviePlayer.view]; // 添加到上层view
这里还要改一下颜色格式，要不然视频还是会被黑色背景遮住。在引擎文件  CCApplication-ios.mm 中找到 onCreateView 方法，我们看到引擎默认像素格式是  RGB565 。注释掉原来代码，把这里改为支持透明度的 RGBA8 格式。RGB565这种格式是不支持透明度的，所以即使我们把 UIView设为透明，视频依然被黑色背景遮住。

# Android端：
首先取消视频置顶。找到 Cocos2dxVideoHelper.java 文件，注释掉 setZOrderOnTop 方法。
![bWKCWV.png](https://s1.ax1x.com/2022/05/25/XFzWsU.png)
紧接着，我们设置像素格式。在Android原生工程中，找到 AppActivity.java 文件，在 onCreateView 方法中添加设置 glSurfaceView 透明代码：(需要保证opengl的Config中有配置alpha通道)
         Cocos2dxGLSurfaceView glSurfaceView = new Cocos2dxGLSurfaceView(this);
        //通常,为了使它与绘图树整合,这意味着它所在的窗口的其他内容都不可见，所以接下来我们需要设置surfaceView透明来使其他内容可见
        
        glSurfaceView.setEGLConfigChooser(8, 8, 8, 8, 16, 8);	  //修改GLSurfaceView参数，其中第四个参数为AlphaSize，这样修改以后View就变透明了
        glSurfaceView.getHolder().setFormat(PixelFormat.RGBA_8888);  //RGBA_8888为android的一种32位颜色格式，R,G,B,A分别用八位表示，Android默认格式是PixelFormat.OPAQUE，其是不带Alpha值的。
        
        //控制这个surfaceView是否被放在另一个普通的surfaceView上面（但是仍然在窗口之后）。这个函数通常被用来将覆盖层至于一个多媒体层上面。
        //这个函数必须在窗口被添加到窗口管理器之前设置。
        //调用这个函数会使之前调用的setZOrderOnTop(boolean)无效。
        glSurfaceView.setZOrderMediaOverlay(true);
![bWKCWV.png](https://s1.ax1x.com/2022/05/25/XkSCWt.png)
这时候视频已经可以显示在下方了，但是，如果视频上方有 cocos 按钮依然不可点击的，我们还需要向下传递事件。找到引擎目录下的  Cocos2dxVideoView.java  文件，重写 onTouchEvent方法，返回false 。这样视频上方的cocos ui就可以响应触摸事件了。
![bWKCWV.png](https://s1.ax1x.com/2022/05/25/XkSwSx.png)

# Web端：
和Android、iOS一样，首先是调整视频层级到最下层。找到 js引擎 目录下的 video-player-impl.js 文件，在 _createDom 方法中添加修改视频层级代码：
// 调整视频层级到最下层
video.style.zIndex = -1;
修改代码后还需要编译一下 js引擎 到engine目录下执行一下 gulp build 构建成功后，才能使修改的代码生效。
如果不想重新编译引擎，可以直接找到 js引擎 bin 文件夹下的 cocos2d-js-for-preview.js 文件搜索_createDom 方法中添加以上代码。
最后我们需要Canvas 背景是透明的，这样才能看到下层的视频。如果是开发时在网页上预览，需要修改Cocos Creator安装目录下， 找到preview-templates 文件夹下的 boot.js 文件，在 cc.game.run 之前加上如下代码：
// 开启支持全透明
cc.macro.ENABLE_TRANSPARENT_CANVAS = true;
如果是打web包，则在 main.js 文件中的 cc.game.run 代码之前加入以上代码。