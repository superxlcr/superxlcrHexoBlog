---
title: Android Context的一些总结
tags: [android,基础知识]
categories: [android]
date: 2016-04-02 10:46:18
description: Context的概念、引用问题、Context应用场景
---
最近看了一些关于Android Context使用要注意的要点，在此写一篇博客总结。

# Context的概念

Context即上下文，或者叫做场景，它提供了关于应用环境的一些全局信息。我们在加载资源、启动一个新的Activity、获取系统服务、获取内部文件（夹）路径、创建View操作时等都需要Context的参与，可见Context有多重要。以下是Context的类结构图：
![Context类结构图](1.jpg)

可以看到我们常用的Activity、Service、Application都是Context的子类，那么这几个类当做Context基类使用的时候有什么不同呢？接下来我们一起来探讨下。

# 引用问题

一个典型的由Context引起的引用问题就是单例模式的使用，先来看看以下代码：
```java
public class Singleton {  
    private static Singleton singleton;  
      
    public static Singleton getISingleton(Context context) {  
        if (singleton == null) {  
            singleton = new Singleton(context);  
        }  
        return singleton;  
    }  
      
    private Context context;  
      
    private Singleton(Context context) {  
        this.context = context;  
    }  
}  
```

对于上述的单例，大家应该都不陌生（请别计较getInstance的效率问题），内部保持了一个Context的引用。上面代码的问题在于：如果传入的Context是一个Activity或Service，那么这个组件将会由于被强引用持有而无法被回收掉，从而导致内存泄露问题。因此，为了让我们的控件能被顺利回收掉，我们此处应该使用Application作为Context使用，把代码中的构造函数改为如下：
```java
private Singleton(Context context) {  
    this.context = context.getApplicationContext();  
}  
```

# Context应用场景

看完上面的例子，可能大家认为只要以后一直使用Application作为Context就好了，然而事实并不是这样的，每个Context的子类都有适合它的应用场景，下面我们来看一下：
![Context应用场景示意图](2.png)

对于上面的No的解释分别如下：
- NO1：启动Activity在这些类中是可以的，但是需要创建一个新的task（强制规定使用参数FLAG_ACTIVITY_NEW_TASK），故一般情况不推荐
- NO2：在这些类中去layout inflate是合法的，但是会使用系统默认的主题样式，如果你自定义了某些样式可能不会被使用
- NO3：在receiver为null时允许，在4.2或以上的版本中，用于获取黏性广播的当前值（可以无视）

PS：ContentProvider、BroadcastReceiver之所以在上述表格中，是因为在其内部方法中都有一个context用于使用。
其实对于上面的表格，我们重点只要把握住一点：凡是跟UI相关的，都应该使用Activity做为Context来处理。