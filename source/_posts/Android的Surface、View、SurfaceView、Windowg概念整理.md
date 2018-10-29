---
title: Android的Surface、View、SurfaceView、Window概念整理
tags: [android,view]
categories: [android]
date: 2018-10-23 19:35:39
description: BufferQueue、SurfaceFlinger、Surface、View
---

最近了解了一下Android中几个关于视图的概念：Surface、View、SurfaceView与Window，在此进行一下总结整理

# BufferQueue

在此之前，我们先介绍一下BufferQueue。
BufferQueue类是 Android 中所有图形处理操作的核心。它的作用很简单：将生成图形数据缓冲区的一方（生产方）连接到接受数据以进行显示或进一步处理的一方（消耗方）。几乎所有在系统中移动图形数据缓冲区的内容都依赖于BufferQueue。
简而言之，BufferQueue是图形生产者消费者模型中沟通的桥梁。
它的基本用法很简单：生产方请求一个可用的缓冲区 (dequeueBuffer())，并指定一组特性，包括宽度、高度、像素格式和用法标记。生产方填充缓冲区并将其返回到队列 (queueBuffer())。随后，消耗方获取该缓冲区 (acquireBuffer()) 并使用该缓冲区的内容。当消耗方操作完毕后，将该缓冲区返回到队列 (releaseBuffer())。

更详细介绍可参考官方文档：
https://source.android.com/devices/graphics/arch-bq-gralloc

# SurfaceFlinger

在介绍Surface之前，我们还需要先了解一下SurfaceFlinger。
SurfaceFlinger的作用是接受来自多个来源的数据缓冲区，对它们进行合成，然后发送到显示设备。
当别的服务向SurfaceFlinger请求一个绘图Surface时，它会创建一个主要组件是BufferQueue的新的层（这里的层博主感觉指的是一层层的视图的意思），同时把自己设定为消耗方。大多数应用通常在屏幕上有三个层：屏幕顶部的状态栏、底部或侧面的导航栏以及应用的界面。
我们可以通过adb的指令来打印当前屏幕的层信息：
```
adb shell dumpsys SurfaceFlinger
```
例，博主的某个应用的信息：
![屏幕层的信息](1.png)

从上面的表格不难看出，博主当前的屏幕中包含3个层（自底向上）：
- 应用的MainActivity视图层
- StatusBar系统的状态栏视图层
- HWC_FRAMEBUFFER_TARGET（目前不清楚是什么）

更详细介绍可参考官方文档：
https://source.android.com/devices/graphics/arch-sf-hwc

# Surface

从 1.0 开始，Surface类一直是公共 API 的一部分。它的描述简单如下：“在由屏幕合成器管理的原始缓冲区上进行处理”。
Surface表示通常（但并非总是！）由SurfaceFlinger消耗的缓冲区队列的生产方。当您渲染到Surface上时，产生的结果将进入相关缓冲区，该缓冲区被传递给消耗方。
简单来说，博主认为我们可以把Surface当做是一层视图，我们通过在Surface上绘制图像，来往其中的BufferQueue生产视图数据，进而交给其消耗方SurfaceFlinger来与其他视图层合成，最终显示到屏幕上。

更详细介绍可参考官方文档：
https://source.android.com/devices/graphics/arch-sh
https://developer.android.com/reference/android/view/Surface

# Window

Window类代表着一个视图的顶层窗口，它管理着这个视图中最顶层的View，提供了一些背景、标题栏、默认按键等标准的UI处理策略。
同时，它拥有唯一一个用以绘制自己的内容的Surface，当app通过WindowManager创建一个Window时，WindowManager会为每一个Window创建一个Surface，并把该Surface传递给app以便应用在上面绘制内容。
博主认为Window与Surface之前存在着一对一的关系。

更详细介绍可参考官方文档：
https://developer.android.com/reference/android/view/Window

# View

View这个类相信大家都普遍比较熟悉，它是Android视图中的一个基本单位。
它代表着屏幕上的一个矩形区域，并负责绘制该区域以及处理区域上的点击事件。
它与ViewGroup通过组合设计模式，共同组成了Android视图的基石，通过最顶层的ViewRootImpl我们可以一层层遍历View组成的树状结构。
最顶层的View会由某个Window进行打理，在对其复杂的层级结构进行测量、布局、绘制后，其中的所有内容最终会绘制到Window拥有的Surface中去。

更详细介绍可参考官方文档：
https://developer.android.com/reference/android/view/View

# SurfaceView

SurfaceView是一种被特殊实现的View，它拥有自己专门的一个Surface，以便让应用直接在里面绘制内容。
它能与普通的View一样，在View组成的树状结构中进行测量与布局，但却独立于普通View所共享Window的那个Surface。它在需要渲染时，内容会变得完全透明，SurfaceView的View部分只是一个透明的占位符。
SurfaceView的工作原理如下，它所做的全部就是要求WindowManager创建一个Window，并告诉WindowManager所创建的Window的Z轴顺序（Z-order），这个Z轴顺序可以帮助WindowManager决定将新建的window置于SurfaceView所属Window的前面还是后面。
然后，WindowManager会将新建的Window放置到SurfaceView在所属Window中的位置。如果新建Window在SurfaceView所属Window后面，SurfaceView会将它在所属Window中占据的部分变透明，以便让后面的Window显示出来。

更详细介绍可参考官方文档：
https://source.android.com/devices/graphics/arch-sv-glsv
https://developer.android.com/reference/android/view/SurfaceView
