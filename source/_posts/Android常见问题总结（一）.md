---
title: Android常见问题总结（一）
tags: [android,基础知识]
categories: [android]
date: 2016-04-22 19:34:56
description: Activity与Fragment的生命周期、Acitivty的四种启动模式与特点、Service的生命周期、怎么保证service不被杀死
---
以下为一些常见的Android的总结

# Activity与Fragment的生命周期

Activity的生命周期如下图所示：
![Activity生命周期示意图](1.jpg)

Fragment生命周期如下图所示：
![Fragment生命周期示意图](2.jpg)

# Acitivty的四种启动模式与特点

Android中的Activity由任务栈管理，当我们start一个新的Activity时，就往任务栈中新加入一个栈帧，而当我们finish一个Activity界面时，则往任务栈中移除一个栈帧
Activity具有四种启动模式，我们可以在配置文件中通过修改launchMode修改，启动模式分别是：standard、singleTop、singleTask和singleInstance

## standard

standard为默认Activity的启动模式。在standard启动模式下，无论何时start一个Activity，系统都会往任务栈中加入一个新的栈帧

## singleTop

在singleTop启动模式下，当我们start一个Activity时，系统会先去检测任务栈栈顶的Activity和要启动的Activity是否相同。如果相同则不进行任何操作，否则往任务栈中加入一个新的栈帧

## singleTask

在singleTask启动模式下，当我们start一个Activity时，系统会先去检测任务栈中是否含有将要启动的Activity。如果含有，则把该Activity所在栈帧的顶部的栈帧移除，使该Activity所在的栈帧处在栈顶，如果没有，则新加入一个栈帧

## singleInstance

在singleInstance启动模式下，当我们start一个新的Activity时，该Activity会在一个新的任务栈中启动

# Service的生命周期

Android中的Service组件可以通过startService和bindService两种方法来启动，其生命周期示意图如下：
![Service生命周期示意图](3.png)

如果一个Service同时被调用了startService和bindService方法，那么它的生命周期就变成如下图所示：
![Service两种绑定生命周期示意图](4.png)

# 怎么保证service不被杀死

要想使Service存活下来，我们就必须保证Service所在的进程不被杀掉，一般来说有以下方法：
1. 在onStartCommand回调方法中返回START_STICKY，那么该进程被杀掉后系统会试图重启它
2. 设置配置文件中application的persistent属性，把应用提升为系统级别应用，免疫low memory killer
3. 在Service的onDestroy方法中重启该Service，不过如果进程被直接杀掉这种方法就无效了
4. 通过监听特殊的系统广播（如屏幕变化、电量变化、网络变化等）去不断重启Service
5. 使用AlarmManager定时重复开启Service
6. 通过设置Service的process属性，把Service放在子进程中，避免与主进程一起被回收
7. 开启一个另外的进程与Service进程互相监视，双方要是有任意一方被杀掉则重启