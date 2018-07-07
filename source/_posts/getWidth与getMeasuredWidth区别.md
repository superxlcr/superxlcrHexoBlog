---
title: getWidth与getMeasuredWidth区别
tags: [android,view]
categories: [android]
date: 2016-10-07 15:55:49
description: getWidth与getMeasuredWidth区别
---
最近遇到了关于View类中，getWidth方法与getMeasuredWidth方法的区别，在此写一遍博客总结一下。
众所周知View的绘制流程分为Measure、Layout和Draw三个阶段，想了解详细情况的可以参考我的这篇博客：

[一个Activity的显示过程总结（四）](/2016/05/17/一个Activity的显示过程总结（四）/)

getMeasuredWidth返回的是：Measure阶段测量的结果 
其代码如下：

```java
	/**
     * Like {@link #getMeasuredWidthAndState()}, but only returns the
     * raw width component (that is the result is masked by
     * {@link #MEASURED_SIZE_MASK}).
     *
     * @return The raw measured width of this view.
     */
    public final int getMeasuredWidth() {
        return mMeasuredWidth & MEASURED_SIZE_MASK;
    }
```


而getView返回的是：Layout阶段测量的结果，一般也是View实际绘制的大小
其代码如下：

```java
    /**
     * Return the width of the your view.
     *
     * @return The width of your view, in pixels.
     */
    @ViewDebug.ExportedProperty(category = "layout")
    public final int getWidth() {
        return mRight - mLeft;
    }
```


通常情况下，两个方法返回的值应该是相等的。
            