---
title: AndroidStudio 奇葩问题之R文件过大解析出错
tags: [android,杂项]
categories: [android]
date: 2019-05-22 19:23:19
description: AndroidStudio 奇葩问题之R文件过大解析出错
---

最近博主在导入公司工程时，遇到了一个奇怪的问题：
在AndroidStudio中，R相关的类全都变成了红色，并显示"cannot resolve symbol R"

一般而言，这种都是因为代码编译出现错误，aapt没有正确生成R文件导致的
然而奇怪的是，在执行了rebuild命令后，编译却是通过的，并正确的生成了R文件，但代码中R相关的类依旧是红色

经过一番搜索，博主在stackOverflow找到了以下资料：
https://stackoverflow.com/questions/17421104/android-studio-marks-r-in-red-with-error-message-cannot-resolve-symbol-r-but/50738195#50738195

根据资料的描述，这是由于我们生成的R文件太大，导致IDE解析R文件的时候出现了错误
修复的方式只要把最大文件大小设置大一点即可

![设置路径](1.png)

如上图所示，我们可以通过 help -> Edit Custom Properties 打开idea.properties文件进行设置

博主的个人设置如下所示
```
# custom Android Studio properties

idea.max.intellisense.filesize=99999
```

设置完重启之后，即可解决问题