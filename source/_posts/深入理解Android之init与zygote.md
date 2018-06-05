---
title: 深入理解Android之init与zygote
tags: [android,linux]
categories: [android]
date: 2016-04-04 19:31:19
description: init进程、zygote进程、system_server进程
---
最近看了《深入理解Android卷I》中关于init进程与zygote进程的知识，特此写一篇博客记录一下收获。

# init进程

init进程是Android系统中的用户空间的第一个进程，它的进程号是1，作为天字第一号进程，init进程需要完成很多工作，其示意图如下：
![init进程工作流程图](1.png)

init进程的工作主要分为四步：
1. 解析配置文件：解析init,rc与init,硬件名.rc（每种类型的手机有自己的硬件名），根据解析的内容执行第二步操作
2. 执行各阶段操作：各种初始化操作，主要分为early-init、init、early-boot和boot四个阶段（其中zygote进程是在boot阶段通过fork与execv启动的）
3. 启动属性服务：属性服务类似于Windows的注册表，可以保存一些设备上的键值对信息。主要分为在共享内存上初始化和循环处理请求两部分
4. 死循环：完成以上操作后，init进程进入一个死循环检测系统中死去的进程，并负责把他们重启和进行收尾工作

# zygote进程

讲完init进程我们再来提一下著名的zygote进程。zygote即受精卵，他与system_server进程分别是Android中Java世界的半边天。但zygote进程刚开始并不叫zygote进程，而是app_process进程，他的工作流程示意图如下：
![zygote进程工作流程图](2.png)

zygote进程的工作主要分为7个步骤：
1. 改名：进程的名字从app_process改为zygote
2. 创建VM：创建虚拟机，并设置虚拟机的堆大小（默认16M）等参数
3. 注册JNI函数：由于zygoteInit类是由Java代码编写而成的，需要调用一些native方法，因此在此注册一些JNI函数
4. 建立IPC的Socket服务器端（从此处开始是Java世界了）
5. 预加载类与资源：此处需要加载1200多个类与大量系统自带的资源（com.android.R.XXX等），因此比较耗时
6. 启动system_server进程：通过fork方法启动系统服务进程
7. runSelectLoopMode：完成上述工作后zygote初始化的工作已基本完成，只需要监听Socket的消息处理IPC请求即可

zygote作为Android系统中创建Java世界的盘古，创建了第一个Java虚拟机，预加载了大量的Java类与Android核心资源，启动了Android系统服务进程。他进行一系列的初始化操作就是通过牺牲开机时间，来减少Android应用启动的时间。当我们启动一个Android应用时，通过IPC通信向zygote进程发送请求，然后zygote进程通过fork方法复制自身来为应用创建一个进程，由于zygote进程已经预加载了大量的类与资源，因此我们启动应用的时间将会显著提升。

# system_server进程

system_server进程是zygote进程通过fork方法创造出来的第一个子进程，而且当system_server进程启动失败时会导致zygote进程自杀重启，因此其重要性不言而喻。那么system_server进程进行了那些工作能呢？我们还是以示意图的方式来看一看：
![system_server进程工作流程图](3.png)

system_server进程主要进行了以下工作：
1. 初始化Native层，与Binder通信系统建立联系（目前还没看到Binder通信系统，以后补充）
2. 调用SystemServer类的main方法（Java层）
3. 加载其动态链接库，注册JNI函数
4. 调用Init1方法（native方法）：创建了一些关键的系统服务，并把调用线程加入了Binder
5. 调用Init2方法（Java方法）：创建了一个线程，启动了大量的系统服务，并在最后调用了Looper.loop方法