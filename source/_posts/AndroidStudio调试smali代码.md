---
title: AndroidStudio调试smali代码
tags: [android,编译]
categories: [android]
date: 2017-10-23 20:34:37
description: 下载AS插件、反编译apk文件、设置AndroidManifest中的debuggable属性为true，并重新打包签名、安装apk，开启调试模式，使用AS进行远程attach调试
---
最近看了一篇关于动态调试smali代码的文章：http://blog.csdn.net/superxlcr/article/details/78296673


由于文章是使用Eclipse调试smali代码的，因此上网找了下使用AndroidStudio调试smali代码的相关资料



# 下载AS插件

首先由于AndroidStudio是不支持smali的，因此我们需要下载相关插件来语法高亮以及设置断点
我们可以下载smalidea插件来安装：https://bitbucket.org/JesusFreke/smali/downloads/


![_pic1](1.png)



安装完后，smali语法会高亮显示，并且可以设置断点了



# 反编译apk文件

利用apktool工具反编译apk，下载地址：https://bitbucket.org/iBotPeaches/apktool/downloads/



# 设置AndroidManifest中的debuggable属性为true，并重新打包签名

为了让我们的apk可以用于调试，我们需要修改AndroidManifest中的debuggable属性为true：
![_pic2](2.png)



接着我们可以通过AndroidManifest找到程序的入口Activity（通过android.intent.action.MAIN，android.intent.category.LAUNCHER等属性）


找到程序入口后，我们有两种方法用于调试：

- 在入口Activity中加入：
```
invoke-static {}, Landroid/os/Debug;->waitForDebugger()V
```
这个是smali语法，对应的Java代码为：
```java
android.os.Debug.waitForDebugger();
```
- 或者在启动Activity时使用am指令，带上-D参数用于调试：
```
adb shell am start -D -n package/MainActivity
```


接着使用apktool重新打包，并使用jarsigner签名




# 安装apk，开启调试模式，使用AS进行远程attach调试

安装完apk后，如果加入了waitForDebugger方法，则直接启动app
若是没有加入，则使用am的debug模式启动app


启动完后，打开ddms，查看应用的debug端口号：



如上图所示，红色小蜘蛛代表应用等待调试，其中应用进程号为10747，debug端口号为8620或者8700（一般用前面那个）


然后我们打开AS，导入我们的反编译工程后，在相应的smali代码中打上断点：
![_pic3](3.png)



上图为在MainActivity的onCreate方法中加入断点


然后，打开AS的Run -&gt; Edit Configurations，新建remote类型：
![_pic4](4.png)


选择attach模式，输入相应的端口号即可：
![_pic5](5.png)


调试成功：
![_pic6](6.png)


