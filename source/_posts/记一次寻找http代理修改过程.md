---
title: 记一次寻找http代理修改过程
tags: [计算机网络,Http]
categories: [计算机网络]
date: 2017-08-14 19:33:45
description: 记一次寻找http代理修改过程
---
最近在使用fiddler抓取http包时，总是收到proxy was changed的提示：
![_pic1](1.png)

由于博主并没有主动运行vpn等工具，因此十分疑惑是什么进程修改了http的代理
经过百度，终于有一篇文章解答了博主的疑惑：
http://www.telerik.com/forums/how-to-auto-reset-fiddler-when-%27system-proxy-was-changed%27

总的来说，查找是什么进程修改http代理方法如下：
首先，我们需要下载SysInternals的Process Monitor工具：
https://docs.microsoft.com/zh-cn/sysinternals/downloads/procmon

打开Process Monitor后，设置Filter，规则为Path contains PROXYSERVER：
![_pic2](2.png)

应用后即可查看到有哪些进程在修改我们的http代理了：
![_pic3](3.png)

