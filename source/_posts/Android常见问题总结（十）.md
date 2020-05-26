---
title: Android常见问题总结（十）
tags: [android,基础知识]
categories: [android]
date: 2020-03-28 22:31:32
description: Android Studio Build Output输出栏内汉字出现乱码的解决方案、Android Studio 正则表达式分组替换信息
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

# Android Studio 正则表达式分组替换信息

在Android Studio中，我们可以通过正则表达式来搜索内容
举个例子：

```
我们可以通过正则表达式：
author:.*
来匹配字符串：
author: lmr
```

那么我们该如何实现分组替换功能呢？
使用**括号与$**即可：
```
author:(.*)
搭配替换字符串：
creator is$1 !!
可以把上面匹配的字符串替换为：
creator is lmr !!
```

PS：$0 表示的是匹配的整个字符串变量