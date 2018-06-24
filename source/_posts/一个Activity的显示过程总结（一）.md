---
title: 一个Activity的显示过程总结（一）
tags: [android,源码]
categories: [android]
date: 2016-05-12 16:30:12
description: Activity的创建、performLaunchActivity、newActivity、attach
---
最近读了深入理解Android卷I，了解了一些有关于Activity显示的知识，在此写下一篇博客总结。
有兴趣自己看Android源码的同学可以前往：
http://grepcode.com/project/repository.grepcode.com/java/ext/com.google.android/android/
本博客分析的Android版本为4.4

# Activity的创建

要谈到Activity的界面显示，我们就得从Activity的创建说起
众所周知，Zygote是我们Android系统与应用启动相关的一个非常重要的进程：当我们要启动一个新的APP进程的时候，Zygote进程就会通过fork创建一个子进程，而子进程的入口函数就是ActivityThread类的main函数：
```java
public static void main(String[] args) {  
        ...  
  
        Looper.prepareMainLooper();  
  
        ActivityThread thread = new ActivityThread();  
        thread.attach(false);  
  
        if (sMainThreadHandler == null) {  
            sMainThreadHandler = thread.getHandler();  
        }  
  
        AsyncTask.init();  
  
        if (false) {  
            Looper.myLooper().setMessageLogging(new  
                    LogPrinter(Log.DEBUG, "ActivityThread"));  
        }  
  
        Looper.loop();  
  
        throw new RuntimeException("Main thread loop unexpectedly exited");  
    }  
```

省略一些无关的代码，我们可以看到在main函数中ActivityThread进行了一些与Looper、Handler相关的初始化操作，初始化完成后就能使用handler来处理message了，我们来看一下能处理什么消息（ActivityThread类的handleMessage函数）：
```java
public void handleMessage(Message msg) {  
            if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));  
            switch (msg.what) {  
                case LAUNCH_ACTIVITY: {  
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityStart");  
                    ActivityClientRecord r = (ActivityClientRecord)msg.obj;  
  
                    r.packageInfo = getPackageInfoNoCheck(  
                            r.activityInfo.applicationInfo, r.compatInfo);  
                    handleLaunchActivity(r, null);  
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);  
                } break;  
                ...  
            }  
            if (DEBUG_MESSAGES) Slog.v(TAG, "<<< done: " + codeToString(msg.what));  
        }  
```

能处理的message实在太多，此处我们只观察第一条。从message的信息来看，这条消息毫无疑问是用来创建一个Activity，我们再来看看第10行关键的handleLaunchActivity方法：
```java
private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent) {  
        ...  
        Activity a = performLaunchActivity(r, customIntent);  
  
        if (a != null) {  
            r.createdConfig = new Configuration(mConfiguration);  
            Bundle oldState = r.state;  
            handleResumeActivity(r.token, false, r.isForward,  
                    !r.activity.mFinished && !r.startsNotResumed);  
  
            ...  
    }  

```

这个函数中有两个关键的点：
1. performLaunchActivity：创建一个Activity
2. handleResumeActivity：为Activity界面显示进行准备

# performLaunchActivity

我们先来分析第一个关键点（ActivityThread#performLaunchActivity）：
```java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {  
        ...  
  
        Activity activity = null;  
        try {  
            java.lang.ClassLoader cl = r.packageInfo.getClassLoader();  
            activity = mInstrumentation.newActivity(  
                    cl, component.getClassName(), r.intent);  
            ...  
                activity.attach(appContext, this, getInstrumentation(), r.token,  
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,  
                        r.embeddedID, r.lastNonConfigurationInstances, config);  
  
                ...  
                mInstrumentation.callActivityOnCreate(activity, r.state);  
                ...  
  
        return activity;  
    }  
```

此处有三个关键的点：
1. newActivity：创建Activity
2. attach：对Activity执行一些初始化工作
3. callActivityOnCreate：执行Activity的onCreate方法初始化

# newActivity

第7行的newActivity方法创建了Activity，我们一起看一下这个方法究竟做了什么（android.app.Instrumentation#newActivity）：
```java
public Activity newActivity(ClassLoader cl, String className,  
            Intent intent)  
            throws InstantiationException, IllegalAccessException,  
            ClassNotFoundException {  
        return (Activity)cl.loadClass(className).newInstance();  
    }  
```

非常简单，newActivity方法通过Java的反射机制创建了Activity

# attach

我们再回到performLaunchActivity方法继续往下看，第10行有一个非常关键的attach函数，我们来看一下（android.app.Activity#attach）：
```java
final void attach(Context context, ActivityThread aThread,  
        Instrumentation instr, IBinder token, int ident,  
        Application application, Intent intent, ActivityInfo info,  
        CharSequence title, Activity parent, String id,  
        NonConfigurationInstances lastNonConfigurationInstances,  
        Configuration config) {  
    ...  
      
    mWindow = PolicyManager.makeNewWindow(this);  
    ...  
  
    mWindow.setWindowManager(  
            (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),  
            mToken, mComponent.flattenToString(),  
            (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);  
    ...  
    mWindowManager = mWindow.getWindowManager();  
    mCurrentConfig = config;  
}  
```

这里出现了两个非常关键的对象：Window和WindowManager

我们先来看一看Window的官方解释：
```
Abstract base class for a top-level window look and behavior policy. An instance of this class should be used as the top-level view added to the window manager. It provides standard UI policies such as a background, title area, default key processing, etc.
```
大致的意思是：WIndow是一个抽象基类，用于控制顶层窗口的外观与行为（即绘制背景和标题栏、默认的按键处理等）。它应该作为一个顶层的View加入到WindowManager中。
因此，View、Window和WindowManager的关系可以用下图来表示：
![View、Window和WindowManager关系图](1.png)

WindowManager拥有Window，而Window是UI的顶层View
那么由于Window是一个抽象类，在方法中它的实际类型是什么呢？我们一起回到代码中的第9行，此处调用了makeNewWindow方法来创建一个Window：
（com.android.internal.policy.PolicyManager）
```java
public final class PolicyManager {  
    private static final String POLICY_IMPL_CLASS_NAME =  
        "com.android.internal.policy.impl.Policy";  
  
    private static final IPolicy sPolicy;  
  
    static {  
        // Pull in the actual implementation of the policy at run-time  
        try {  
            Class policyClass = Class.forName(POLICY_IMPL_CLASS_NAME);  
            sPolicy = (IPolicy)policyClass.newInstance();  
        ...  
    }  
  
    ...  
  
    // The static methods to spawn new policy-specific objects  
    public static Window makeNewWindow(Context context) {  
        return sPolicy.makeNewWindow(context);  
    }  
  
    ...  
}  
```

从第18行的方法可以看到，PolicyManager调用了sPolicy变量的makeNewWindow方法来返回一个Window对象。那么sPolicy是什么呢？从第10行和第2行我们可以看出这是一个由Java反射加载的类，我们继续去看sPolicy的makeNewWindow方法：
（com.android.internal.policy.impl.Policy）
```java
public Window makeNewWindow(Context context) {  
    return new PhoneWindow(context);  
}  
```

OK，看来我们的Window的实际对象是一个PhoneWindow。
我们回到attach方法中，继续看一下WindowManager是个什么鬼：
（android.app.Activity）
```java
final void attach(Context context, ActivityThread aThread,  
        Instrumentation instr, IBinder token, int ident,  
        Application application, Intent intent, ActivityInfo info,  
        CharSequence title, Activity parent, String id,  
        NonConfigurationInstances lastNonConfigurationInstances,  
        Configuration config) {  
    ...  
      
    mWindow = PolicyManager.makeNewWindow(this);  
    ...  
  
    mWindow.setWindowManager(  
            (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),  
            mToken, mComponent.flattenToString(),  
            (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);  
    ...  
    mWindowManager = mWindow.getWindowManager();  
    mCurrentConfig = config;  
}  
```

看到第17行的WindowManager与第12行的setWindowManager相关，由于mWindow的实际类型是PhoneWindow，我们来看一下他的setWindowManager方法：
（com.android.internal.policy.impl.PhoneWindow）
```java
public void setWindowManager(WindowManager wm, IBinder appToken, String appName,  
        boolean hardwareAccelerated) {  
    mAppToken = appToken;  
    mAppName = appName;  
    mHardwareAccelerated = hardwareAccelerated  
            || SystemProperties.getBoolean(PROPERTY_HARDWARE_UI, false);  
    if (wm == null) {  
        wm = (WindowManager)mContext.getSystemService(Context.WINDOW_SERVICE);  
    }  
    mWindowManager = ((WindowManagerImpl)wm).createLocalWindowManager(this);  
}  
```

在第10行我们可以看到调用了createLocalWindowManager来生成一个WindowManager，该方法如下：
（android.view.WindowManagerImpl）
```java
public WindowManagerImpl createLocalWindowManager(Window parentWindow) {  
    return new WindowManagerImpl(mDisplay, parentWindow);  
}  
```

因此我们最终的WindowManager的实际类型是一个WindowManagerImpl。
此时，View、Window和WindowManager的关系图更新为如下所示：
![新View、Window和WindowManager关系图](2.png)

OK，剩下的内容我们将在下一篇博客中继续分析，在此之前我们先来回顾一下本篇博客的分析路线：
![总结分析路线](3.png)

上图中绿色的区域为我们分析完成的部分，现在我们经过以上步骤，获得了：Activity、Window和与Window绑定的WindowManger，下次我们将继续从callActivityOnCreate方法开始继续分析Activity的显示过程