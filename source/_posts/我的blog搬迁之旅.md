---
title: 我的blog搬迁之旅
tags:
  - 博客
categories:
  - 杂项
date: 2018-03-24 00:21:12
description: 前言、搭建本地hexo环境、编写博客、创建 github io repository、关联github并部署
---
# 前言
本来博客是搭在csdn上面的，也就是当个云记事本来使用  
不料csdn的服务器感觉不是特别的友好啊……老是登不上去什么鬼乱七八糟的，现在又弄了个新的编辑器，但并不是特别好用orz  
因此，思前想后，觉得还是把自己的博客搬到github上面好了  
在此，记录一下要做的工作吧，使得以后再次搬迁的时候尽量快点吧  

# 搭建本地hexo环境
首先，去官网下载node.js：https://nodejs.org/en/
下载安装完成后，通过指令可以看到node.js的版本
```bash
node -v
```
接着，通过npm下载安装hexo
```bash
npm install hexo-cli -g
npm install hexo --save
```
下载完后，通过指令可以看到hexo的版本
```bash
hexo -v
```
然后，我们需要创建一个空文件夹，并在下面执行指令初始化hexo
```bash
hexo init
hexo install
```
接着，我们通过指令来生成并运行hexo
```bash
hexo g
hexo s
```
下面的指令也能达到同样的效果
```bash
hexo s -g
```
最后，如果我们需要修改hexo的相关配置，可以修改根目录下的_config.yml文件  
如果我们想要使用新的hexo主题，下载资源文件到theme文件夹中，并修改配置文件即可  
至此，本地hexo环境配置基本结束  

# 编写博客
使用以下指令即可
```bash
hexo new post "title"
```
详情可查看官方doc：https://hexo.io/docs/writing.html

# 创建 github io repository
首先，创建一个 <username\>.github.io 的库
然后，选择右上角的 Settings 选项，里面的 Github Pages 选项，随便选择一个主题  
处理成功后，就能访问 <username\>.github.io 的博客页面了

# 关联github并部署
首先我们需要设置user.name、user.email以及ssh key等东西（教程百度吧）
接着需要在根目录的_config.yml文件中，修改相应的属性
```
deploy:
  type: git
  repo: git@github.com:yourname/yourname.github.io.git
  branch: master
```
接着我们需要通过以下指令安装关联插件
```bash
npm install hexo-deployer-git --save
```
接着，我们通过指令来生成并部署hexo
```bash
hexo g
hexo d
```
下面的指令也能达到同样的效果
```bash
hexo d -g
```
所有的搭建博客步骤已记录完毕，接下来就开始进行博客搬迁的工作吧