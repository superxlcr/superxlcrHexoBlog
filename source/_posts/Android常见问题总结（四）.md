---
title: Android常见问题总结（四）
tags: [android,基础知识]
categories: [android]
date: 2016-05-09 11:29:34
description: Android三种动画、Handler、Looper消息队列模型、怎样退出终止App、Asset目录与res目录的区别、
---
上一篇文章的传送门：[Android常见问题总结（三）](/2016/05/02/Android常见问题总结（三）/)

# Android三种动画

如今Android的动画主要有三种，分别是：逐帧（Frame）动画，补间（Tween）动画，属性（Property）动画

## 逐帧（Frame）动画

逐帧动画是最容易理解的动画，它要求我们把动画过程的每张静态图片都准备好，然后依次显示，利用人眼“视觉暂留”的原理形成动画效果。
例子：肥波跳舞？
素材准备（共27帧）：

fat_po.xml，animation-list的oneshot属性用于设置动画是否只播放一次，true是，false表示循环播放
```xml
<?xml version="1.0" encoding="utf-8"?>  
<animation-list xmlns:android="http://schemas.android.com/apk/res/android"  
    android:oneshot = "false" >  
  
    <item android:drawable="@drawable/fat_po_f01" android:duration="60" />  
    <item android:drawable="@drawable/fat_po_f02" android:duration="60" />  
    <item android:drawable="@drawable/fat_po_f03" android:duration="60" />  
    <item android:drawable="@drawable/fat_po_f04" android:duration="60" />  
    <item android:drawable="@drawable/fat_po_f05" android:duration="60" />  
    <item android:drawable="@drawable/fat_po_f06" android:duration="60" />  
    <item android:drawable="@drawable/fat_po_f07" android:duration="60" />  
    <item android:drawable="@drawable/fat_po_f08" android:duration="60" />  
    <item android:drawable="@drawable/fat_po_f09" android:duration="60" />  
    <item android:drawable="@drawable/fat_po_f10" android:duration="60" />  
    <item android:drawable="@drawable/fat_po_f11" android:duration="60" />  
    <item android:drawable="@drawable/fat_po_f12" android:duration="60" />  
    <item android:drawable="@drawable/fat_po_f13" android:duration="60" />  
    <item android:drawable="@drawable/fat_po_f14" android:duration="60" />  
    <item android:drawable="@drawable/fat_po_f15" android:duration="60" />  
    <item android:drawable="@drawable/fat_po_f16" android:duration="60" />  
    <item android:drawable="@drawable/fat_po_f17" android:duration="60" />  
    <item android:drawable="@drawable/fat_po_f18" android:duration="60" />  
    <item android:drawable="@drawable/fat_po_f19" android:duration="60" />  
    <item android:drawable="@drawable/fat_po_f20" android:duration="60" />  
    <item android:drawable="@drawable/fat_po_f21" android:duration="60" />  
    <item android:drawable="@drawable/fat_po_f22" android:duration="60" />  
    <item android:drawable="@drawable/fat_po_f23" android:duration="60" />  
    <item android:drawable="@drawable/fat_po_f24" android:duration="60" />  
    <item android:drawable="@drawable/fat_po_f25" android:duration="60" />  
    <item android:drawable="@drawable/fat_po_f26" android:duration="60" />  
    <item android:drawable="@drawable/fat_po_f27" android:duration="60" />  
      
</animation-list>  
```

Activity布局xml，把逐帧（Frame）动画设置为ImageView的背景：
```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"  
    android:layout_width="match_parent"  
    android:layout_height="match_parent"  
    android:gravity="center">  
  
    <ImageView   
        android:id="@+id/image_view"  
        android:layout_width="wrap_content"  
        android:layout_height="wrap_content"  
        android:background="@animator/fat_po"/>  
  
</LinearLayout>  
```

Activity的Java代码：
```java
public class MyActivity extends Activity {  
      
    @Override  
    protected void onCreate(Bundle savedInstanceState) {  
        super.onCreate(savedInstanceState);  
        setContentView(R.layout.activity_main);  
        ImageView imageView = (ImageView)findViewById(R.id.image_view);  
        // 开始动画，默认为停止  
        ((AnimationDrawable)imageView.getBackground()).start();  
    }  
      
}  
```

由于逐帧动画默认是停止，因此我们需要调用其start方法才能播放动画。

动画效果如下：
![肥波跳舞动画](1.gif)

## 补间（Tween）动画

相比与逐帧动画要求我们把动画的每一帧都列出来，补间动画只需要我们指定动画开始、动画结束等“关键帧”，而其中动画变化的“中间帧”由系统计算补齐。
例子：花瓣开合
花瓣动画xml文件：
花瓣关闭xml文件，使用了scale（大小变化），alpha（透明度变化），rotate（旋转变化）三种动画，执行的速度是linear线性的
```xml
<?xml version="1.0" encoding="utf-8"?>  
<set xmlns:android="http://schemas.android.com/apk/res/android"  
    android:interpolator="@android:anim/linear_interpolator">  
      
    <scale android:fromXScale="1.0"  
        android:toXScale="0.01"  
        android:fromYScale="1.0"  
        android:toYScale="0.01"  
        android:pivotX="50%"  
        android:pivotY="50%"  
        android:fillAfter="true"  
        android:duration="3000"/>  
      
    <alpha   
        android:fromAlpha="1"  
        android:toAlpha="0.05"  
        android:duration="3000"/>  
  
    <rotate   
        android:fromDegrees="0"  
        android:toDegrees="1800"  
        android:pivotX="50%"  
        android:pivotY="50%"  
        android:duration="3000"/>  
</set>  
```

花瓣打开xml文件
```xml
<?xml version="1.0" encoding="utf-8"?>  
<set xmlns:android="http://schemas.android.com/apk/res/android"  
    android:interpolator="@android:anim/linear_interpolator">  
      
    <scale android:fromXScale="0.01"  
        android:toXScale="1.0"  
        android:fromYScale="0.01"  
        android:toYScale="1.0"  
        android:pivotX="50%"  
        android:pivotY="50%"  
        android:fillAfter="true"  
        android:duration="3000"/>  
      
    <alpha   
        android:fromAlpha="0.05"  
        android:toAlpha="1"  
        android:duration="3000"/>  
  
    <rotate   
        android:fromDegrees="1800"  
        android:toDegrees="0"  
        android:pivotX="50%"  
        android:pivotY="50%"  
        android:duration="3000"/>  
</set>  
```

界面的xml文件，只有一个简单的imageView：
```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"  
    android:layout_width="match_parent"  
    android:layout_height="match_parent"  
    android:gravity="center">  
  
    <ImageView   
        android:id="@+id/image_view"  
        android:layout_width="wrap_content"  
        android:layout_height="wrap_content"  
        android:src="@drawable/flower"/>  
  
</LinearLayout>  
```

Activity的Java代码，加载了close与open动画后，通过Timer的重复任务不断发送消息给handler，通过handler来播放动画:
```java
public class MyActivity extends Activity {  
      
    private Handler handler = new Handler() {  
        public void handleMessage(android.os.Message msg) {  
            if (flag)  
                imageView.startAnimation(close);  
            else  
                imageView.startAnimation(open);  
            flag = !flag;  
        };  
    };  
      
    private Animation open, close;  
    private ImageView imageView;  
    boolean flag = true; // 花瓣状态，true开，false合  
      
    @Override  
    protected void onCreate(Bundle savedInstanceState) {  
        super.onCreate(savedInstanceState);  
        setContentView(R.layout.activity_main);  
        imageView = (ImageView)findViewById(R.id.image_view);  
        // 花瓣张开  
        open = AnimationUtils.loadAnimation(this, R.anim.open);  
        // 保留动画变化后的状态  
        open.setFillAfter(true);  
        // 花瓣关闭  
        close = AnimationUtils.loadAnimation(this, R.anim.close);  
        // 保留动画变化后的状态  
        close.setFillAfter(true);  
        // 设置重复任务  
        new Timer().schedule(new TimerTask() {  
              
            @Override  
            public void run() {  
                // 通过handler发送消息刷新UI  
                handler.sendEmptyMessage(0);  
            }  
        }, 0, 3500);  
    }  
      
}  
```

效果如下：
![花瓣开合动画](2.gif)

对于补间（Tween）动画我们只需要准备动画开始与结束的关键帧即可，其中的“中间帧”使用Android提供的多种变换（Scale，Alpha，Rotate，translate）生成即可

## 属性（Property）动画

从某种角度来看，属性动画是增强版的补间动画，属性动画的强大主要体现在两个方面：
1. 补间动画只能定义两个关键帧在“透明度”、“旋转”、“缩放”、“位移”4个方面的变化，但属性动画可以定义任何属性的变化
2. 补间动画只能对UI组件执行动画，但属性动画几乎可以对任何对象执行动画

由于属性动画比较复杂，本文在此不展开讨论了，推荐两篇大神博客供大家参考：
[Android 属性动画（Property Animation） 完全解析 （上）](https://blog.csdn.net/lmj623565791/article/details/38067475)
[Android 属性动画（Property Animation） 完全解析 （下）](https://blog.csdn.net/lmj623565791/article/details/38092093)

# Handler、Looper消息队列模型

简单提提Handler、Looper模型中各部分的作用，主要有以下三部分：
- MessageQueue：消息队列，存储待处理的消息
- Looper：封装了消息队列与Handler，线程绑定，使用loop方法循环处理消息
- Handler：消息处理的辅助类，里面封装了消息的投递、处理和获取等一系列操作

详细的情况请参考看我另一篇博文：
[Android Looper和Handler小结](/2016/03/08/Android-Looper和Handler小结)

# 怎样退出终止App

测试方案：
依次打开3个Activity（ActivityA，ActivityB，ActivityC），并在第3个Activity中终止App

## System.exit（0）

暴露一个Application的public方法:
```java
public class MyApplication extends Application {  
  
    public void exit() {  
        System.exit(0);  
    }  
      
}  
```

在第三个Activity中调用该方法：
```java
public class ActivityC extends Activity {  
  
    @Override  
    protected void onCreate(Bundle savedInstanceState) {  
        super.onCreate(savedInstanceState);  
        setContentView(R.layout.activity);  
        Button button = (Button)findViewById(R.id.btn);  
        button.setText("Finish app");  
        button.setOnClickListener(new OnClickListener() {  
              
            @Override  
            public void onClick(View v) {  
                // 退出App  
                ((MyApplication)getApplication()).exit();  
            }  
        });  
    }  
}  
```

**失败**，应用进程被杀死，然而过会应用重启了……并且ActivityA和ActivityB都被“复活了”，只杀死了ActivityC

## android.os.Process.killProcess(android.os.Process.myPid())

**失败**，与System.exit（0）一样，虽然杀死了进程，但过会就被重启了，并且ActivityA和ActivityB都被“复活了”

## ActivityManager#killBackgroundProcesses()

**失败**，毫无反应

## 自定义BaseActivity维护Activity列表

自定义一个BaseActivity：
```java
public class BaseActivity extends Activity {  
  
    // 维护一个Activity软引用的列表  
    private static List<SoftReference<Activity>> list = new ArrayList<SoftReference<Activity>>();  
  
    @Override  
    protected void onCreate(Bundle savedInstanceState) {  
        super.onCreate(savedInstanceState);  
        list.add(new SoftReference<Activity>(this));  
    }  
  
    @Override  
    protected void onDestroy() {  
        super.onDestroy();  
        list.remove(new SoftReference<Activity>(this));  
    }  
  
    /** 
     * 关闭所有的Activity 
     */  
    public void finishAll() {  
        for (SoftReference<Activity> sr : list) {  
            if (sr.get() != null) {  
                sr.get().finish();  
            }  
        }  
    }  
  
}  
```

对于ActivityA、ActivityB和ActivityC继承BaseActivity而不是Activity，在ActivityC中调用finishAll方法即可关闭所有Activity进而退出App

## finishAffinity()

直接关闭相同任务栈中的所用Activity，与上一个方法效果差不多，但是是Android自带的，方便多了

## 总结

综上所述，测试结果如下：
1. System.exit（0）：只能关闭当前Activity，关闭进程可能导致数据存储问题，不推荐
2. android.os.Process.killProcess(android.os.Process.myPid())：同上
3. ActivityManager#killBackgroundProcesses()：测试无效
4. 自定义BaseActivity维护Activity列表：可以关闭依次启动的所用Activity，进而退出整个App
5. finishAffinity()：可以关闭同一个任务栈中的所有Activity，Android自带方法，比较方便

# Asset目录与res目录的区别

Asset目录和res目录均为Android中用来存放资源的目录，其中：
- asset目录下存放的资源代表应用无法直接访问的原生资源，应用程序需要通过AssetManager以二进制流的形式来读取资源
- res目录下的资源可通过R资源清单类访问，Android SDK会在编译时在R类中为他们创建对应的索引项

# Android怎么加速启动Activity

个人认为，影响Activity启动时间的主要有两个地方：
1. onCreate、onStart、onResume等回调方法的执行时间
2. Activity对应的界面的inflate时间

对于第一点，我们应该尽量减少在这些回调方法中执行耗时操作（涉及数据库，图片等），如果一定要执行耗时操作，可以考虑新开子线程处理。
对于第二点，我们应该合理使用各种xml的优化标签，并界面上减少View的嵌套层数与绘制时间。（可参考Android常见问题总结（三））