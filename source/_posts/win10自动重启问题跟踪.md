---
title: win10自动重启问题跟踪
tags: [杂项]
categories: [杂项]
date: 2020-09-07 20:01:55
description: 开关机记录、客户体验改善计划、关闭自动更新
---

作为程序员，不关电脑是很正常的一件事

因为有时需要趁着下班跑一些耗时长的任务，或者因为开启浏览页面以及IDE等需要耗费大量的时间，博主也是经常会不关电脑只锁屏

不过最近博主的电脑频繁被重启，就搞得整个人很烦，因此开始尝试跟踪是什么原因导致的

# 开关机记录

根据博客：https://blog.csdn.net/wangmx1993328/article/details/86298066

博主在一次重启之后，搜索了相关的开关机事件，看见了在关机事件中，备注的

**客户体验改善计划的用户注销通知**

因此开始搜索如何关闭这玩意

# 客户体验改善计划

根据网上找到的资料：
https://answers.microsoft.com/zh-hans/windows/forum/all/%E5%AE%A2%E6%88%B7%E4%BD%93%E9%AA%8C%E6%94%B9/a504cc6d-0d8d-433a-b076-a51c21afda8e

博主关掉了客户体验改善计划

顺便提一下，打开组策略相关的资料：
https://windows10.pro/win10-group-policy/

但是之后还是出现了电脑被重启的现象，因此根据网上的说法，博主决定还需要关掉win10的自动更新

# 关闭自动更新

根据网上找到的资料：https://jingyan.baidu.com/article/5225f26b620438e6fa09080c.html

博主关闭了自动更新

到此为止，电脑被重启的现象有所好转

不过某天博主发现电脑还是被重启了，查了下，好像是win10的自动更新**只能关闭特定的时长，过一段时间后还会自动启动，还需要重新关闭……**