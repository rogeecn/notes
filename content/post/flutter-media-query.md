---
title: "Flutter MediaQuery 相关说明"
date: 2021-11-26T16:07:23+08:00
draft: false
tags: ["Flutter", "MediaQuery"]
---


`MediaQueryData`是`MediaQuery.of`获取数据的类型。说明如下：

|属性|说明|
|-|-|
|size|逻辑像素，并不是物理像素，类似于Android中的dp，逻辑像素会在不同大小的手机上显示的大小基本一样，物理像素 = size*devicePixelRatio。|
|devicePixelRatio|单位逻辑像素的物理像素数量，即设备像素比。|
|textScaleFactor|单位逻辑像素字体像素数，如果设置为1.5则比指定的字体大50%。|
|platformBrightness|当前设备的亮度模式，比如在Android Pie手机上进入省电模式，所有的App将会使用深色（dark）模式绘制。|
|viewInsets|被系统遮挡的部分，通常指键盘，弹出键盘，viewInsets.bottom表示键盘的高度。|
|padding|被系统遮挡的部分，通常指“刘海屏”或者系统状态栏。|
|viewPadding|被系统遮挡的部分，通常指“刘海屏”或者系统状态栏，此值独立于padding和viewInsets，它们的值从MediaQuery控件边界的边缘开始测量。在移动设备上，通常是全屏。|
|systemGestureInsets|显示屏边缘上系统“消耗”的区域输入事件，并阻止将这些事件传递给应用。比如在Android Q手势滑动用于页面导航（ios也一样），比如左滑退出当前页面。|
|physicalDepth|设备的最大深度，类似于三维空间的Z轴。|
|alwaysUse24HourFormat|是否是24小时制。|
|accessibleNavigation|用户是否使用诸如TalkBack或VoiceOver之类的辅助功能与应用程序进行交互，用于帮助视力有障碍的人进行使用。|
|invertColors|是否支持颜色反转。|
|highContrast|用户是否要求前景与背景之间的对比度高， iOS上，方法是通过“设置”->“辅助功能”->“增加对比度”。 此标志仅在运行iOS 13的iOS设备上更新或以上。|
|disableAnimations|平台是否要求尽可能禁用或减少动画。|
|boldText|平台是否要求使用粗体。|
|orientation|是横屏还是竖屏。|
<!--more-->

padding、viewPadding和viewInsets的区别如下：

![](image.webp)


