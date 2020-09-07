---
title: Android常见问题总结（十）
tags: [android,基础知识]
categories: [android]
date: 2020-03-28 22:31:32
description: Android Studio Build Output输出栏内汉字出现乱码的解决方案、Android Studio 正则表达式分组替换信息、aar库依赖项打包问题、下载AndroidSDKtools目录
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

# aar库依赖项打包问题

当我们在打包aar时，会发现其实打包进aar里面的代码只有我们源码编译出来的产物（或者是以文件形式依赖的jar库等）
我们再gradle中声明的各种依赖并没有打进aar中，那么当我们使用打包出来的aar的时候，为什么不会出现运行错误呢？

在经过一番搜索后，找到了相关的资料：
https://stackoverflow.com/questions/51310920/transitive-aar-dependencies

经过了解，其实aar只会打包自己源码的东西，我们aar所依赖的项都是由maven来进行管理维护的
当我们发布aar的时候，会生成相应的pom文件，来描述我们这个aar的属性以及他所需要的依赖的库
当我们使用发布的aar的时候，maven以及gradle会根据这个pom文件把相应的依赖给补全，这就是传递依赖的场景

当我们不需要使用传递依赖时，我们在gradle中可以通过exclude方法或者transitive属性来控制：

```
implementation (xxx) {
	exclude (group : "xxx", module : "xxx")
}

implementation (xxx) {
	transitive = false
}
```

# 下载AndroidSDKtools目录

在Android的SDK下的tools目录中，存放着一些开发相关的工具，比如博主经常使用的monitor（里面包含一大堆工具，虽然新版AS也有，但总感觉不好用，尤其是界面dump工具）
不过貌似在AS 3.0及以上，官方默认下载SDK时就不下载这套工具了

博主经过网上搜索了一番后，找到了相关的资料来下载：
https://stackoverflow.com/questions/28789556/android-studio-sdk-tools-directory-is-missing/61040516#61040516

其实就是因为被标记废弃，所以被隐藏起来了，只要去掉勾选隐藏废弃package就能找到Android SDK Tools