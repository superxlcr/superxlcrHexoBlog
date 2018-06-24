---
title: 一个Activity的显示过程总结（二）
tags: [android,源码]
categories: [android]
date: 2016-05-13 16:28:29
description: callActivityOnCreate、installDecor、inflate、handleResumeActivity
---
有兴趣自己看Android源码的同学可以前往：
http://grepcode.com/project/repository.grepcode.com/java/ext/com.google.android/android/
本博客分析的Android版本为4.4

上一篇博客传送门：[一个Activity的显示过程总结（一）](/2016/05/12/一个Activity的显示过程总结（一）/)

上次我们追踪源码分析到了android.app.Activity的attach方法中，接下来我们来看看下一个关键的callActivityOnCreate方法：

# callActivityOnCreate

（android.app.Instrumentation）
```java
public void callActivityOnCreate(Activity activity, Bundle icicle) {  
    ...  
      
    activity.performCreate(icicle);  
      
    ...  
}  
```

调用了Activity的performCreate方法：
（android.app.Activity）
```java
final void performCreate(Bundle icicle) {  
    onCreate(icicle);  
    mVisibleFromClient = !mWindow.getWindowStyle().getBoolean(  
            com.android.internal.R.styleable.Window_windowNoDisplay, false);  
    mFragments.dispatchActivityCreated();  
}  
```

调用了我们熟悉的Activity生命周期里的onCreate，在onCreate我们最熟悉的对界面初始化的函数绝对非setContentView莫属了，接下来我们一起看一下setContentView：
（android.app.Activity）
```java
public void setContentView(int layoutResID) {  
    getWindow().setContentView(layoutResID);  
    initActionBar();  
}  
```

getWindow方法返回了什么？
（android.app.Activity）
```java
	public Window getWindow() {  
        return mWindow;  
    }  
```

原来setContentView方法把工作委托给了mWindow的setContentView方法，从上一篇博客我们得知mWindow的实际类型是PhoneWindow，下面来看看PhoneWindow的setContentView方法：
（com.android.internal.policy.impl.PhoneWindow）
```java
public void setContentView(int layoutResID) {  
    if (mContentParent == null) {  
        installDecor();  
    } else {  
        mContentParent.removeAllViews();  
    }  
    mLayoutInflater.inflate(layoutResID, mContentParent);  
    final Callback cb = getCallback();  
    if (cb != null && !isDestroyed()) {  
        cb.onContentChanged();  
    }  
}  
```

在这个方法里有两个关键点：
1. 第3行的installDecor：第一次调用setContentView时初始化顶层的DecorView
2. 第7行的inflate：把我们传入setContentView inflate并置入mContentParent中

先来分析installDecor

# installDecor

首先我们先来了解一下什么是DecorView：DecorView是PhoneWindow的一个内部类，继承自FrameLayout：
（com.android.internal.policy.impl.PhoneWindow）
```java
private final class DecorView extends FrameLayout implements RootViewSurfaceTaker {  
```

DecorView是一个非常重要的ViewGroup容器，因为它是我们Activity的第一个View，是Activity最顶层的ViewGroup。DecorView的设计使用了装饰器模式，我们setContentView设置的View其实只是DecorView的子View，DecorView通过包装在我们设置的View的外部，为我们的Activity提供了ActionBar、标题栏等控件。也即我们的Activity的UI层级可以这样表示：
![Activity的UI层级图](1.png)

接下来一起来看看installDecor方法：
（com.android.internal.policy.impl.PhoneWindow）
```java
private void installDecor() {  
    if (mDecor == null) {  
        mDecor = generateDecor();  
        ...  
    }  
    if (mContentParent == null) {  
        mContentParent = generateLayout(mDecor);  
  
        // Set up decor part of UI to ignore fitsSystemWindows if appropriate.  
        mDecor.makeOptionalFitsSystemWindows();  
  
        mTitleView = (TextView)findViewById(com.android.internal.R.id.title);  
		// 设置Title和ActionBar，涉及许多Feature参数，因此对Activity的requestWindowFeature方法应该在setContentView前完成  
        ...  
    }  
}  
protected DecorView generateDecor() {  
    return new DecorView(getContext(), -1);  
}  
```

首先我们看到第3行的generateDecor方法，这个方法创建了一个DecorView对象并赋给了mDecor。然后我们再看到第7行，这里有一个关键的generateLayout方法，设置了非常关键的mContentParent变量：
（com.android.internal.policy.impl.PhoneWindow）
```java
public static final int ID_ANDROID_CONTENT = com.android.internal.R.id.content;  
protected ViewGroup generateLayout(DecorView decor) {  
    // WindowFeature相关设置  
    ...  
  
    View in = mLayoutInflater.inflate(layoutResource, null);  
    decor.addView(in, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));  
  
    ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);  
    // 背景相关设置  
    ...  
  
    return contentParent;  
}  
```

在该方法的第9行，使用了findViewById方法返回了我们需要的contentParent，id为第一行所示的android.R.id.content，该方法定义在Window基类中：
（android.view.Window）
```java
public View findViewById(int id) {  
    return getDecorView().findViewById(id);  
} 
```

其实就是返回DecorView中的id为android.R.id.content的ViewGroup（其实是一个FrameLayout）

# inflate

接下来让我们回到setContentView中第7行的inflate方法，调用该方法时我们传入了我们设置layout的id和id为android.R.id.content的FrameLayout。由于inflate是个比较常见的方法，这里就不深入分析了，该方法执行的结果是：我们inflate了我们设置的layout，并把它加入了id为android.R.id.content的FrameLayout中，此时Activity的UI变为：
![Activity的UI](2.png)

万万没想到原来Activity的界面是如此复杂的啊~有兴趣的朋友还可以使用SDK的tools目录下的hierarchyviewer工具打开一个Activity来查看一下是不是这样。

到这里Activity的onCreate方法的分析就结束了，先让我们看一看我们的分析路线：
![分析路线图](3.png)

# handleResumeActivity

看来是时候回到ActivityThread类中，解决遗留下来的handleResumeActivity方法了：
（android.app.ActivityThread）
```java
final void handleResumeActivity(IBinder token, boolean clearHide, boolean isForward,  
        boolean reallyResume) {  
    ...  
  
    // 与调用Activity的onResume相关  
    ActivityClientRecord r = performResumeActivity(token, clearHide);  
  
    ...  
            // PhoneWindow  
            r.window = r.activity.getWindow();  
            // 获取Activity的DecorView  
            View decor = r.window.getDecorView();  
            decor.setVisibility(View.INVISIBLE);  
            // WindowManagerImpl  
            ViewManager wm = a.getWindowManager();  
            WindowManager.LayoutParams l = r.window.getAttributes();  
            a.mDecor = decor;  
            l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;  
            l.softInputMode |= forwardBit;  
            if (a.mVisibleFromClient) {  
                a.mWindowAdded = true;  
                wm.addView(decor, l);  
            }  
  
        ...  
}  
```

第6行的方法调用与Activity的onResume生命周期有关，但这次我们的重点不在这里，我们一起来看看第22行的wm（即WindowManagerImpl）的addView方法：
（android.view.WindowManagerImpl）
```java
	private final WindowManagerGlobal mGlobal = WindowManagerGlobal.getInstance();  
  
	public void addView(View view, ViewGroup.LayoutParams params) {  
       mGlobal.addView(view, params, mDisplay, mParentWindow);  
   }  
```

工作被委托给了WindowManagerGlobal（由创建方式可以猜测其使用了单件设计模式），我们一起进入WindowManagerGlobal看看：
（android.view.WindowManagerGlobal）
```java
public void addView(View view, ViewGroup.LayoutParams params,  
        Display display, Window parentWindow) {  
    ...  
  
    ViewRootImpl root;  
    View panelParentView = null;  
  
    synchronized (mLock) {  
        ...  
  
        root = new ViewRootImpl(view.getContext(), display);  
  
        ...  
    }  
  
    // do this last because it fires off messages to start doing things  
    try {  
        root.setView(view, wparams, panelParentView);  
    } catch (RuntimeException e) {  
        // BadTokenException or InvalidDisplayException, clean up.  
        synchronized (mLock) {  
            final int index = findViewLocked(view, false);  
            if (index >= 0) {  
                removeViewLocked(index, true);  
            }  
        }  
        throw e;  
    }  
}  
```

到这里，我们UI绘制的主角终于登场了——ViewRoot
这个方法里有两个关键点：
1. 第11行ViewRoot的创建
2. 第18行的setView方法

由篇幅问题，剩下的内容我们就放到下篇博客在分析了，最后再以我们的分析路线来结束本篇博客吧：
![总结分析路线图](4.png)