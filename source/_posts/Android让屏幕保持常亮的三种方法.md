---
title: Android让屏幕保持常亮的三种方法
tags: [android]
categories: [android]
date: 2017-12-16 22:17:59
description: 方法一：持有WakeLock、方式二：在Window设置flag、方式三：在界面布局xml中顶层添加属性
---

# 方法一：持有WakeLock

首先获取WakeLock相关权限：

```java
<uses-permission android:name="android.permission.WAKE_LOCK" />
```


然后通过PowerManager获取WakeLock后，在onResume以及onPause执行相应操作：

```java
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        PowerManager powerManager = (PowerManager)getSystemService(POWER_SERVICE);
        if (powerManager != null) {
            mWakeLock = powerManager.newWakeLock(PowerManager.FULL_WAKE_LOCK, "WakeLock");
        }
    }

    @Override
    protected void onResume() {
        super.onResume();
        if (mWakeLock != null) {
            mWakeLock.acquire();
        }
    }

    @Override
    protected void onPause() {
        super.onPause();
        if (mWakeLock != null) {
            mWakeLock.release();
        }
    }

```


WakeLock获取时相关的flag如下所示：

1. PARTIAL_WAKE_LOCK :保持CPU 运转，屏幕和键盘灯有可能是关闭的。
2. SCREEN_DIM_WAKE_LOCK ：保持CPU 运转，允许保持屏幕显示但有可能是灰的，允许关闭键盘灯
3. SCREEN_BRIGHT_WAKE_LOCK ：保持CPU 运转，允许保持屏幕高亮显示，允许关闭键盘灯
4. FULL_WAKE_LOCK ：保持CPU 运转，保持屏幕高亮显示，键盘灯也保持亮度

PS：现在官方已经不推荐使用这种方式保持亮屏了，推荐改为以下两种方式



# 方式二：在Window设置flag




```java
getWindow().addFlags(WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON);
```


这种方式不需要申请权限，也是官方推荐的做法


# 方式三：在界面布局xml中顶层添加属性


可以再界面xml文件中的顶层布局添加属性即可：

```html
android:keepScreenOn="true"
```




