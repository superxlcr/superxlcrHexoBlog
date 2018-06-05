---
title: scrollTo方法图解，以及Scroller的使用
tags: [android,view]
categories: [android]
date: 2018-04-01 10:16:41
description: scrollTo方法图解，以及Scroller的使用，通过computeScroll实现滑动效果
---

# scrollTo方法图解

Android系统手机屏幕的左上角为坐标系，同时y轴方向与笛卡尔坐标系的y轴方向想反。通过提供的api如getLeft , getTop, getBottom, getRight可以获得控件在parent中的相对位置。
当我们编写一些自定义的滑动控件时，会用到一些api如scrollTo(),scrollBy(),getScrollX(), getScrollY()。由于常常会对函数getScrollX(), getScrollY()返回的值的含义产生混淆，尤其是正负关系，因此本文将使用几幅图来对这些函数进行讲解以方便大家记忆。
值得注意的是，调用View的scrollTo()和scrollBy()是用于<font color=#ff0000>滑动View中的内容，而不是把某个View的位置进行改变。</font>
scrollTo(int x, int y) 是<font color=#ff0000>将View中内容滑动到相应的位置</font>，参考的坐标系原点为parent View的左上角。
调用scrollTo(100, 0)表示将View中的内容移动到x = 100， y = 0的位置，如下图所示。注意，图中黄色矩形区域表示的是一个parent View，绿色虚线矩形为parent view中的内容。一般情况下两者的大小一致，本文为了显示方便，将虚线框画小了一点。图中的黄色区域的位置始终不变，发生位置变化的是显示的内容。
![scrollTo(100,0)示意图](1.png)
同理，scrollTo(0, 100)的效果如下图所示：
![scrollTo(0,100)示意图](2.png)
scrollTo(100, 100)的效果图如下：
![scrollTo(100,100)示意图](3.png)
若函数中参数为负值，则子View的移动方向将相反:
![scrollTo(-100,0)示意图](4.png)

# Scroller的使用

使用scrollTo方法时，我们看到view的内容位置是立即进行刷新的，会给人一种不是特别友好的感觉
当我们需要实现一种滑动动画的效果时，就需要使用Scroller工具类
Scroller是一个用于计算滑动动画的工具类，我们的调用方式如下：
```java
// 调用方法开始计算滑动
scroller.startScroll(startX, startY, endX, endY, duration)
// 判断是否滑动结束，如果返回true，表示滑动动画仍未结束
scroller.computeScrollOffset
// 通过各种getter获取各种坐标
scroller.getStartX
scroller.getStartY
scroller.getCurrX
scroller.getCurrY
scroller.getFinalX
scroller.getFinalY
```
我们可以通过重写View#computeScroll回调方法来计算每次滑动的距离，最终实现滑动动画的效果：
```java
    @Override
    public void computeScroll() {
        if (scroller.computeScrollOffset()) {
            scrollTo(scroller.getCurrX(), scroller.getCurrY());
        }
    }
```