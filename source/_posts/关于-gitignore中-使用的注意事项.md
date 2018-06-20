---
title: 关于.gitignore中!使用的注意事项
tags: [git]
categories: [git]
date: 2016-04-16 17:33:48
description: 关于.gitignore中!使用的注意事项
---
最近博主设置github的.gitignore文件时遇到了一些有趣的问题，在此写一篇博客记录一下。

github提供了.gitignore配置文件用于配置不需要加入版本管理的文件，其配置语法如下：
1. 以斜杠“/”开头表示目录
2. 支持用正则表达式来匹配
3. 以叹号“!”表示不忽略(跟踪)匹配到的文件或目录

举博主的例子来说，博主的.gitignore所在目录如下：
![目录示意图](1.png)

博主打算把文件配置为仅追踪.gitignore与build/outputs/apk中的所有文件，其余文件全部忽略，因此开始时的.gitignore写为：
```
/build  
!/build/outpus/apk  
```

结果却发现github把整个build文件夹都给忽略掉了……把!那一句移到第一行也无济于事。
在经过辛苦的百度搜索后，终于找到了答案，在stackOverflow上有对此类问题的回答：
http://stackoverflow.com/questions/5533050/gitignore-exclude-folder-but-include-specific-subfolder

博主对此问题的理解是，要想追踪某个文件或文件夹，必须确保其父文件夹也被追踪，如果其父文件夹被忽略了，那么该文件自然被忽略掉了。
将.gitignore文件改为如下后成功实现了博主想要的效果：
```
build/*  
!/build/outputs  
/build/outputs/*  
!/build/outputs/apk  
```

其效果图如下（绿色代表github追踪文件）：
![git追踪目录示意图](2.png)