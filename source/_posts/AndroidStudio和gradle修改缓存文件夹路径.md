---
title: AndroidStudio和gradle修改缓存文件夹路径
tags: [android,gradle]
categories: [android]
date: 2020-07-16 20:51:36
description: （转载）Android Studio的缓存文件、如何配置
---

本文转载自：http://blog.csdn.net/xx326664162/article/details/52004676

# Android Studio的缓存文件

默认安装的AndroidStudio会在C:\Users\YourName\ .xxx 缓存一些数据

主要有四个文件夹，分别是

- .android：这个文件夹是Android SDK生成的AVD（Android Virtual Device Manager）即模拟器存放路径
- .AndroidStudio：配置、插件缓存文件夹、最近打开的项目
- .gradle：这其中存储的是本地的gradle全局配置文件 ，但是在每次更新gradle后，这个文件都会增大（可以配置离线gradle）
- .m2：maven仓库下载的库文件保存在这里，你使用的所有的maven仓库都会先缓存到这里然后再添加到你的项目中进行使用；如果你用的插件越多这个文件夹将会持续增大

这三个文件默认都是在C盘的，如何把它们移动到指定的路径呢？？

# 如何配置

## .android文件夹的修改

1、这个文件夹是由Android SDK配置模拟器生成的，也是最占空间的一个。

首先，需要添加一个系统的环境变量ANDROID_SDK_HOME，如下图：

![环境变量设置](1.png)

变量名其实有些误导人，这个如果Google官方定义成AVD_HOME可能还好一些，其实应该是模拟器的默认路径。

2、添加好环境变量后到新的路径下修改下相应的.ini文件内的路径信息，然后重启系统生效。

![ini文件设置](2.png)

## .AndroidStudio文件夹的修改

这个文件夹的配置有些不太一样，只能从默认的安装文件中去配置。首先进入你的 AndroidStudio 安装目录中的 Bin 文件夹。

```
\Android\AndroidStudio\bin
```

![idea.properties文件位置](3.png)

进入文件：idea.properties ，而后修改如下：

![idea.properties文件](4.png)

这里是我的修改方式，当然你可以设置到你需要的地方，修改好后如果不想 AndroidStudio 重新更新下载，那么直接把文件夹从原来的地方剪切到你设置的地方去吧。

测试发现，虽然正确修改idea.properties文件，并且移动文件夹到新路径，但是在C盘还是会出现.AndroidStudio文件夹，但是Android Studio使用的却是新路径的文件夹。我使用的是AndroidStudio2.1.2

## .gradle文件夹的修改

这个文件夹直接进入 AndroidStudio > File > Settings

![gradle文件夹修改](5.png)

我使用这个方法可行，但是我参考的[这篇文章](https://blog.csdn.net/qiujuer/article/details/44160127)却说这个方法不行，如果你使用这个方法也不行，可以[参考这里](https://blog.csdn.net/qiujuer/article/details/44257993)

测试发现，虽然正确配置路径，并且移动文件夹到新路径，但是在C盘还是会出现.gradle文件夹，但是Android Studio使用的却是新路径的文件夹。我使用的是AndroidStudio2.1.2

## .m2文件夹的修改

1、这个的配置也相对简单，同样在设置中进行更改：

![maven文件夹修改](6.png)

2、修改好后如果不想 AndroidStudio 重新更新下载，那么直接把.m2文件夹从原来的地方剪切到你设置的地方。

参考文章：
http://blog.csdn.net/qiujuer/article/details/44160127
http://blog.csdn.net/qiujuer/article/details/44257993
http://www.jianshu.com/p/7a58c5f154c5