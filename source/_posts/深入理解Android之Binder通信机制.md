---
title: 深入理解Android之Binder通信机制
tags: [android,linux]
categories: [android]
date: 2016-04-08 10:26:49
description: 概述、Binder、Server、ServiceManager、Client、Android使用Binder通信的原因
---
最近看《深入理解Android卷I》了解了一些有关于Binder通信的知识，在此写一篇博客作为总结。

# 概述

Binder是Android系统提供的一种IPC（进程间通信）机制。对于Android系统，我们基本上可以把它看做一个基于Binder通信的C/S架构，Binder就像网络一样，把系统的各个部分连接在了一起，因此它是非常重要的。在Android系统的C/S架构中除了Client端和Server端外，还有一个全局的ServiceManager端，其作用是管理系统中的各种服务，三者的关系如下图：
![Binder架构示意图](1.png)

其中：
1. 对于ServiceManager而言：Client与Server都是客户端
2. 对于Client而言：ServiceManager与Server都是服务端
3. 对于Server而言：Client是客户端，而ServiceManager是服务端

Binder提供的作用就是上图中黑线的连接作用。

# Binder

Binder的通信结构分为三层，如下图所示：
![Binder通信结构示意图](2.png)

## 虚拟设备层

在Linux中，Android通过kernel/drivers/staing/android/binder.c文件实现了一个虚拟设备Binder用于通信。由于博主对Linux了解不深，此处不再深入讨论。

## 通信层

通信层主要有两个类：BpBinder客户端代理与BBinder服务器代理，两个类均派生自IBinder类。通信层的主要任务是把业务层传来的数据通过与Binder设备交互发送给目标进程，其最主要的方法为transact方法。以BpBinder为例，其工作示意图如下：
![BpBinder通信示意图](3.png)

从上图可以看到，BpBinder的transact方法把通信的任务交给了IPCThreadState。

IPCThreadState是一个线程私有的变量，它被每个线程存储在TLS（Thread Local Storage）中。IPCThreadState中主要含有mIn和mOut两个缓冲区，分别用于存储从Binder接收的数据及需要发往Binder的数据。

IPCThreadState调用方法talkWithDriver后最终使用Linux的ioctl方法来与Binder设备进行通信。

补充：构造BpBinder时需要传入一个handle数值表示与其对应的BBinder。

## 业务层

业务层主要包括BpServiceXXX和BnServiceXXX，其中XXX会随服务器提供的服务不同而变化，由于业务层经过了层层封装，类关系较为复杂，在此不深入讨论。

# Server

了解Binder是如何在两个进程间进行通信之后，我们来看看C/S架构中Server是如何工作的。Server的工作示意图如下：
![Server的工作示意图](4.png)

下面是对上图的每个步骤的解释：
1. 初始化processState：在初始化的过程中我们打开了binder虚拟设备，并使用mmap为其分配了内存，由于processState是一个用了单例模式实现的类，因此每个进程只会打开设备一次
2. getDefaultServiceManager：顾名思义，获取ServiceManager。由于Server此时是作为客户端，因此得到了BpServiceManager，BpServiceManager中含有BpBinder，其传入的handle为0，代表ServiceManager的BBinder。
3. instantiate：使用BpServiceManager的addService方法注册服务，以字符串标识自己的服务
4. startThreadPool：这是一个可选的操作，当系统认为服务可能较为繁忙时才会创建多个线程，会为每个线程设置IPCThreadState（用于通信），创建完后调用joinThreadPool
5. joinThreadPool：把当前线程加入线程池中，监听来自客户端的请求并处理，得到请求后通过executeCommand方法来处理

# ServiceManager

看完了Server执行的工作后，我们再来看看ServiceManager这个服务总管会做什么。ServiceManager的工作示意图如下：
![ServiceManager的工作示意图](5.png)

ServiceManger的工作只有3步：
1. binder_open：顾名思义，就是打开binder设备，与Server在processState初始化时进行的操作类似
2. binder_become_contextt_manager：通过ioctl把自己的handle值设置为0，代表独一无二的Manager
3. binder_loop：进入一个循环监听请求，并作出响应的处理
 
值得注意的是：不是所有Server进程都能往ServiceManager中注册服务的，只有root或system级别的进程才有注册服务的权限。但ServiceManager中还维护了一个allowed的白名单，上面注明了那些服务是允许被注册的，这些服务可以被任意Server进程注册。

通过上面我们不难发现，Android引入ServiceManager端的主要目的如下：
1. ServiceManager能集中管理系统内的所有服务，它能施加权限控制，规定哪些服务可以注册哪些不可以
2. 允许Client通过字符串名来查找对应的服务，提供一个类似于DNS的功能
3. 由于各种原因Server进程可能生死无常，如果要Client单独去检查未免压力太大，此时Client只需要查询ServiceManager就能知道关于Server进程的最新消息

# Client

有了ServiceManager端与Server端的精心准备后，Client使用服务就简单多了。
Client使用服务只需要分为两步就好：
1. 通过defaultServiceManager方法获取ServiceManager
2. 通过ServiceManager的getService方法传入字符串获取相应的服务并操作

# Android使用Binder通信的原因

## 传输性能方面

常见IPC对比：

| IPC方式 | 数据拷贝次数 |
| - | - |
| 共享内存 | 0 |
| Binder | 1 |
| Socket/管道/消息队列 | 2 |

共享内存虽不需要数据拷贝，但要处理进程间的同步问题，控制复杂，较难使用。因此从传输性能上来看，Binder是个不错的选择。而且Binder基于C/S架构，与共享内存相比，Binder架构清晰明朗，Server端与Client端相对独立，稳定性较好。

## 安全性考虑

传统IPC接收方无法确认发送方的身份（不知道发送方的UID/PID，只能由用户在数据包中填入UID/PID，不安全）。可靠的身份标记只有由IPC机制本身在内核中添加。其次传统IPC访问接入点是开放的，无法建立私有通道。从安全角度，Binder的安全性更高。

## 语言层面的角度

Linux是基于C语言(面向过程的语言)，而Android是基于Java语言(面向对象的语句)，而对于Binder恰恰也符合面向对象的思想，将进程间通信转化为通过对某个Binder对象的引用调用该对象的方法，而其独特之处在于Binder对象是一个可以跨进程引用的对象，它的实体位于一个进程中，而它的引用却遍布于系统的各个进程之中。可以从一个进程传给其它进程，让大家都能访问同一Server，就像将一个对象或引用赋值给另一个引用一样。Binder模糊了进程边界，淡化了进程间通信过程，整个系统仿佛运行于同一个面向对象的程序之中。从语言层面，Binder更适合基于面向对象语言的Android系统。