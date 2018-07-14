---
title: Android常见问题总结（六）
tags: [android,基础知识]
categories: [android]
date: 2017-08-08 19:54:28
description: 如何处理Android Crash 并重启手机、如何把view转化为bitmap、library 与 app资源同名冲突问题、Android 扬声器与听筒切换、关于addView后宽高的一些细节
---
上一篇博客传送门：[Android常见问题总结（五）](/2016/05/17/Android常见问题总结（五）/)
# 如何处理Android Crash 并重启手机
一般而言，发生了APP Crash是由于我们的程序抛出了我们未捕获的RuntimeException
在Java中，我们可以通过setDefaultUncaughtExceptionHandler方法为某个线程设置默认未捕获异常处理器，当程序抛出未捕获异常时，会交由改处理器处理
Android的默认UncaughtExceptionHandler为弹出Dialog提醒用户APP Crash，并杀死进程退出程序
我们可以通过编写自己的UncaughtExceptionHandler来处理APP Crash的问题：

```java
        Thread.setDefaultUncaughtExceptionHandler(new Thread.UncaughtExceptionHandler() {
            @Override
            public void uncaughtException(Thread t, Throwable e) {
                Intent intent = new Intent(MainActivity.this, MainActivity.class);
                startActivity(intent);
                Process.killProcess(Process.myPid());
            }
        });
```


下面例子中，点击按钮会抛出一个运行时异常，上图为未设置Handler之前，下图为设置Handler之后：


![_pic1](1.gif)![_pic2](2.gif)




# 如何把view转化为bitmap
可以使用getDrawingCache方法：

```java
// 设置能否缓存图片信息（drawing cache）
view.setDrawingCacheEnabled(true);
// 如果能够缓存图片，则创建图片缓存
view.buildDrawingCache();
// 如果图片已经缓存，返回一个bitmap
Bitmap bitmap = view.getDrawingCache();
// 释放缓存占用的资源
view.destroyDrawingCache();
```

# library 与 app资源同名冲突问题

假设有以下场景：
![_pic3](3.png)



在一个Project中有两个module，一个是应用app，另一个是android的library：mylibrary
两个module中存在着相同的资源名称


这种情况会导致什么问题呢？
会导致mylibrary module中的activity_main.xml被覆盖，无法获取


在网上查到的R类索引生成规则如下所示：


资源ID是一个4字节的无符号整数，其中，最高字节表示Package ID，次高字节表示Type ID，最低两字节表示Entry ID。        
Package ID相当于是一个命名空间，限定资源的来源。Android系统当前定义了两个资源命令空间，其中一个系统资源命令空间，它的Package ID等于0x01，另外一个是应用程序资源命令空间，它的Package ID等于0x7f。所有位于[0x01, 0x7f]之间的Package ID都是合法的，而在这个范围之外的都是非法的Package ID。前面提到的系统资源包package-export.apk的Package ID就等于0x01，而我们在应用程序中定义的资源的Package ID的值都等于0x7f，这一点可以通过生成的R.java文件来验证。
        
Type ID是指资源的类型ID。资源的类型有animator、anim、color、drawable、layout、menu、raw、string和xml等等若干种，每一种都会被赋予一个ID。        
Entry ID是指每一个资源在其所属的资源类型中所出现的次序。注意，不同类型的资源的Entry ID有可能是相同的，但是由于它们的类型不同，我们仍然可以通过其资源ID来区别开来。


由于名称相同，两个module会对该资源生成相同的索引即（app.R.layout.activity_main == mylibrary.R.layout.activity_main）
如果app module依赖mylibrary module，那么最终生成的apk中所有的资源文件res和R类都会被集中到一起，也就是说mylibrary中的activity_main会被完全覆盖
原本在mylibrary module中的获取的activity_main资源均会变为app中的activity_main资源


stackoverflow对此问题给出的解决方案是：为你的android library资源添加相关的前缀，如AppCompat的前缀为abc_
https://stackoverflow.com/questions/42513509/is-there-any-way-in-android-module-to-pick-up-resources-from-its-own-module-and



# Android 扬声器与听筒切换
要实现扬声器与听筒的切换.而android中实现对音量和振铃模式的控制主要通过AudioManager类来实现.



获取到AudioManager后，我们可以通过其几个关键的方法改变扬声器与听筒的状态：

1. isSpeakerphoneOn：用于检测当前是否开启了扬声器
2. setSpeakerphoneOn：设置是否打开扬声器
3. setMode：设置音频模式，设置扬声器时，使用MODE_NORMAL，设置听筒时，使用MODE_IN_COMMUNICATION

具体实现如下：


```java
    /**
     * 设置是否开启扬声器
     * @param isSpeakerOn 是否开启
     */
    public void setSpeakerphone(boolean isSpeakerOn) {
        AudioManager audioManager = (AudioManager) getSystemService(AUDIO_SERVICE);
        if (audioManager != null) {
            audioManager.setMode(isSpeakerOn ? AudioManager.MODE_NORMAL : AudioManager.MODE_IN_COMMUNICATION);
            audioManager.setSpeakerphoneOn(isSpeakerOn);
        }
    }
```


值得注意的是直接调用这个方法并不能切换生效，我们还需要申请MODIFY_AUDIO_SETTINGS的权限才行：

```java
<uses-permission android:name="android.permission.MODIFY_AUDIO_SETTINGS" />
```



# 关于addView后宽高的一些细节
对于ViewGroup#addView方法，我们应该都不陌生，该方法经常可以用来动态添加子View
但如果我们使用时不注意的话，添加到ViewGroup中的View的宽高往往并不会如我们所愿
想要避免这种情况，我们就需要确保调用addView方法时，我们的View具有正确的LayoutParams（宽高信息包含在里面），以下有三种方式确认我们的View设置了正确的LayoutParams：

1. 在使用LayoutInflater时，传入父ViewGroup：LayoutInflater只有当我们传入父ViewGroup时，才会为我们解析的View设置正确的LayoutParams，否则其实我们写在xml上的布局信息并不会准确的生效
2. 在addView之前，调用setLayoutParams设置
3. 调用addView带有LayoutParams参数的方法

一般而言，最常用的ViewGroup#addView方法如下所示：

```java
    public void addView(View child) {
        addView(child, -1);
    }

    public void addView(View child, int index) {
        if (child == null) {
            throw new IllegalArgumentException("Cannot add a null child view to a ViewGroup");
        }
        LayoutParams params = child.getLayoutParams();
        if (params == null) {
            params = generateDefaultLayoutParams();
            if (params == null) {
                throw new IllegalArgumentException("generateDefaultLayoutParams() cannot return null");
            }
        }
        addView(child, index, params);
    }
	
	protected LayoutParams generateDefaultLayoutParams() {
        return new LayoutParams(LayoutParams.WRAP_CONTENT, LayoutParams.WRAP_CONTENT);
    }
```


从第11行我们可以看出，如果我们动态添加的View并没有设置好相应的LayoutParams，系统变会填充上一个默认的LayoutParams，而大多数情况下，这个LayoutParams的布局方式并不是我们想要的

这也是为什么有时候addView之后，我们添加的View宽高为0的原因之一
