
---
date: 2016-12-31T11:34:50+08:00
title: "Go语言将支持Android"
description: ""
disqus_identifier: 1485833690086528871
slug: "Goyu-yan-jiang-zhi-chi-Android"
source: "https://segmentfault.com/a/1190000000585605"
tags: 
- Android 
- golang 
topics:
- 编程语言与开发
---

> 在前些日子谷歌于2009年发布的第二款自家语言Go语言刚刚发布了1.3正式版，大幅度解决了GC问题，最近谷歌开发版里又有了新爆料：谷歌终于要在1.4版本中正式支持Android系统开发，可以编译器直接编译发布到Android平台

### 概述

我们建议将Go语言引入Android平台，重点是用Go语言编写游戏程序，API将在Android
NDK中定义。

### 背景

Android平台被设定为一个多应用操作系统，一个相对于传统UNIX系统来说更依赖于网络库和服务的操作系统，这意味着我们将为Go运行时开放更多的API。

将Golang带入Android平台是一件非常糟糕的事情，因为Android本身是Java构建的并且拥有巨量的API，任何试图将这些API用Go替代的结果将会导致一个非常糟糕的结果。无论是手工建立包装类还是自动建立都会导致程序运行非常缓慢。

然而如果使用基于C的API，通过AndroidNDK来编写一些游戏类程序可能会带来意想不到的收获。

### 提议

在Golang
1.4周期中，Android平台的编译将纳入Go的源，并且包含Android支持的cgo（由Elias
Naur贡献）。Dalvik-loadable .so外部连接文件的支持也会由Android NDK提供。

在发布上我们仍然提供交叉平台编译，并且支持在Linux主机上adb工具在Android设备上测试程序。\
我们将会提供一个名叫go.mobile的子版本，包含：

-   通过Android NDK对OpenGL,OpenSL和OpenMAX的支持。
-   一个Java-&gt;Go的工具，支持通过Java调用Go的Package，所以许多游戏的菜单UI可以通过标准SDK构建。（有可能会支持通过其他语言调用Go，比如Objective-C。）
-   集成AndroidStudio

新特性将会在Go1.4稳定版中提供。\
[戳这里查看原文](https://docs.google.com/document/d/1N3XyVkAP8nmWjASz8L_OjjnjVKxgeVBjIsTr5qIUcA4/preview?sle=true&pli=1)

