---
title: scrollTo方法图解，以及Scroller的使用
tags: [android,view]
categories: [android]
date: 2018-04-01 10:16:41
description: scrollTo方法图解、Scroller的使用、scroll原理
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

# scroll原理

Scroll滚动的本质，实际上是针对View显示内容区域的偏移，通过canvas的translate方法来偏移绘制的原点，来达到绘制内容移动的效果

我们可以追踪下View的draw方法来了解scroll的原理：

以下有部分内容转自：https://juejin.im/entry/5948bfabfe88c2006a939278

```java
public void draw(Canvas canvas) {
    final int privateFlags = mPrivateFlags;
    final boolean dirtyOpaque = (privateFlags & PFLAG_DIRTY_MASK) == PFLAG_DIRTY_OPAQUE &&
            (mAttachInfo == null || !mAttachInfo.mIgnoreDirtyState);
    mPrivateFlags = (privateFlags & ~PFLAG_DIRTY_MASK) | PFLAG_DRAWN;

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
        drawBackground(canvas);
    }

    // skip step 2 & 5 if possible (common case)
    final int viewFlags = mViewFlags;
    boolean horizontalEdges = (viewFlags & FADING_EDGE_HORIZONTAL) != 0;
    boolean verticalEdges = (viewFlags & FADING_EDGE_VERTICAL) != 0;
    if (!verticalEdges && !horizontalEdges) {
        // Step 3, draw the content
        if (!dirtyOpaque) onDraw(canvas);

        // Step 4, draw the children
        dispatchDraw(canvas);

        // Overlay is part of the content and draws beneath Foreground
        if (mOverlay != null && !mOverlay.isEmpty()) {
            mOverlay.getOverlayView().dispatchDraw(canvas);
        }

        // Step 6, draw decorations (foreground, scrollbars)
        onDrawForeground(canvas);

        // we're done...
        return;
    }

    ......
}
```

代码有删简，去掉了 Fading 边缘效果的处理代码。不过，我们仍然可以得到一些很重要的信息，其中包括一个 View 的绘制流程。代码注释中写的很详细。

View 绘制流程 
1. 绘制背景 
2. 绘制内容 
3. 绘制 children 
4. 如果有需要，绘制渐隐(fading) 效果 
5. 绘制装饰物 （scrollbars）

大家可能会注意到 dirtyOpaque 这个变量，它代表的是一个 View 是否是实心的，如果不是实心的就要绘制 background，否则就不需要。

```java
private void drawBackground(Canvas canvas) {
    final Drawable background = mBackground;
    if (background == null) {
        return;
    }

    setBackgroundBounds();

    // Attempt to use a display list if requested.
    if (canvas.isHardwareAccelerated() && mAttachInfo != null
            && mAttachInfo.mHardwareRenderer != null) {
        mBackgroundRenderNode = getDrawableRenderNode(background, mBackgroundRenderNode);

        final RenderNode renderNode = mBackgroundRenderNode;
        if (renderNode != null && renderNode.isValid()) {
            setBackgroundRenderNodeProperties(renderNode);
            ((DisplayListCanvas) canvas).drawRenderNode(renderNode);
            return;
        }
    }

    final int scrollX = mScrollX;
    final int scrollY = mScrollY;
    if ((scrollX | scrollY) == 0) {
        background.draw(canvas);
    } else {
        canvas.translate(scrollX, scrollY);
        background.draw(canvas);
        canvas.translate(-scrollX, -scrollY);
    }
}
```

在这里面，倒是看到了 canvas.translate(scrollX, scrollY),但是绘制了背景之后它又立马平移回去了。这里有些莫名其妙。但是，它不是我们的目标，我们的目标是 view.onDraw()。

在 draw()方法中，我们并没有找到线索。那么，我们注意到这个方法中来————dispatchDraw(),注释说它是绘制 children，那么显然它是属于 ViewGroup 中的方法。 ViewGroup.java

```java
protected void dispatchDraw(Canvas canvas) {
    boolean usingRenderNodeProperties = canvas.isRecordingFor(mRenderNode);
    final int childrenCount = mChildrenCount;
    final View[] children = mChildren;
    int flags = mGroupFlags;



    int clipSaveCount = 0;


    // We will draw our child's animation, let's reset the flag
    mPrivateFlags &= ~PFLAG_DRAW_ANIMATION;
    mGroupFlags &= ~FLAG_INVALIDATE_REQUIRED;

    boolean more = false;
    final long drawingTime = getDrawingTime();


    final int transientCount = mTransientIndices == null ? 0 : mTransientIndices.size();
    int transientIndex = transientCount != 0 ? 0 : -1;
    // Only use the preordered list if not HW accelerated, since the HW pipeline will do the
    // draw reordering internally
    final ArrayList<View> preorderedList = usingRenderNodeProperties
            ? null : buildOrderedChildList();
    final boolean customOrder = preorderedList == null
            && isChildrenDrawingOrderEnabled();
    for (int i = 0; i < childrenCount; i++) {
        while (transientIndex >= 0 && mTransientIndices.get(transientIndex) == i) {
            final View transientChild = mTransientViews.get(transientIndex);
            if ((transientChild.mViewFlags & VISIBILITY_MASK) == VISIBLE ||
                    transientChild.getAnimation() != null) {
                more |= drawChild(canvas, transientChild, drawingTime);
            }
            transientIndex++;
            if (transientIndex >= transientCount) {
                transientIndex = -1;
            }
        }

        final int childIndex = getAndVerifyPreorderedIndex(childrenCount, i, customOrder);
        final View child = getAndVerifyPreorderedView(preorderedList, children, childIndex);
        if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE || child.getAnimation() != null) {
            more |= drawChild(canvas, child, drawingTime);
        }
    }
    while (transientIndex >= 0) {
        // there may be additional transient views after the normal views
        final View transientChild = mTransientViews.get(transientIndex);
        if ((transientChild.mViewFlags & VISIBILITY_MASK) == VISIBLE ||
                transientChild.getAnimation() != null) {
            more |= drawChild(canvas, transientChild, drawingTime);
        }
        transientIndex++;
        if (transientIndex >= transientCount) {
            break;
        }
    }



    // mGroupFlags might have been updated by drawChild()
    flags = mGroupFlags;

    if ((flags & FLAG_INVALIDATE_REQUIRED) == FLAG_INVALIDATE_REQUIRED) {
        invalidate(true);
    }


}
```

我们注意到 drawChild() 这个方法。 ViewGroup.java

```java
protected boolean drawChild(Canvas canvas, View child, long drawingTime) {
        return child.draw(canvas, this, drawingTime);
}
```

这里引出了 View.draw(Canvas canvas, ViewGroup parent, long drawingTime) 方法，这个方法不同于 View.draw(Canvas canvas)。

```java
/**
* This method is called by ViewGroup.drawChild() to have each child view draw itself.
*
* This is where the View specializes rendering behavior based on layer type,
* and hardware acceleration.
*/
boolean draw(Canvas canvas, ViewGroup parent, long drawingTime) {
    final boolean hardwareAcceleratedCanvas = canvas.isHardwareAccelerated();
    /* If an attached view draws to a HW canvas, it may use its RenderNode + DisplayList.
    *
    * If a view is dettached, its DisplayList shouldn't exist. If the canvas isn't
    * HW accelerated, it can't handle drawing RenderNodes.
    */
    boolean drawingWithRenderNode = mAttachInfo != null
        && mAttachInfo.mHardwareAccelerated
        && hardwareAcceleratedCanvas;

    boolean more = false;

    final int parentFlags = parent.mGroupFlags;


    Transformation transformToApply = null;
    boolean concatMatrix = false;
    final boolean scalingRequired = mAttachInfo != null && mAttachInfo.mScalingRequired;


    // Sets the flag as early as possible to allow draw() implementations
    // to call invalidate() successfully when doing animations
    mPrivateFlags |= PFLAG_DRAWN;



    int sx = 0;
    int sy = 0;
    if (!drawingWithRenderNode) {
        computeScroll();
        sx = mScrollX;
        sy = mScrollY;
    }

    final boolean drawingWithDrawingCache = cache != null && !drawingWithRenderNode;
    final boolean offsetForScroll = cache == null && !drawingWithRenderNode;

    int restoreTo = -1;
    if (!drawingWithRenderNode || transformToApply != null) {
        restoreTo = canvas.save();
    }
    if (offsetForScroll) {
        canvas.translate(mLeft - sx, mTop - sy);
    } else {
        if (!drawingWithRenderNode) {
            canvas.translate(mLeft, mTop);
        }
        if (scalingRequired) {
            if (drawingWithRenderNode) {
                // TODO: Might not need this if we put everything inside the DL
                restoreTo = canvas.save();
            }
            // mAttachInfo cannot be null, otherwise scalingRequired == false
            final float scale = 1.0f / mAttachInfo.mApplicationScale;
            canvas.scale(scale, scale);
        }
    }




    if (!drawingWithRenderNode) {
    // apply clips directly, since RenderNode won't do it for this draw
    if ((parentFlags & ViewGroup.FLAG_CLIP_CHILDREN) != 0 && cache == null) {
        if (offsetForScroll) {
            canvas.clipRect(sx, sy, sx + getWidth(), sy + getHeight());
        } else {
            if (!scalingRequired || cache == null) {
                canvas.clipRect(0, 0, getWidth(), getHeight());
            } else {
                canvas.clipRect(0, 0, cache.getWidth(), cache.getHeight());
            }
        }
    }

    if (mClipBounds != null) {
        // clip bounds ignore scroll
        canvas.clipRect(mClipBounds);
    }
    }

    if (!drawingWithDrawingCache) {
        if (drawingWithRenderNode) {

        } else {
            // Fast path for layouts with no backgrounds
            if ((mPrivateFlags & PFLAG_SKIP_DRAW) == PFLAG_SKIP_DRAW) {
                mPrivateFlags &= ~PFLAG_DIRTY_MASK;
                dispatchDraw(canvas);
            } else {
                // 在这里调用 draw() 单参数方法。
                draw(canvas);
            }
        }
    } else if (cache != null) {

    } else {


    }

    if (restoreTo >= 0) {
        canvas.restoreToCount(restoreTo);
    }


    return more;
}

```

原本的代码很长，并且涉及到软件绘制和硬件绘制两种不同的流程。为了便于学习，现在剔除了硬件加速绘制流程和一些矩阵变换的代码。

drawingWithRenderNode 变量代表的就是是否要执行硬件加速绘制。

代码运行中，先会调用 computeScroll() 方法，然后将 mScrollX 和 mScrollY 赋值给变量 sx 和 sy 变量。<font color="red">(这也是为什么我们使用Scroller的时候需要重写computeScroll方法的原因)</font>

```java
/**
 * Called by a parent to request that a child update its values for mScrollX
 * and mScrollY if necessary. This will typically be done if the child is
 * animating a scroll using a {@link android.widget.Scroller Scroller}
 * object.
 */
public void computeScroll() {
}
```

在 View 中 computeScroll() 是一个空方法，但注释说的很明白，这个方法是用来更新 mScrollX 和 mScrollY 的。典型用法就是一个 View 通过 Scroller 进行滚动动画（animating a scroll）时在这里更新 mScrollX 和 mScrollY。 

接下来就是最关键的一环了。

```java
final boolean drawingWithDrawingCache = cache != null && !drawingWithRenderNode;
final boolean offsetForScroll = cache == null && !drawingWithRenderNode;

int restoreTo = -1;
if (!drawingWithRenderNode || transformToApply != null) {
    restoreTo = canvas.save();
}
if (offsetForScroll) {
    canvas.translate(mLeft - sx, mTop - sy);
} else {
    if (!drawingWithRenderNode) {
        canvas.translate(mLeft, mTop);
    }

}
```

由于我们研究的目标不是说 View 的绘制是通过之前的缓存绘制，而是全新的绘制，所以 cache == null，offsetForScroll = true。那么，程序就会执行下面这段代码：

```java
canvas.translate(mLeft - sx, mTop - sy);
```

我们苦苦追寻的答案终于来临，canvas 确实平移了。好，我们继续向下。

```java
if ((mPrivateFlags & PFLAG_SKIP_DRAW) == PFLAG_SKIP_DRAW) {
                mPrivateFlags &= ~PFLAG_DIRTY_MASK;
    dispatchDraw(canvas);
} else {
    // 在这里调用 draw() 单参数方法。
    draw(canvas);
}
```

最后的地方调用了 draw(canvas)，而 draw(canvas) 中调用了开发者常见的 onDraw(canvas)。

综上所述，scroll的原理是
1. 我们通过各种方式来更新View的mScrollX以及mScrollY属性
2. 在View的绘制过程中，通过canvas的translate操作平移画布，从而实现显示内容滑动的效果

PS:
- 可以通过 ViewConfiguration.getTouchSlop() 来获取最小能够识别的滑动距离，来判断滑动是否生效
- 通过 VelocityTracker 来计算速度，实现fling的快速滚动效果