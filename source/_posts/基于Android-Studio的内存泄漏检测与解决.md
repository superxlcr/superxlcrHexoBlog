---
title: 基于Android Studio的内存泄漏检测与解决
tags: [android,内存]
categories: [android]
date: 2017-11-23 20:25:17
description: （转载）什么是内存泄漏、内存泄漏的检测方法、
---



# 什么是内存泄漏

Android虚拟机的垃圾回收采用的是根搜索算法。GC会从根节点（GC Roots）开始对heap进行遍历。到最后，部分没有直接或者间接引用到GC Roots的就是需要回收的垃圾，会被GC回收掉。而内存泄漏出现的原因就是存在了无效的引用，导致本来需要被GC的对象没有被回收掉。

比如：


```java
public class LeakActivity extends AppCompatActivity {

    public static Leak mLeak;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_leak);
        Toolbar toolbar = (Toolbar) findViewById(R.id.toolbar);
        setSupportActionBar(toolbar);
        mLeak = new Leak();
    }

    class Leak {

    }
}
```



mLeak是存储在静态区的静态变量，而Leak是内部类，其持有外部类Activity的引用。这样就导致Activity需要被销毁时，由于被mLeak所持有，所以系统不会对其进行GC，这样就造成了内存泄漏。
另外一个例子


```java
public class Singleton {
    private static Singleton singleton;
    private Context mContext;

    private Singleton(Context context) {
        this.mContext = context;
    }

    public static Singleton getSingleton(Context context) {
        if (singleton == null) {
            synchronized (Singleton.class) {
                if (singleton == null) {
                    singleton = new Singleton(context);
                }
            }
        }
        return singleton;
    }
}
```



如果我们在在调用Singleton的getInstance()方法时传入了Activity。那么当instance没有释放时，这个Activity会一直存在。因此造成内存泄露。

解决方法可以将new Singleton(context)改为new Singleton(context.getApplicationContext())即可，这样便和传入的Activity没关系了。

# 内存泄漏的检测方法

打开Android Studio，编译代码运行App，然后点击Android Monitor，然后点击Monitor对应的Monitors Tab，界面如下：
![这里写图片描述_pic1](1.png)
在Memory一栏中，可以观察不同时间App内存的动态使用情况，点击Memory 右侧第二个按钮可以手动触发GC，点击第三个按钮可以进入HPROF Viewer界面，查看Java的Heap，如下图
![这里写图片描述_pic2](2.png)
Reference Tree代表指向该实例的引用，可以从这里面查看内存泄漏的原因，Shallow Size指的是该对象本身占用内存的大小，Retained Size代表该对象被释放后，垃圾回收器能回收的内存总和。
内存泄漏检测的方法 

点击手动进行GC，再点击观看JavaHeap，点击Analyzer Task，Android Monitor就可以为我们自动分析泄漏的Activity啦，分析出来如下图所示：
![这里写图片描述_pic3](3.png)
在Reference Tree里面，我们直接就可以看到持有该Activity的单例对象，直接定位到该单例中的代码，发现代码中出现了


```java
public class LeakActivity extends AppCompatActivity {

    public static Leak mLeak;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_leak);
        Toolbar toolbar = (Toolbar) findViewById(R.id.toolbar);
        setSupportActionBar(toolbar);
        mLeak = new Leak();
    }

    class Leak {

    }
}
```





