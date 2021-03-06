
---
date: 2016-12-31T11:34:11+08:00
title: "使用intelliJ做为Golang的IDE"
description: ""
disqus_identifier: 1485833651247541375
slug: "shi-yong-intelliJzuo-wei-Golangde-IDE"
source: "https://segmentfault.com/a/1190000003933657"
tags: 
- ide 
- intellij-idea 
- golang 
topics:
- 编程语言与开发
---

`intelliJ`是java的开发利器，一个非常好用的IDE，`Android Studio`就是基于`intelliJ`改的，选择使用`intelliJ`作为`Golang`的IDE，很是便利。

下载intelliJ
------------

首先要下载<http://www.jetbrains.com/idea/>\
`intelliJ`分为社区版和专业版，建议使用专业版,如果暂时还不想付费，也可以考虑先实现社区版是免费的，对于做`Golang`或者`Java`开发的影响很小，但是使用社区版的`intelliJ`想要开发`html`,`css`和`JavaScript`就没有那么方便了。

安装Golang插件
--------------

`Golang`插件是是个开源项目，项目地址：[https://github.com/go-lang-pl...](https://github.com/go-lang-plugin-org/go-lang-idea-plugin)

在`intelliJ`的`Plugins`中可以找`Golang`的插件,直接下载即可。但是大部分时候因为大家都知道的原因，并不一定能够搜索到，或者能够搜索到也不一定能够下载，遇到这种情况可以去官网上下载对应的`jar包`再手动安装。

插件地址：<https://plugins.jetbrains.com/plugin/5047>\
下载最新版本的`Golang`插件。\
选择 `install plugin form disk` 安装,安装完成之后可以看到`Golang`插件:

配置go sdk
----------

在已经设置`GOPATH`的情况下，`intelliJ`会自动找到go语言的目录,如果找不到也可以自己配置.\
`perferences`-&gt;`Languages&Frameworks`-&gt;`Go`-&gt;`Go Libraries`,可以看到`GOPATH`的配置：\
\
可以看到直接使用系统的`GOPATH`，也可以在`+`中选择`GOPATH`的位置。

测试
----

新建项目，选择go语言：

新建文件`main.go`,输入代码：

        package main
        import "fmt"
       
        func main(){
            fmt.Println("hello go")
        }

直接运行,输出结果：

    hello go

