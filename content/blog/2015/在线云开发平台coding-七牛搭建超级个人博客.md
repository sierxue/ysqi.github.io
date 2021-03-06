---
date: 2015-08-23
title: 在线云开发平台coding+七牛搭建超级个人博客
slug: coding-qiniu-build_blog
tags : 
- WebIDE
- 七牛
- 教程
Topics: 
- 软件
description: 如何使用在线开发平台搭建基于七牛的个人博客教程
disqus_identifier: 100002
---

昨天加入了codefollow这个大家庭，话说要鼓励大家写作与分享，趁着老婆睡觉偷偷的写一篇如何利用云开发平台coding以及七牛这条大黄牛来搭建一个超级方便又稳定的个人博客。
<!--more-->

### 准备篇
+ 国内已备案域名

  原因是博客搭建在七牛上，而七牛不支持个性域名，而是一圈不美观的随机域名，如`7xlagq.com1.z0.glb.clouddn.com`，所以需要一个已备案的域名来做七牛自定义域名配置

+ 七牛账户、Coding.net账户

  账户不必多说，这是必须的，至于账号申请于空间设置，自己去搜索，这里不提供。

+ 了解hexo

  静态博客使用hexo构建，想进一步了解，移步->[hexo.io](hexo.io)

+ 了解Coding的WebIDE

  使用Coding.net的WebIDE来进行环境搭建和写作

### 博客搭建原理

我们搭建的静态博客网站，既然全是静态文件，那么我们就可以直接将网址文件全部上传到七牛服务器，让七牛做静态资源服务器。

而Coding.net则提高一个在线写作环境，写好文章后，在线预览审阅后，直接同步到七牛，这样网友们通过我们的自定义域名就可以直接访问我们上传到七牛的网站啦。

### 为什么选择Coding.net

一个非常重要的原因是找一个在线平台，因为我要到处走，也没法一直带着自己的笔记本，如果有一个云端写作，那么不管我在哪，联网便可写作并发布。

其实我刚开始时准备使用c9.io的，用过的都说好，可是遇到了一个问题，如何在c9中运行`hexo server -p 4000` 访问这个4000端口，从而实现博客预览目的，如果谁知道，就在下面评论中回复我，不用谢！

而coding.net刚要是国内相对比较好的一个在线开发平台，也可以非常方便的设置端口访问，不折腾了，就用它吧！

### 搭建Coding.net写作环境

1. 在Coding.net新建项目
   创建项目时，我们切记选择私有项目，因为项目需要记录你个人的AppKey

2. 进入项目，搭建hexo写作环境
  + 进入项目的WebIDE环境中，执行命令安装hexo
  ```
  npm install hexo-cli -g
  ```
  + 检查hexo是否安装成功,如果成功则会显示hexo版本信息。
  ```
  hexo version
  hexo: 3.1.1
  os: Linux 3.13.0-58-generic linux x64
  http_parser: 2.3
  node: 0.12.7
  v8: 3.28.71.19
  uv: 1.6.1
  zlib: 1.2.8
  modules: 14
  openssl: 1.0.1p
  ```
  + 执行命令初始化一个blog目录。*切记*不要直接`hexo init`,后果是你写的文章无法生成，原因不明（欢迎评论）。
  ```
  hexo init blog
  ```
  + 进入目录
  ```
  cd blog
  ```
  + 执行命令，创建一篇文章.
  ```
  hexo new "想想都很幸福
  ```
  + 生成文章
  ```
  hexo g
  ```
  + 开启在线预览
  ```
  hexo s -p 4000
  ```
  + Coding开启端口`4000`，如下截图操作，即可在线预览。


### 七牛同步设置
 安装七牛同步插件
  ```
  
  ```