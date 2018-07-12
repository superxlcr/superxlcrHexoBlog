---
title: adb常用命令总结
tags: [android,应用]
categories: [android]
date: 2017-07-29 23:39:49
description: adb介绍、开启关闭adb服务、查看当前连接的设备、安装和卸载apk程序、上传和下载文件、显示和导出log信息、adb获取root权限、启动adb命令行、adb截屏并下载到电脑
---
# adb介绍
ADB，即 Android Debug Bridge，它是 Android 开发/测试人员不可替代的强大工具，也是 Android 设备玩家的好玩具。借助adb工具，我们可以管理设备或手机模拟器的状态。还可以进行很多手机操作，如安装软件、系统升级、运行shell命令等等。



# 开启关闭adb服务

- 关闭adb服务：adb kill-server
- 开启adb服务：adb start-server




# 查看当前连接的设备
adb devices


# 安装和卸载apk程序

- 安装apk：adb install &lt;apk_name&gt;
- 卸载apk：adb uninstall &lt;apk_name&gt;




# 上传和下载文件

- 上传文件：adb push &lt;本地文件&gt; &lt;远程路径&gt;
- 下载文件：adb pull &lt;远程文件&gt; &lt;本地路径&gt;




# 显示和导出log信息

- 显示log：adb logcat
- 导出log信息：adb logcat &gt; 1.txt




# adb获取root权限

- adb以root权限执行：adb root
- adb回收root权限：adb unroot




# 启动adb命令行
adb shell


# adb截屏并下载到电脑
adb exec-out screencap -p &gt; picture_name.png



更多adb命令可见：https://github.com/mzlogin/awesome-adb
