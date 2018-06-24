---
title: 一个Activity的显示过程总结（三）
tags: [android,源码]
categories: [android]
date: 2016-05-15 11:14:28
description: ViewRoot、ViewRootImpl#setView
---
有兴趣自己看Android源码的同学可以前往：
http://grepcode.com/project/repository.grepcode.com/java/ext/com.google.android/android/
本博客分析的Android版本为4.4

上一篇博客传送门：[一个Activity的显示过程总结（二）](/2016/05/13/一个Activity的显示过程总结（二）/)

上次我们追踪源码，分析到了ViewRoot这个关键的对象，接下来我们就从ViewRoot说起吧（ViewRoot貌似是android 2.x时候的说法了，现在变成了ViewRootImpl）：

# ViewRoot

首先我们先来说明一下，ViewRoot是什么？
官方的注释如下：
```
The top of a view hierarchy, implementing the needed protocol between View and the WindowManager.
```
翻译为：View层次的顶端，实现View和WindowManager之间所需的协议。ViewRoot是由View组成的视图树状结构的“根”，但它并不处理绘画，而是处理有关于整个树状结构的事情（如遍历初始化、点击分发等）。

先来看看ViewRoot中的关键对象和构造函数：
（android.view.ViewRootImpl）
```java
final IWindowSession mWindowSession;  
final W mWindow;  
View mView;  
// These can be accessed by any thread, must be protected with a lock.  
// Surface can never be reassigned or cleared (use Surface.clear()).  
private final Surface mSurface = new Surface();  
  
public ViewRootImpl(Context context, Display display) {  
    mContext = context;  
    mWindowSession = WindowManagerGlobal.getWindowSession();  
    ...  
    mWindow = new W(this);  
    ...  
}  
```

关键对象有这几个：
1. mWindowSession：IWindowSession类型对象，用于Binder通信，在ViewRoot中表示WindowManagerService，用以调用WindowManagerService的服务
2. mWindow：W类型对象（实际为IWindow.Stub类型），同样用于Binder通信，用以由WindowManagerService发送消息给ViewRoot
3. mView：保存我们的DecorView
4. mSurface：Surface对象，官方的注释为：Handle onto a raw buffer that is being managed by the screen compositor。即它的职责是管理一块用于绘制的缓存区，因此Surface也就是我们绘制时的画布

用一幅图大致说明ViewRoot的关键对象：
![ViewRoot对象说明](1.png)

由于IWindowSession、IWindow以及Surface涉及JNI以及Native层的代码，这里就不继续深入分析了。

# ViewRootImpl#setView

接下来我们来看看ViewRoot的setView方法：
（android.view.ViewRootImpl）
```java
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {  
    synchronized (this) {  
        if (mView == null) {  
            mView = view;  
            ...  
  
            // Schedule the first layout -before- adding to the window  
            // manager, to make sure we do the relayout before receiving  
            // any other events from the system.  
            requestLayout();  
            ...  
        }  
    }  
}  
```

setView方法首先把我们传入的DecorView赋给了mView，然后调用了requestLayout方法：
（android.view.ViewRootImpl）
```java
public void requestLayout() {  
    if (!mHandlingLayoutInLayoutRequest) {  
        checkThread();  
        mLayoutRequested = true;  
        scheduleTraversals();  
    }  
}  
```

又调用了scheduleTraversals方法：
（android.view.ViewRootImpl）
```java
void scheduleTraversals() {  
    if (!mTraversalScheduled) {  
        mTraversalScheduled = true;  
        mTraversalBarrier = mHandler.getLooper().postSyncBarrier();  
        mChoreographer.postCallback(  
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);  
        scheduleConsumeBatchedInput();  
    }  
}  
```

在该方法的第6行，传入了一个关键的Runnable对象mTraversalRunnable作为回调，我们来看看它的run方法：
（android.view.ViewRootImpl）
```java
final class TraversalRunnable implements Runnable {  
    @Override  
    public void run() {  
        doTraversal();  
    }  
}  
final TraversalRunnable mTraversalRunnable = new TraversalRunnable();  
```

调用了doTraversal方法：
（android.view.ViewRootImpl）
```java
void doTraversal() {  
    if (mTraversalScheduled) {  
        ...  
        try {  
            performTraversals();  
        ...  
    }  
}  
```

doTraversal调用了关键的performTraversals方法：
（android.view.ViewRootImpl）
```java
private void performTraversals() {  
    // cache mView since it is used so much below...  
    final View host = mView; // DecorView  
  
    ...  
  
                 // Ask host how big it wants to be  
                performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);  
  
                ...  
        performLayout(lp, desiredWindowWidth, desiredWindowHeight);  
  
       ...  
            performDraw();  
       ...  
}  
```

原代码非常长，我挑出了3个最主要的方法：
1. performMeasure：与测量View的大小相关
2. performLayout：与View的布局有关
3. performDraw：与View的绘制相关

这三个方法与我们涉及View绘制过程时耳熟能详的三部曲Measure、Layout、Draw相关，由于这几个函数间的关系还是比较复杂的，因此我们就留到下一篇博客再分析它们了。最后还是以我们的分析路线结束本篇博客：
![分析路线图](2.png)