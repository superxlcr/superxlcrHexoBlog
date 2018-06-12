---
title: Android闹钟app提醒界面属性小结
tags: [android,应用]
categories: [android]
date: 2016-04-08 15:55:54
description: Android闹钟app提醒界面属性小结
---
前些日子回顾到自己以前实现的一款闹钟APP，发现其中的提醒界面（即一个时间到了的闹钟界面）有许多以前没有注意到的细节，因此在此写下一篇小结。

闹钟界面的xml代码如下：
```xml
<activity  
    android:name=".AlarmActivity"  
    android:configChanges="orientation|keyboardHidden|keyboard|navigation"  
    android:excludeFromRecents="true"  
    android:label="@string/app_name"  
    android:launchMode="singleInstance"  
    android:theme="@android:style/Theme.Wallpaper.NoTitleBar" >  
</activity>  
```

以下针对关键属性进行解释：

**configChanges**

一般而言，当一个Activity界面横竖屏切换了以后，整个Activity会重新加载来重启一次Activity以适应变化。但对于我们的闹钟界面而言，我们当然不希望当用户切横竖屏时，整个界面就被重新加载了。因此我们在configChanges属性上表示出这些改变的事件，在configChanges上标识的事件发生时不会导致Activity重新加载，而是调用onConfigurationChanged这个回调方法来处理。
值的注意的是，网上有资料说明：自从Android 3.2（API 13），在设置Activity的android:configChanges="orientation|keyboardHidden"后，还是一样 会重新调用各个生命周期的。因为screen size也开始跟着设备的横竖切换而改变。所以，在AndroidManifest.xml里设置的MiniSdkVersion和 TargetSdkVersion属性大于等于13的情况下，如果我们想阻止程序在运行时重新加载Activity，除了设置"orientation"， 我们还必须设置"ScreenSize"

**excludeFromRecents**

这个属性设置为true之后，该Activity的运行将不会使应用出现在最近运行的列表上。设置该属性的理由很简单，如果用户很久以前设置了一个闹钟，刚刚突然响了，我们总不能就认为用户刚刚才使用了我们的应用吧。

**launchMode**

这是属性是控制Activity的启动模式的属性，有四种属性可选。此处我们选择的启动模式为singleInstance，即把这个Activity分配到一个独立的任务栈中去。为什么要这么做呢？我也是一次偶然的机会发现了这么一个情景：假设用户设置了一个闹钟后，并没有关闭应用，而是回到桌面转到别的应用中去了。那么如果在闹钟应用进程没有被回收之前，闹钟响了，触发了闹钟界面，此时关闭闹钟界面后就有可能出现问题。如果闹钟界面的启动模式不是singleInstance，那么关闭闹钟界面后留下的极有可能是用户之前没有关闭的设置闹钟的界面，而不是用户在闹钟响起之前所浏览的界面了。

从上面的总结可以看出，一个小小的闹钟界面也是有很大的讲究的。对于应用的设计我们需要关注很多细节的东西，最后才能创造出一款精致的作品。