---
title: 关于Android Log类使用的一些反思
tags: [android]
categories: [android]
date: 2017-02-27 22:30:11
description: 日志无法正常输出问题、Log打印异常错误问题
---
最近在使用Android的日志Log类的时候遇到了两个问题，特此写下一篇总结记录：
# 日志无法正常输出问题
经测试发现System.out.print 可以正常打印消息，但是使用Log类却无法打印日志。


解决办法：手机开发者选项中禁用了日志打印的功能，开启即可。


下图为魅族4开启开发者选项中的高级日志输出：


![_pic1](1.jpg)


Coolpad 8720L 手机：


机器在出厂时将log的级别做了限制，方法是：拨号盘输入*20121220#   -&gt;  选择日志输出级别  -&gt;  选择Java log level -&gt; 选择LOGD即可。



# Log打印异常错误问题
在Java中我们经常习惯使用e.printStackTrace 来输出异常，但是在Android中我们应该使用Android提供的Log工具打印日志：

```java
Log.e(TAG, Log.getStackTraceString(e));
```


详见：http://stackoverflow.com/questions/3855187/is-it-a-bad-idea-to-use-printstacktrace-in-android-exceptions


