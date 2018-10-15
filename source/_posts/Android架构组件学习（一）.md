---
title: Android架构组件学习（一）
tags: []
categories: []
description: 前言、ViewModel、Room、LiveData
date: 2018-10-09 21:17:00
---


# 前言

为了帮助广大开发者更好更快的开发出新的app，google推出了Android Jetpack项目
Android Jetpack主页：https://developer.android.com/jetpack/

Android Jetpack 是一套组件、工具和指导，可以帮助我们构建出色的 Android 应用。Android Jetpack 组件将现有的支持库与架构组件联系起来，并将它们分成四个类别
- 基础组件（Foundation）
- 架构组件（Architecture）
- 行为组件（Behavior）
- 视图组件（UI）

Android Jetpack 组件以“未捆绑的”库形式提供，这些库不是基础 Android 平台的一部分。

其中，架构组件又包含以下组件：
- Data Binding：用声明的方式绑定可观察的数据与UI控件
- Lifecycles：管理Activity与Fragment的生命周期
- LiveData：当底层数据改变时通知UI控件
- Navigation：处理所有app内的导航
- Paging：从你的数据源中按需地加载信息
- Room：使SQLite数据库的使用更便捷
- ViewModel：通过生命周期感知的方式来管理UI相关的数据
- WorkManager：管理Android的后台工作

最近，博主学习了官方是样例，研究了其中的ViewModel、Room、LiveData三种组件，在此记录一下

官方指南：https://developer.android.com/jetpack/docs/guide
代码实验室：https://codelabs.developers.google.com/codelabs/build-app-with-arch-components/index.html#0

# ViewModel

个人认为，想要理清楚ViewModel的作用，我们需要从MVC设计模式开始慢慢说起

## MVC

MVC全名是Model View Controller，是模型(model)、视图(view)、控制器(controller)的缩写，一种软件设计典范
其示意图如下所示：
![MVC示意图](1.jpg)

MVC设计模式的核心在于数据与视图的分离，降低代码的耦合程度，提高代码的重用
在管理数据的model层与管理视图的view层实行了分离后，我们可以轻易地为相同的数据提供不同的表现形式
比如相同的统计数据，需要展示成柱状图与饼状图等不同的形式。这种情况下，我们可以在不改动model的情况下，直接拓展view层来实现

## MVP

后来，MVC模式又逐渐发展成了MVP模式
MVP全称：Model-View-Presenter，是从经典的模式MVC演变而来
其示意图如下所示：
![MVP示意图](2.jpg)

在MVC模式中，controller主要的作用在于分离view层中处理数据的逻辑，view层接收用户的输入，并把输入转交给controller，由controller来解析、处理逻辑并通知model数据层进行相应的修改
而当数据发生变动后或者view层需要查找某些数据时，还是直接通过接触model层来获取
这样直接获取固然方便，但有时model的数据获取后并不是直接能用的，因此有些获取数据后的业务逻辑不可避免的被塞在了view层

MVP为了解决这个问题，使得view层回归原本的只关注视图的设计，切断了view层与model层的联系
在MVP中，view层并不与model层打交道，而是通过presenter层来进行数据的获取
这种设计精简了view层的代码，把获取数据后的相关业务逻辑移到了presenter层，使得三部分的职责更分明：
- view层只关心视图相关的部分
- model只关心数据的存取
- presenter层负责相关业务逻辑的处理

## MVVM

MVVM是Model-View-ViewModel的简写，它本质上就是MVC的改进版
其示意图如下所示：
![MVVM示意图](3.jpg)

MVVM模式与MVP模式区别不大，其中的ViewModel层就对应着Presenter层
而我们Android架构组件中的ViewModel也是同样的功能，对应着MVP中的Presenter层，负责处理业务逻辑，以及内存中临时存储一些业务相关的变量

除此以外，官方提供的ViewModel组件主要还帮我们处理了生命周期相关的问题
ViewModel与Activity的生命周期关系如下：
![ViewModel生命周期示意图](4.png)

从图中可以看出，对于屏幕旋转等“非正常”退出的情况下，ViewModel是不会被销毁的，直到Activity正常finish的情况它才销毁

由于我们的view层组件（Activity与Fragment等）的生命周期都是受系统framework层控制的，因此其实它们的状态并不是可控的，系统framework层很可能由于用户的某些行为或一些系统事件把他们销毁掉
在这种情况下，如果我们使用了官方的ViewModel组件，一些业务上的初始化逻辑以及业务相关的变量就能顺利保留下来（PS：进程被回收的情况不包括在内，此处并不涉及数据的持久化）
当我们在重构视图层的时候，我们只需要直接获取数据即可，不需要担心由于视图层的多次“非正常”销毁与重建导致一些初始化逻辑不断地执行或者是状态错乱

具体ViewModel的用法可以参考官方的文档：
https://developer.android.com/topic/libraries/architecture/viewmodel#the_lifecycle_of_a_viewmodel

# Room

为了让我们更好、更安全、更高效地使用SQLite数据库，google推出了Room组件方便我们使用
Room是一种orm框架（orm即object relation mapping，对象关系映射，博主个人的理解是一种：把操作复杂晦涩难用的SQL语句，改为操作简单浅显易用的Object的方法）
它提供了DAO层（即data access object，数据访问对象），让我们通过使用DAO层代替直接使用SQL语句，减少SQL语句的编写，减少出错的几率
同时，它还把SQL语句的检查过程提前到了程序**编译阶段**，方便我们更好更快地发现问题

具体Room的用法可以参考官方的文档：
https://developer.android.com/training/data-storage/room/

# LiveData

LiveData是一款可观察数据的持有组件，与别的观察者不一样的是，该组件是可感知生命周期的
LiveData保证只有在app组件是活跃状态的情况下（即处于STARTED或者是RESUMED状态），才进行观察者的回调，而在不活跃的状态下是不会收到任何通知的
并且当app组件（其实是实现了LifecycleOwner接口的类都可以）转换到了DESTROYED状态后，LiveData会自动注销相关的观察者

稍微总结一下，使用LiveData的好处有：
- **避免内存泄露**：平常我们使用异步回调比较头疼的问题在于，在app组件生命周期结束的时候应该合理的释放回调，因为一般而言界面结束之后并不关心回调的结果了，如果没有合理释放的话，会因为内部类持有外部引用的问题，导致我们生命周期结束的app组件无法被回收，从而造成内存泄露。在使用了LiveData之后，由于其自动注销DESTROYED状态观察者的特性，我们并不用关系这个问题
- **避免一些生命周期相关的问题**：像我们平常遇见比较多的Fragment#getContext为null的问题，以及一些在Activity组件结束后更新其中的UI组件或者dialog弹层等异常崩溃问题，都是由于在app组件非活跃状态下进行了一些不当的UI操作造成的，而这些操作往往是通过异步回调带来的。LiveData只在活跃状态下回调的特性，可以很好地帮助我们避免这些奇奇怪怪的问题
- **保证更新到最新的数据**：LiveData中的数据是有version版本记录的，当它发现注册观察者接收数据的版本低于其version值时，便会尝试回调新数据过去。这种特性能很好地帮助我们在app组件意外销毁后，重建时直接获取数据进行复原（通过使用ViewModel可以使得LiveData的数据再app组件“非正常”销毁的情况下得以保留）

具体LiveData的用法可以参考官方的文档：
https://developer.android.com/topic/libraries/architecture/livedata
