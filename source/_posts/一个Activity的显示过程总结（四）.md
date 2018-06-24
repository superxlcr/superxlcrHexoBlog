---
title: 一个Activity的显示过程总结（四）
tags: [android,源码]
categories: [android]
date: 2016-05-17 12:29:01
description: measure流程、layout流程、draw流程
---
有兴趣自己看Android源码的同学可以前往：
http://grepcode.com/project/repository.grepcode.com/java/ext/com.google.android/android/
本博客分析的Android版本为4.4

上一篇博客传送门：[一个Activity的显示过程总结（三）](/2016/05/15/一个Activity的显示过程总结（三）/)

上一篇博客我们讲到了ViewRoot中与UI相关的三个重要步骤：performMeasure（测量）、performLayout（布局）和performDraw（绘制），这次我们就来重点研究一下这三个方法。先上图说明三个方法的关系：
![UI相关三个步骤](1.png)

# measure流程

在performTraversals中有多次measure的流程，我们只分析其中一次即可：
（android.view.ViewRootImpl）
```java
final WindowManager.LayoutParams mWindowAttributes = new WindowManager.LayoutParams();  
private void performTraversals() {  
    ...  
    WindowManager.LayoutParams lp = mWindowAttributes;  
...  
                int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);  
                int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);  
  
                ...  
  
                 // Ask host how big it wants to be  
                performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);  
  
               ...  
}  
```

首先我们构造了一个WindowManager.LayoutParams对象：mWindowAttributes，其中包含了有关于Window（最外层布局）的信息。在performTraversals中，我们把它赋值给了lp，并通过getRootMeasureSpec方法返回了两个关键的测量量。我们一起来看看getRootMeasureSpec方法：
（android.view.ViewRootImpl）
```java
/** 
 * Figures out the measure spec for the root view in a window based on it's 
 * layout params. 
 * 
 * @param windowSize 
 *            The available width or height of the window 
 * 
 * @param rootDimension 
 *            The layout params for one dimension (width or height) of the 
 *            window. 
 * 
 * @return The measure spec to use to measure the root view. 
 */  
private static int getRootMeasureSpec(int windowSize, int rootDimension) {  
    int measureSpec;  
    switch (rootDimension) {  
  
    case ViewGroup.LayoutParams.MATCH_PARENT:  
        // Window can't resize. Force root view to be windowSize.  
        measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);  
        break;  
    case ViewGroup.LayoutParams.WRAP_CONTENT:  
        // Window can resize. Set max size for root view.  
        measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.AT_MOST);  
        break;  
    default:  
        // Window wants to be an exact size. Force root view to be that size.  
        measureSpec = MeasureSpec.makeMeasureSpec(rootDimension, MeasureSpec.EXACTLY);  
        break;  
    }  
    return measureSpec;  
}  
```

由该方法的注释我们可以得知这个方法是用来确定rootView（即DecorView）的measure spec（一个测量量）的。该方法传入两个参数，第一个是window的实际大小，第二个是window的尺寸（一般而言是MATCH_PARENT，即占满整个屏幕）。首先我们一起来看看MeasureSpec是什么东西：
（android.view.View）
```java
/** 
 * A MeasureSpec encapsulates the layout requirements passed from parent to child. 
 * Each MeasureSpec represents a requirement for either the width or the height. 
 * A MeasureSpec is comprised of a size and a mode. There are three possible 
 * modes: 
 * UNSPECIFIED 
 * The parent has not imposed any constraint on the child. It can be whatever size 
 * it wants. 
 * EXACTLY 
 * The parent has determined an exact size for the child. The child is going to be 
 * given those bounds regardless of how big it wants to be. 
 * AT_MOST 
 * The child can be as large as it wants up to the specified size. 
 * 
 * MeasureSpecs are implemented as ints to reduce object allocation. This class 
 * is provided to pack and unpack thesize, mode tuple into the int. 
 */  
public static class MeasureSpec {  
    private static final int MODE_SHIFT = 30;  
    private static final int MODE_MASK  = 0x3 << MODE_SHIFT;  
  
    /** 
     * Measure specification mode: The parent has not imposed any constraint 
     * on the child. It can be whatever size it wants. 
     */  
    public static final int UNSPECIFIED = 0 << MODE_SHIFT;  
  
    /** 
     * Measure specification mode: The parent has determined an exact size 
     * for the child. The child is going to be given those bounds regardless 
     * of how big it wants to be. 
     */  
    public static final int EXACTLY     = 1 << MODE_SHIFT;  
  
    /** 
     * Measure specification mode: The child can be as large as it wants up 
     * to the specified size. 
     */  
    public static final int AT_MOST     = 2 << MODE_SHIFT;  
  
    public static int makeMeasureSpec(int size, int mode) {  
        if (sUseBrokenMakeMeasureSpec) {  
            return size + mode;  
        } else {  
            return (size & ~MODE_MASK) | (mode & MODE_MASK);  
        }  
    }  
  
    ...  
}  
```

MeasureSpec是View内部的一个静态类。根据其注释我们可以得知：MeasureSpec用于描述通过父类到子类的布局要求（子类布局与父类相关），每个MeasureSpec表示宽度或高度的要求，一个MeasureSpec由一个大小（size）和模式（mode）组成（其实就是一个int，32位，根据计算方法可知前2位表示mode，后30位表示size）。定义的模式有三种：
- UNSPECIFIED：父类对子类没有任何约束
- EXACTLY：父View已经测量出子Viwe所需要的精确大小，这时候View的最终大小就是SpecSize所指定的值。对应于match_parent和精确数值这两种模式
- AT_MOST：子View的最终大小是父View指定的SpecSize值，并且子View的大小不能大于这个值，即对应wrap_content这种模式

makeMeasure方法的作用就是通过计算组合出一个合理的MeasureSpec。

回到getRootMeasureSpec，由于我们的Window默认是MATCH_PARENT，充满屏幕大小的，因此getRootMeasureSpec返回的MeasureSpec为：size是屏幕的宽、高，mode是EXACTLY。在获取了Window的MeasureSpec后，我们在performTraversals方法中调用了performMeasure方法，并把Window的MeasureSpec作为参数传入：
（android.view.ViewRootImpl）
```java
private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {  
    Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");  
    try {  
        mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);  
    } finally {  
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);  
    }  
}  
```

该方法调用了mView（即DecorView）的measure，由于View中的measure是个final方法，因此DecorView调用的方法即是View的measure方法：
（android.view.View）
```java
int mPrivateFlags;  
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {  
    ...  
    // 大致是强制需要测量的意思  
    if ((mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT ||  
            widthMeasureSpec != mOldWidthMeasureSpec ||  
            heightMeasureSpec != mOldHeightMeasureSpec) {  
  
        // first clears the measured dimension flag  
        mPrivateFlags &= ~PFLAG_MEASURED_DIMENSION_SET;  
  
        resolveRtlPropertiesIfNeeded();  
  
        // measure ourselves, this should set the measured dimension flag back  
        onMeasure(widthMeasureSpec, heightMeasureSpec);  
  
        // flag not set, setMeasuredDimension() was not invoked, we raise  
        // an exception to warn the developer  
        if ((mPrivateFlags & PFLAG_MEASURED_DIMENSION_SET) != PFLAG_MEASURED_DIMENSION_SET) {  
            throw new IllegalStateException("onMeasure() did not set the"  
                    + " measured dimension by calling"  
                    + " setMeasuredDimension()");  
        }  
  
        mPrivateFlags |= PFLAG_LAYOUT_REQUIRED;  
    }  
  
    mOldWidthMeasureSpec = widthMeasureSpec;  
    mOldHeightMeasureSpec = heightMeasureSpec;  
}  
```

首先在View中出现了一个很重要的变量：mPrivateFlags，这是一个int类型的变量，它的每一个bit用于表示一种状态。这个变量在研究layout与draw方法时我们也能见到。
在measure中，如果当前的状态为需要强制测量，而传入的MeasureSpec又不等于旧值时，就会调用onMeasure方法。onMeasure方法是我们自定义View时候可以重写的方法（不能重写final方法measure），在重写onMeasure方法时有一点需要注意：我们需要在onMeasure方法中调用setMeasuredDimension方法设置宽与高，否则在第20行就会抛出IllegalStateException异常。
View提供的默认onMeasure实现就调用了setMeasuredDimension方法：
（android.view.View）
```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {  
     setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),  
             getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));  
 } 
```

总而言之，经过了measure流程，View的宽与高的大小就确定了

# layout流程

measure流程确定了View的大小，接下来的layout流程就要确定View的位置了，在performTraversals中我们调用了performLayout方法：
（android.view.ViewRootImpl）
```java
private void performTraversals() {  
        ...  
        performLayout(lp, desiredWindowWidth, desiredWindowHeight);  
        ...  
}  
```

（android.view.ViewRootImpl）
```java
private void performLayout(WindowManager.LayoutParams lp, int desiredWindowWidth,  
        int desiredWindowHeight) {  
    ...  
  
    final View host = mView;  
    ...  
    try {  
        host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());  
  
        ...  
}  
```

在performLayout中，我们调用了host（即DecorView）的layout方法，由于ViewGroup类重写了layout，我们来看看ViewGroup的layout方法：
（android.view.ViewGroup）
```java
@Override  
public final void layout(int l, int t, int r, int b) {  
    if (!mSuppressLayout && (mTransition == null || !mTransition.isChangingLayout())) {  
        if (mTransition != null) {  
            mTransition.layoutChange(this);  
        }  
        super.layout(l, t, r, b);  
    } else {  
        // record the fact that we noop'd it; request layout when transition finishes  
        mLayoutCalledWhileSuppressed = true;  
    }  
}  
```

貌似挖掘不了什么有用的信息，我们继续看看super调用的View的layout方法：
（android.view.View）
```java
public void layout(int l, int t, int r, int b) {  
    int oldL = mLeft;  
    int oldT = mTop;  
    int oldB = mBottom;  
    int oldR = mRight;  
    boolean changed = isLayoutModeOptical(mParent) ?  
            setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);  
    if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {  
        onLayout(changed, l, t, r, b);  
        mPrivateFlags &= ~PFLAG_LAYOUT_REQUIRED;  
  
        ...  
    }  
    mPrivateFlags &= ~PFLAG_FORCE_LAYOUT;  
}  
```

该方法首先调用setFrame方法查看View的大小布局与上次相比是否发生变化，如果发生变化或mPrivateFlags的状态为需要进行layout，则调用onLayout进行布局。我们先来看看setFrame方法（setOpticalFrame内部也是通过setFrame方法完成）：
（android.view.View）
```java
protected boolean setFrame(int left, int top, int right, int bottom) {  
    boolean changed = false;  
  
    ...  
  
    if (mLeft != left || mRight != right || mTop != top || mBottom != bottom) {  
        changed = true;  
  
        // Remember our drawn bit  
        int drawn = mPrivateFlags & PFLAG_DRAWN;  
  
        int oldWidth = mRight - mLeft;  
        int oldHeight = mBottom - mTop;  
        int newWidth = right - left;  
        int newHeight = bottom - top;  
        boolean sizeChanged = (newWidth != oldWidth) || (newHeight != oldHeight);  
  
        ...  
  
  
        if (sizeChanged) {  
            if ((mPrivateFlags & PFLAG_PIVOT_EXPLICITLY_SET) == 0) {  
                // A change in dimension means an auto-centered pivot point changes, too  
                if (mTransformationInfo != null) {  
                    mTransformationInfo.mMatrixDirty = true;  
                }  
            }  
            sizeChange(newWidth, newHeight, oldWidth, oldHeight);  
        }  
  
        ...  
    }  
    return changed;  
}  
```

setFrame通过比较left、right、top、bottom四个变量确定View的布局是否发生变化，并返回该布尔值。另外，如果setFrame通过计算发现View的大小也发生了变化，则会调用sizeChange方法：
（android.view.View）
```java
private void sizeChange(int newWidth, int newHeight, int oldWidth, int oldHeight) {  
    onSizeChanged(newWidth, newHeight, oldWidth, oldHeight);  
    if (mOverlay != null) {  
        mOverlay.getOverlayView().setRight(newWidth);  
        mOverlay.getOverlayView().setBottom(newHeight);  
    }  
}  
```

sizeChange会调用onSizeChanged，我们可以重写该方法执行一些View大小变化时的操作。

回到layout方法，setFrame的返回值会被存在changed变量中，当changed为true时，即当View的布局发生了变化时，layout方法会调用onLayout方法。onLayout一般由View的子类进行重写以执行一些布局操作，ViewGroup把onLayout重写为抽象方法，使得每一个ViewGroup布局都需要重写onLayout实现自己的特定布局效果：
（android.view.ViewGroup）
```java
@Override  
protected abstract void onLayout(boolean changed,  
        int l, int t, int r, int b);  
```

# draw流程

layout流程完成后，我们获得了View的大小和布局，剩下的工作就是把View绘制到我们的屏幕上了。在performTraversals中，我们调用了performLayout获取布局后，调用了performDraw来绘制我们需要的图像：
（android.view.ViewRootImpl）
```java
private void performDraw() {  
    ...  
  
    final boolean fullRedrawNeeded = mFullRedrawNeeded;  
    mFullRedrawNeeded = false;  
  
    mIsDrawing = true;  
    Trace.traceBegin(Trace.TRACE_TAG_VIEW, "draw");  
    try {  
        draw(fullRedrawNeeded);  
    } finally {  
        mIsDrawing = false;  
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);  
    }  
  
    ...  
}  
```

在performDraw方法中，我们调用了draw方法进行绘制，并传入了一个布尔值表示是否整个屏幕都需要重新绘制：
（android.view.ViewRootImpl）
```java
private void draw(boolean fullRedrawNeeded) {  
    Surface surface = mSurface;  
    ...  
  
    final Rect dirty = mDirty;  
    ...  
  
    if (fullRedrawNeeded) {  
        attachInfo.mIgnoreDirtyState = true;  
        dirty.set(0, 0, (int) (mWidth * appScale + 0.5f), (int) (mHeight * appScale + 0.5f));  
    }  
  
    ...  
  
    // 使用硬件渲染绘制  
    if (!dirty.isEmpty() || mIsAnimating) {  
        if (attachInfo.mHardwareRenderer != null && attachInfo.mHardwareRenderer.isEnabled()) {  
            ...  
  
            attachInfo.mHardwareRenderer.draw(mView, attachInfo, this,  
                    animating ? null : mCurrentDirty);  
        } else {  
            ...  
  
            // 使用软件渲染绘制  
            if (!drawSoftware(surface, attachInfo, yoff, scalingRequired, dirty)) {  
                return;  
            }  
        }  
    }  
  
    ...  
}  
```

在draw方法中，我们首先根据传入的布尔值计算出一个dirty的矩形区域（脏区域，表示要绘制的区域），然后使用硬件或软件的方法进行渲染绘制，由于博主对硬件渲染不熟悉，这里我们分析软件方式渲染绘制的drawSoftware方法：
（android.view.ViewRootImpl）
```java
/** 
 * @return true if drawing was succesfull, false if an error occurred 
 */  
private boolean drawSoftware(Surface surface, AttachInfo attachInfo, int yoff,  
        boolean scalingRequired, Rect dirty) {  
  
    // Draw with software renderer.  
    Canvas canvas;  
    try {  
        int left = dirty.left;  
        int top = dirty.top;  
        int right = dirty.right;  
        int bottom = dirty.bottom;  
  
        canvas = mSurface.lockCanvas(dirty);  
  
        ...  
  
    try {  
        ...  
  
            mView.draw(canvas);  
  
        ...  
    } finally {  
        try {  
            surface.unlockCanvasAndPost(canvas);  
        }   
        ...  
    }  
    return true;  
}  
```

在drawSoftware方法中，第15行首先调用mSurface（Surface对象，管理一块用于绘制的缓存区）的lockCanvas方法，通过传入dirty变量（脏区域）锁定获取了一块画布（Canvas对象），然后调用了mView（即DecorView）的draw方法在canvas上进行绘制，最后再使用mSurface的unlockCanvasAndPost方法解锁提交画布，交给底层进行渲染。虽然View的子类重写了draw方法，但他们都调用了super.draw，因此我们接下来一起看看最关键的View的draw方法：
（android.view.View）
```java
public void draw(Canvas canvas) {  
    ...  
  
    /* 
     * Draw traversal performs several drawing steps which must be executed 
     * in the appropriate order: 
     * 
     *      1. Draw the background 
     *      2. If necessary, save the canvas' layers to prepare for fading 
     *      3. Draw view's content 
     *      4. Draw children 
     *      5. If necessary, draw the fading edges and restore layers 
     *      6. Draw decorations (scrollbars for instance) 
     */  
  
    // Step 1, draw the background, if needed  
    int saveCount;  
  
    if (!dirtyOpaque) {  
        ...  
                background.draw(canvas);  
        ...  
    }  
  
    // Step 2, save the canvas' layers  
    ...  
  
    // Step 3, draw the content  
    if (!dirtyOpaque) onDraw(canvas);  
  
    // Step 4, draw the children  
    dispatchDraw(canvas);  
  
    // Step 5, draw the fade effect and restore layers  
    ...  
  
    // Step 6, draw decorations (scrollbars)  
    onDrawScrollBars(canvas);  
  
    if (mOverlay != null && !mOverlay.isEmpty()) {  
        mOverlay.getOverlayView().dispatchDraw(canvas);  
    }  
}  
```

draw的注释解释的非常清晰，其过程主要分为6个步骤：
1. 绘制背景：调用background.draw绘制背景
2. 保存布局为渐变准备
3. 绘制View本身：调用onDraw
4. 绘制子View：调用dispatchDraw
5. 绘制渐变效果并回复布局
6. 绘制装饰品（如滚动条等）

当我们需要为自定义的View绘制时，只需重写onDraw方法即可。

以上即是一个Activity的显示过程的简略总结，其中还有许多细节没有研究，希望以后有时间可以去进一步深入探索。