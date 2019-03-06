---
title: Android性能优化【应用启动优化与白屏源码分析】
date: 2019-01-08 22:01:27
tags: 性能优化 源码分析  白屏优化
---
## 白屏是什么？
在Android系统中，Activity组件在启动之后，并且在它的窗口显示出来之前，可以显示一个启动窗口。这个启动窗口可以看作是Activity组件的预览窗口，是由WindowManagerService服务统一管理的，即由WindowManagerService服务负责启动和结束。
Activity组件的启动窗口是由ActivityManagerService服务来决定是否要显示的。如果需要显示，那么ActivityManagerService服务就会通知WindowManagerService服务来为正在启动的Activity组件显示一个启动窗口，而WindowManagerService服务又是通过窗口管理策略类PhoneWindowManager来创建这个启动窗口的。这个过程如图所示。
![](https://app.yinxiang.com/shard/s8/res/03c1489d-203b-4e4a-a3bc-bd9cd3d1af66.jpg)

窗口管理策略类PhoneWindowManager创建完成Activity组件的启动窗口之后，就会请求WindowManagerService服务将该启动窗口显示出来。当Activity组件启动完成，并且它的窗口也显示出来的时候，WindowManagerService服务就会结束显示它的启动窗口。

## 如何检测App启动时间
### ADB命令
`adb shell am start -W -S com.xrom.intl.appcenter/com.xrom.intl.appcenter.ui.main.MainActivity`
```shell
Activity: com.xrom.intl.appcenter/.ui.main.MainActivity
ThisTime: 1258
TotalTime: 1258
WaitTime: 1328
Complete
```

WaitTime = endTime - startTime

WaitTime就是总的耗时，包括前一个应用Activity pause的时间和新应用启动的时间；
ThisTime表示一连串启动Activity的最后一个Activity的启动耗时；
TotalTime表示新应用启动的耗时，包括新进程的启动和Activity的启动，但不包括前一个应用Activity pause的耗时。
**也就是说，开发者一般只要关心TotalTime即可，这个时间才是自己应用真正启动的耗时。**

### sysTrace工具
Android的SDK开发包中platform-tools文件夹里包含了systrace工具，在使用命令之前可以配置参数和别名，避免重复敲命令。
```shell
alias st-start='python /home/tanhuanpei/IDE/Sdk/platform-tools/systrace/systrace.py'
alias systrace='st-start -t 8 gfx input view sched freq wm am hwui workq res dalvik sync disk load perf hal rs idle mmc'
```
使用时
```shell
systrace -a com.xrom.intl.appcenter -o test.trace
```
## 启动时间分析
使用adb命令和systrace命令生成test.trace文件

![](https://app.yinxiang.com/shard/s8/res/7bd315f3-a9db-44da-8860-d3dc43fe9515.png)

**执行bindApplication**
从图中第一行可以看到,由于是第一次启动,这个应用的`bindApplication`方法被调用,

**执行onCreate--onStart--onResume**
然后是`activityStart`方法,对应应用程序中的`onCreate`--`onStart`--`onResume`三个方法.

**执行performTraversals**
紧接着会执行两次performTraversals方法(其源码位于Frameworks/base/core/java/android/view/ViewRootImpl.java), ViewRootImpl使用mFirst这个变量来标记是否是第一次启动.第一次执行performTraversals会执行mAttachInfo.mHardwareRenderer.initializ方法,初始化Surface.
![](https://app.yinxiang.com/shard/s8/res/0a0b1acd-4406-49c6-bf9e-9155cf591a13.png)

第一次创建Surface之后,newSurface为true,从下面的代码来看会执行另一次performTraversals.这就是为什么启动应用的时候需要执行两次performTraversals.分析Trace图的时候也可以根据两次performTraversals执行的情况看出问题.

![](https://app.yinxiang.com/shard/s8/res/b08c8918-d9d7-45d2-a213-66e69eef4d11.png)


所以第二个performTraversals中会执行performDraw方法。

**显示应用界面**
一般来说第二个performTraversals执行完成后, 表示应用程序的第一帧被绘制完成.
一旦应用绘制完成,WMS会收到`FINISHED_STARTING`标记应用启动完成,这时会Remove掉Starting Window.这样应用就显示出来了.

## 应用启动优化的目标
由于热启动和冷启动在优化方面的差异不大,就以最常见的的冷启动方式来确定优化的目标.

通过之前的知识可以知道,冷启动的耗时比较长,应用初始化的时间比较长.所以大部分人情况下Starting Window都是做完动画(即撑满屏幕)后,过一会儿才会被Remove掉. 从优化的角度来说,我们肯定是希望应用启动的时间越快越好.但是也要根据实际情况为其设置一个合理的数值.

### 启动优化的目标设定思路
在手机使用方面,人的感官对于卡顿的感知是很灵敏的,同样对于加载时长的容忍也是很有限的,除非使用其他手段在加载时吸引用户的注意.

但是对于应用启动来说,用户的需求是非常明确的: 就是要快速打开应用,看到主界面.在用户点击到主界面显示,其中比较重要的可以优化的几个点:
- Starting Window的初始化
- Starting Window的动画
- 应用的初始化时间

其中对应的可优化的点:
- 优化Starting Window的初始化时间(系统级优化)
- 优化Starting Window的动画(系统级优化,包括动画的时长,开始的大小,以及透明度等)
- 优化应用程序自身(应用级优化,包括精简onCreate/onStart/onResume函数,优化主界面的复杂度等)

从流畅性和连贯性的角度来说,如果在Starting Window刚好做完动画的时候,应用也初始化完成,这时候将Starting Window Remove掉.从用户的角度来看白屏没有停留,就显得很流畅.



从系统优化的角度来看,Android应用启动优化分为两个阶段的优化:

- 系统级服务的优化:
	-  ActivityManagerService的优化.
	- WindowManagerService的优化.
	- Touch Event的优化.
	- Launcher的优化.
	- 系统公共控件的优化.
	
- 应用相关的优化
	- 应用布局优化.
	- 懒加载: 即按需加载.
	- 延迟加载: 即精简OnCreate函数.

## 参考链接
[怎么计算apk的启动时间](https://www.zhihu.com/question/35487841/answer/63007551 "怎么计算apk的启动时间")