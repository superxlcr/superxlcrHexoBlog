---
title: gradle插件编写（三）
tags: [gradle]
categories: [gradle]
date: 2020-07-16 19:56:12
description: 如何查看Android官方gradle插件源码
---

上一篇博客可见：[gradle插件编写（二）](/2020/03/29/gradle插件编写（二）/)

博主最近有在编写gradle相关的插件，总结了一些新的相关问题，在此记录一下

# 如何查看Android官方gradle插件源码

当我们在编写gradle插件时，偶尔会有需要参考或者修改官方编译流程的时候
这时我们该去哪里查看官方的gradle插件源码呢？

经过一番查找，博主找到如下记录：
https://stackoverflow.com/questions/41379103/source-code-of-googles-gradle-plugin-for-building-android

上文的回答意思是我们可以去gradle的缓存目录中（一般在C盘的用户文件夹下）
.gradle/caches/modules-2/files-2.1/com.android.tools.build/gradle-core/
我们在该文件夹下可以搜索到一个类似于gradle-core-2.3.1-sources.jar的文件（2.3.1表示你使用的gradle插件版本）
反编译即可得到源码了

又或者我们可以直接clone别人准备好的gradle源码，方便用于debug编译流程调试
https://github.com/zawn/android-gradle-plugin