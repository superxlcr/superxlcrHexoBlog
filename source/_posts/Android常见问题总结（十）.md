---
title: Android常见问题总结（十）
tags: [android,基础知识]
categories: [android]
date: 2020-03-28 22:31:32
description: Android Studio Build Output输出栏内汉字出现乱码的解决方案
---

上一篇博客传送门：[Android常见问题总结（九）](/2019/08/07/Android常见问题总结（九）/)

# Android Studio Build Output输出栏内汉字出现乱码的解决方案

最近博主在导入一个Android工程时，遇到了编译后Build Output输出栏乱码的问题，情况大致如下图：

![网上找的图](1.png)

在通过搜索引擎找了一通答案之后，最后的解决方案如下：

1. 打开Configure —> Edit Custom VM Options
2. 添加如下内容后重启Android Studio
```
 -Dfile.encoding=UTF-8
```

![还是网上找的图](2.png)

大意就是设置gradle编译jvm的编码是UTF-8，设置完成后重启即可解决问题