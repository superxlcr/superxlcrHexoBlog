---
title: 记一次Native层崩溃追踪
tags: [android,cpp]
categories: [android]
date: 2017-10-13 15:36:39
description: 记一次Native层崩溃追踪
---
前一阵子遇到了一次Native层的崩溃，在此记录一下debug的心得：
不能只看Error级别的日志：因为崩溃原因在Native层，因此并不会打印Error级别的Java堆栈信息，这是我们可以通过搜索 Build fingerprint 关键字，来搜索Android在Native层打印出来的日志：

![_pic1](1.png)



可以看到打印出来的Native日志是Info级别的，因此我们不能只关注Error级别的日志


Native日志包含了进程死亡时接收的信号量，寄存器的状态以及内存的一些状态，异常定位代码（C层面的），还有就是进程死前的pid以及tid


虽然大部分信息都看不懂……但是还是可以依据pid确定进程死亡的时间，再结合前后的日志以及操作确定具体问题的。

