---
title: 修改debuggable调试手机应用
tags: [android]
categories: [android]
date: 2019-08-06 19:19:15
description: 通过修改手机的ro.debuggable属性，调试所有应用
---

最近博主在调试手机应用时遇到了一些问题，因为某些原因在一些低版本的手机上无法编译出
```
android:debuggable="true"
```
属性的安装包，导致无法调试应用

在网上经过查阅资料后，发现想要调试手机上的app只要达到以下条件之一即可：
- apk的AndroidManifest属性是android:debuggable="true"
- 手机上的ro.debuggable属性置为1

既然我们无法编译出一个可调试的apk，那么我们只能修改手机的debuggable属性了
不过由于ro.debuggable是个系统属性，因此我们使用adb setprop并不能改变它，需要重新编译rom才可以

又在网上经过查阅资料后，博主发现了一个名为mprop的工具
只要操作一下几步即可处理问题：
1. 下载mprop工具并push到手机上（博主push到了/data/local/tmp/下，不知道目录是否有影响）
2. root手机
3. chmod使得mprop工具变为可执行文件
4. ./mprop执行
5. 使用setprop修改想要修改的属性
6. ./mprop -r 使得修改生效即可

通过把手机的ro.debuggable属性设置为1之后，我们就可以愉快的调试手机上的所有应用了！

参考自（内含mprop下载目录）：https://www.jianshu.com/p/e540f34cec07