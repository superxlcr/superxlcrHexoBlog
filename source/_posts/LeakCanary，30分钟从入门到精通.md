---
title: LeakCanary，30分钟从入门到精通
tags: [android,java,内存]
categories: [java]
date: 2018-12-18 20:12:29
description: （转载）简述、用法简介、核心类分析、结论、补充
---

本文转载自：https://www.jianshu.com/p/1e7e9b576391

# 简述

在性能优化中，内存是一个不得不聊的话题；然而内存泄漏，显示已经成为内存优化的一个重量级的方向。当前流行的内存泄漏分析工具中，不得不提的就是LeakCanary框架；这是一个集成方便， 使用便捷，配置超级简单的框架，实现的功能却是极为强大的。

# 用法简介

## 你需要添加到配置的只有这个

```java
dependencies {
	debugCompile 'com.squareup.leakcanary:leakcanary-android:1.3'
	releaseCompile 'com.squareup.leakcanary:leakcanary-android-no-op:1.3'
}
```

## 你肯定需要初始化一下，当然，推荐在Application中

```java
public class MyApplicationextends Application {
	@Override
	public void onCreate() {
		super.onCreate();
		LeakCanary.install(this);
	}
}
```

## 什么？你还在下一步？已经结束了！通过以上配置，你就可以轻松使用LeakCanary检测内存泄漏了

关于LeakCanary的详细使用教程，建议去看：[LeakCanary 中文使用说明](https://www.liaohuqiu.net/cn/posts/leak-canary-read-me/)


闲聊结束，我想你们来肯定不是看我这些废话的，那么现在进入正题！本篇我们所想的，就是LeakCanary为什么可以这么神奇，它是怎么检测内存泄漏的？下面我们解开谜题！

# 核心类分析

| 类 | 作用 |
| - | - |
| LeakCanary | SDK提供类 |
| DisplayLeakActivity | 内存泄漏的查看页面 |
| HeapAnalyzerService | 内存堆分析服务，为了保证App进程不会因此受影响变慢&内存溢出，运行于独立的进程 |
| HeapAnalyzer | 分析由RefWatcher生成的堆转储信息， 验证内存泄漏是否真实存在 |
| HeapDump | 堆转储信息类，存储堆转储的相关信息 |
| ServiceHeapDumpListener | 一个监听，包含了开启分析的方法 |
| RefWatcher | 核心类， 翻译自官方： 检测不可达引用（可能地），当发现不可达引用时，它会触发HeapDumper(堆信息转储) |
| ActivityRefWatcher | Activity引用检测， 包含了Activity生命周期的监听执行与停止 |

通过以上列表，让大家对LeakCanary框架的主要类有个大体的了解，并基于以上列表，对这个框架的大体功能有一个模糊的猜测。

漫无目的的看源码，很容易迷失在茫茫的Code Sea中，无论是看源码，还是接手别人的项目，都是如此；因此，带着问题与目的性来看这些复杂的东西是很有必要的，也使得我们阅读效率大大提高；想要了解LeakCanary,我们最大的疑惑是什么，我列出来，看看是与你不约而同。

- Question1:在Application中初始化之后，它是如何检测所有的Activity页面的？
- Question2:内存泄漏的判定条件是什么 ？检测内存泄漏的机制原理是什么？
- Question3:检测出内存泄漏后，它又是如何生成泄漏信息的？内存泄漏的输出轨迹是怎么得到的？

回顾一下这个框架，其实我们想了解的机制不外乎三：
1. 内存泄漏的检测机制
2. 内存泄漏的判定机制
3. 内存泄漏的轨迹生成机制

我们会在源码分析最后，依次回答以上的三个问题，可能在阅读源码之前，我们先要对内存泄漏做一些基础概念与原理的理解。

## 什么是内存泄漏（MemoryLeak）？

大家对这个概念应该不陌生吧，当我们使用一个Bitmap,使用完成后，没有recycle回收；当我们使用Handler, 在Activity销毁时没有处理；当我们使用Cursor，最后没有close并置空;以上这些都会导致一定程度上的内存泄漏问题。那么，什么是内存泄漏？

内存泄漏（Memory Leak）是指程序中己动态分配的堆内存由于某种原因程序未释放或无法释放，造成系统内存的浪费，导致程序运行速度减慢甚至系统崩溃等严重后果。

以上是百度百科的解释，总结下为：内存泄漏是不使用或用完的内存，因为某些原因无法回收，造成的一种内存浪费；内存泄漏的本质是内存浪费。以个人理解来解释，通俗一点就是

1. GC回收的对象必须是当前没有任何引用的对象
2. 当对象在使用完成后（对我们而言已经是垃圾对象了）， 我们没有释放该对象的引用，导致GC不能回收该对象而继续占用内存
3. 垃圾对象依旧占用内存，这块内存空间便浪费了

## 内存泄漏与内存溢出的区别是什么？

从名称来看，一个泄漏，一个溢出，其实很好理解。

- 内存泄露：垃圾对象依旧占据内存，如水龙头的泄漏，水本来是属于水源的， 但是水龙头没关紧，那么泄漏到了水池；再来看内存，内存本来应该被回收，但是依旧在内存堆中；总结一下就是内存存在于不该存在的地方（没用的地方）
- 内存溢出：内存占用达到最大值，当需要分配内存时，已经没有内存可以分配了，就是溢出；依旧以水池为例， 水池的水如果满了，那么如果继续需要从水龙头流水的话，水就会溢出。总结一下就是，内存的分配超出最大阀值，导致了一种异常

## 明白了两者的概念，那么两者有什么关系呢？

内存的溢出是内存分配达到了最大值，而内存泄漏是无用内存充斥了内存堆；因此内存泄漏是导致内存溢出的元凶之一，而且是很大的元凶；因为内存分配完后，哪怕占用再大，也会回收，而泄漏的内存则不然；当清理掉无用内存后，内存溢出的阀值也会相应降低。

## JVM如何判定一个对象是垃圾对象？

该问题也即垃圾对象搜索算法，JVM采用图论的可达遍历算法来判定一个对象是否是垃圾对象， 如果对象A是可达的，则认为该对象是被引用的，GC不会回收；如果对象A或者块B（多个对象引用组成的对象块）是不可达的，那么该对象或者块则判定是不可达的垃圾对象，GC会回收。

![pic1](1.jpg)

以上科普的两个小知识：1） 内存泄漏  2） JVM搜索算法 是阅读LeakCanary源码的基本功，有助于源码的理解与记忆。

好了，下面来看一下LeakCanary的源码，看看LeakCanary是怎么工作的吧！既然LeakCanary的初始化是从install()开始的，那么从init开始看

![pic2](2.png)

![pic3](3.png)

回顾一下核心类模块可知，内存分析模块是在独立进程中执行的，这么设计是为了保证内存分析过程不会对App进程造成消极的影响，如使App进程变慢或导致out of Memory问题等。因此

第一步： 判断APP进程与内存分析进程是否属于同一进程；如果是， 则返回空的RefWatcher DISABLED;如果不是，往下走，看第二步

![pic4](4.png)

是否与分析进程是同一个，isInAnalyzerProcess

![pic5](5.png)

与分析进程一致，返回一个空的DISABLED

第二步： enableDisplayLeakActivity 开启显示内存泄漏信息的页面

![pic6](6.png)

调用了setEnabled,记住参数DisplayLeakActivity就是查看泄漏的页面，enable传true,开启Activity组件

![pic7](7.png)

这个方法的作用是设置四大组件开启或禁用，从上图传的参数看，是开启了查看内存泄漏的页面

第三步：初始化一个ServiceHeapDumpListener，这是一个开启分析的接口实现类，类中定义了analyze方法，用于开启一个DisplayLeakService服务，从名字就可以看出，这是一个显示内存泄漏的辅助服务

![pic8](8.png)

看注释，这个服务的作用是分析HeapDump，写入一个记录文件，并弹出一个Notification

第四步：初始化两个Watcher, RefWatcher和ActivityRefWatcher. 这两个Watcher的作用分别为分析内存泄漏与监听Activity生命周期

![pic9](9.png)

ActivityRefWatcher监听Activity生命周期，在初始化时开始监听Activity生命周期（watchActivities）

![pic10](10.png)

watchActivities中注册了所有Activity的生命周期统一监听；onActiityDestroy会在onDestroy时执行，执行watch，检测内存泄漏

**通过以上代码分析，我们可以得出第一个问题的答案。LeakCanary通过ApplicationContext统一注册监听的方式，来监察所有的Activity生命周期，并在Activity的onDestroy时，执行RefWatcher的watch方法，该方法的作用就是检测本页面内是否存在内存泄漏问题。**

下面我们继续来分析核心类RefWatcher中的源码，检测机制的核心逻辑便在RefWatcher中；相信阅读完这个类后，第二个问题的答案便呼之欲出了。

既然想弄明白RefWatcher做了什么，那么先来看一下官方的解释

![pic11](11.png)

监听可能不可达的引用，当RefWatcher判定一个引用可能不可达后，会触发HeapDumper（堆转储）

从上面图可以看出官方的解释。 RefWatcher是一个引用检测类，它会监听可能会出现泄漏（不可达）的对象引用，如果发现该引用可能是泄漏，那么会将它的信息收集起来（HeapDumper）
.从RefWatcher源码来看，核心方法主要有两个： watch() 和 ensureGone()。如果我们想单独监听某块代码，如fragment或View等，我们需要手动去调用watch()来检测；因为上面讲过，默认的watch()仅执行于Activity的Destroy时。watch（）是我们直接调用的方法，ensureGone（）则是具体如何处理了，下面我们来看一下 

![pic12](12.png)

watch 检测核心方法

上图为watch()的源码, 我们先来看一下官方的注释监听提供的引用，检查该引用是否可以被回收。这个方法是非阻塞的，因为检测功能是在Executor中的异步线程执行的

从上述源码可以看出，watch里面只是执行了一定的准备工作，如判空（checkNotNull）, 为每个引用生成一个唯一的key, 初始化KeyedWeakReference;关键代码还是在watchExecutor中异步执行。引用检测是在异步执行的，因此这个过程不会阻塞线程。

![pic13](13.png)

检测核心代码 gone()判定WeakReference中是否包含当前引用

以上是检测的核心代码实现，从源码可以看出，检测的流程：
1. 移除不可达引用，如果当前引用不存在了，则不继续执行
2. 手动触发GC操作，gcTrigger中封装了gc操作的代码
3. 再次移除不可达引用，如果引用不存在了，则不继续执行
4. 如果两次判定都没有被回收，则开始分析这个引用，最终生成HeapDump信息

总结一下原理：
1. 弱引用与ReferenceQueue联合使用，如果弱引用关联的对象被回收，则会把这个弱引用加入到ReferenceQueue中；通过这个原理，可以看出removeWeaklyReachableReferences()执行后，会对应删除KeyedWeakReference的数据。如果这个引用继续存在，那么就说明没有被回收。
2. 为了确保最大保险的判定是否被回收，一共执行了两次回收判定，包括一次手动GC后的回收判定。两次都没有被回收，很大程度上说明了这个对象的内存被泄漏了，但并不能100%保证；因此LeakCanary是存在极小程度的误差的。

上面的代码，总结下流程就是

1. 判定是否回收（KeyedWeakReference是否存在该引用）， Y -> 退出， N -> 向下执行
2. 手动触发GC
3. 判定是否回收， Y -> 退出， N-> 向下执行
4. 两次未被回收，则分析引用情况：
5. humpHeap :  这个方法是生成一个文件，来保存内存分析信息
6. analyze: 执行分析

通过以上的代码分析，第二个问题的答案已经浮出水面了吧！

接下来分析内存泄漏轨迹的生成~

最终的调用，是在RefWatcher中的ensureGone()中的最后，如图

![pic14](14.png)

分析最终调用，在ensureGone()中
很明显，走的是heapdumpListener中的analyze方法，继续追踪heapdumpListener是在LeakCanary初始化的时候初始化并传入RefWatcher的，如图

![pic15](15.png)

在install中初始化并传入RefWatcher
打开进入ServiceHeapDumpListener,看里面实现，如图

![pic16](16.png)

ServiceHeapDumpListener中的analyze
              
调用了HeapAnalyzerService，在单独的进程中进行分析，如图 

![pic17](17.png)

HeapAnalyzerService分析进程
                       
HeapAnalyzerService中通过HeapAnalyzer来进行具体的分析，查看HeapAnalyzer源码，如图

![pic18](18.png)

HeapAnalyzer

进行分析时，调用了openSnapshot方法，里面用到了SnapshotFactory

![pic19](19.png)

org.eclipse.mat

从上图可以看出，这个版本的LeakCanary采用了MAT对内存信息进行分析，并生成结果。其中在分析时，分为findLeakingReference与findLeakTrace来查找泄漏的引用与轨迹，根据GCRoot开始按树形结构依次建议当前引用的轨迹信息。

# 结论

通过上述分析，最终得出的结果为：

## Activity检测机制是什么？

答： 通过application.registerActivityLifecycleCallbacks来绑定Activity生命周期的监听，从而监控所有Activity; 在Activity执行onDestroy时，开始检测当前页面是否存在内存泄漏，并分析结果。因此，如果想要在不同的地方都需要检测是否存在内存泄漏，需要手动添加。

## 内存泄漏检测机制是什么？

答： KeyedWeakReference与ReferenceQueue联合使用，在弱引用关联的对象被回收后，会将引用添加到ReferenceQueue；清空后，可以根据是否继续含有该引用来判定是否被回收；判定回收， 手动GC, 再次判定回收，采用双重判定来确保当前引用是否被回收的状态正确性；如果两次都未回收，则确定为泄漏对象。

## 内存泄漏轨迹的生成过程 ？

答： 该版本采用eclipse.Mat来分析泄漏详细，从GCRoot开始逐步生成引用轨迹。

通过整篇文章分析，你还在疑惑么？

# 补充

最近微信开源了[Matrix](https://github.com/Tencent/matrix)性能检测工具，里面也使用到了LeakCanary，不过同时还进行了一定的改进，主要有下面几点：

1. GC不一定成功问题，改进为：通过一个一定能被回收的“哨兵”对象，用来确认系统确实进行了GC
2. 对象加入弱引用ReferenceQueue有延迟问题，改进为：直接通过WeakReference#get()判断对象是否被成功回收
3. 判断Activity是否可回收时其正好还被局部变量持有引起误判，改进为：多次判断，且重复创建才确认泄露
4. 改进了相同Activity重复泄露多次提醒的问题
5. 对生成的hprof文件进行了一定的裁剪

详情可参考文章：https://cloud.tencent.com/developer/article/1379397
