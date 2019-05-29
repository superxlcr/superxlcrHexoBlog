---
title: Android常见问题总结（八）
tags: [android,基础知识]
categories: [android]
date: 2019-05-06 14:13:12
description: Android全屏方式对比，ListView的Header与Footer设置Visibility为Gone不起作用，ViewGroup没有调用onDraw方法，ViewPager获取当前显示的Fragment
---
上一篇博客传送门：[Android常见问题总结（七）](/2019/03/29/Android常见问题总结（七）/)

# Android全屏方式对比

博主目前发现两种在Android中实现全屏的方案，分别是：
- 通过 View#setSystemUiVisibility 方法，使用 View#SYSTEM_UI_FLAG_FULLSCREEN
- 通过 Window#addFlags 方法，使用 WindowManager.LayoutParams#FLAG_FULLSCREEN

两者在视觉效果上并没有什么不同，其中，View#SYSTEM_UI_FLAG_FULLSCREEN 说明如下：
```java
    /**
     * Flag for {@link #setSystemUiVisibility(int)}: View has requested to go
     * into the normal fullscreen mode so that its content can take over the screen
     * while still allowing the user to interact with the application.
     *
     * <p>This has the same visual effect as
     * {@link android.view.WindowManager.LayoutParams#FLAG_FULLSCREEN
     * WindowManager.LayoutParams.FLAG_FULLSCREEN},
     * meaning that non-critical screen decorations (such as the status bar) will be
     * hidden while the user is in the View's window, focusing the experience on
     * that content.  Unlike the window flag, if you are using ActionBar in
     * overlay mode with {@link Window#FEATURE_ACTION_BAR_OVERLAY
     * Window.FEATURE_ACTION_BAR_OVERLAY}, then enabling this flag will also
     * hide the action bar.
     *
     * <p>This approach to going fullscreen is best used over the window flag when
     * it is a transient state -- that is, the application does this at certain
     * points in its user interaction where it wants to allow the user to focus
     * on content, but not as a continuous state.  For situations where the application
     * would like to simply stay full screen the entire time (such as a game that
     * wants to take over the screen), the
     * {@link android.view.WindowManager.LayoutParams#FLAG_FULLSCREEN window flag}
     * is usually a better approach.  The state set here will be removed by the system
     * in various situations (such as the user moving to another application) like
     * the other system UI states.
     *
     * <p>When using this flag, the application should provide some easy facility
     * for the user to go out of it.  A common example would be in an e-book
     * reader, where tapping on the screen brings back whatever screen and UI
     * decorations that had been hidden while the user was immersed in reading
     * the book.
     *
     * @see #setSystemUiVisibility(int)
     */
```

主要需要注意的细节有这种方式实现的全屏效果是**短暂的（transient）**，会因为用户的某些操作（如：跳转到其他应用，下拉状态栏等）退出全屏模式

而 WindowManager.LayoutParams#FLAG_FULLSCREEN 说明如下：
```java
        /**
         * Window flag: hide all screen decorations (such as the status bar) while
         * this window is displayed.  This allows the window to use the entire
         * display space for itself -- the status bar will be hidden when
         * an app window with this flag set is on the top layer. A fullscreen window
         * will ignore a value of {@link #SOFT_INPUT_ADJUST_RESIZE} for the window's
         * {@link #softInputMode} field; the window will stay fullscreen
         * and will not resize.
         *
         * <p>This flag can be controlled in your theme through the
         * {@link android.R.attr#windowFullscreen} attribute; this attribute
         * is automatically set for you in the standard fullscreen themes
         * such as {@link android.R.style#Theme_NoTitleBar_Fullscreen},
         * {@link android.R.style#Theme_Black_NoTitleBar_Fullscreen},
         * {@link android.R.style#Theme_Light_NoTitleBar_Fullscreen},
         * {@link android.R.style#Theme_Holo_NoActionBar_Fullscreen},
         * {@link android.R.style#Theme_Holo_Light_NoActionBar_Fullscreen},
         * {@link android.R.style#Theme_DeviceDefault_NoActionBar_Fullscreen}, and
         * {@link android.R.style#Theme_DeviceDefault_Light_NoActionBar_Fullscreen}.</p>
         */
```

主要需要注意的是跟软键盘弹出的一些交互问题

# ListView的Header与Footer设置Visibility为Gone不起作用

常用的ViewGroup，例如LinearLayout，在onMeasure方法内对每个child view执行measure前，会判断child view的visibility是否为gone。如果是gone，则不对这个child view执行measure操作，即这个child view的高度不被计算在linearLayout的高度里面。LinearLayout的measureVertical代码片段
```java
void measureVertical(int widthMeasureSpec, int heightMeasureSpec) {
	...

	// See how tall everyone is. Also remember max width.
	for (int i = 0; i < count; ++i) {
		final View child = getVirtualChildAt(i);
		if (child == null) {
			mTotalLength += measureNullChild(i);
			continue;
		}

		if (child.getVisibility() == View.GONE) {
		   i += getChildrenSkipCount(child, i);
		   continue;
		}

		...

		i += getChildrenSkipCount(child, i);
	}

	...
}
```

view在measure自己时，并不会去判断自己的Visibility是GONE。这个逻辑操作如上述代码所示，是在parent view里面做的。所以当对LinearLayout里面的一个childView设置Visiblility为gone时，这个view不会被measure，最终也不会被显示出来。

在使用ListView时，经常会添加一些headerView、footerView。但是当设置headerView、footerView的visibility为gone时，却发现headerView、footerView虽然没有显示出来，但垂直方向其所占的位置还是被显示出来了，从而出现了空白区域。网上查到的解决办法是：不能直接设置headerView、footerView的Visibility为gone。而是要**在HeaderView、FooterView外面包一层parent view**（FrameLayout RelativeLayout 都可以），并设置layout_height=“wrap_content”。然后对里面的childView设置visibility为GONE、VISIBLE都会生效。查看源码，情况确实如此。

ListView的onMeasure里面，如果ListView的widthMode、heightMode有一个是unspecified时（应该对应于在XML中没有对listView设置layout_width、layout_height），会调用方法measureScrapChild。如果没有unspecified的情况，则会调用measureHeightOfChildren方法，而此方法内部也会调用measureScrapChild方法。查看measureScrapChild方法:

```java
private void measureScrapChild(View child, int position, int widthMeasureSpec, int heightHint) {
	LayoutParams p = (LayoutParams) child.getLayoutParams();
	if (p == null) {
		p = (AbsListView.LayoutParams) generateDefaultLayoutParams();
		child.setLayoutParams(p);
	}
	p.viewType = mAdapter.getItemViewType(position);
	p.isEnabled = mAdapter.isEnabled(position);
	p.forceAdd = true;

	final int childWidthSpec = ViewGroup.getChildMeasureSpec(widthMeasureSpec,
			mListPadding.left + mListPadding.right, p.width);
	final int lpHeight = p.height;
	final int childHeightSpec;
	if (lpHeight > 0) {
		childHeightSpec = MeasureSpec.makeMeasureSpec(lpHeight, MeasureSpec.EXACTLY);
	} else {
		childHeightSpec = MeasureSpec.makeSafeMeasureSpec(heightHint, MeasureSpec.UNSPECIFIED);
	}
	child.measure(childWidthSpec, childHeightSpec);

	// Since this view was measured directly aginst the parent measure
	// spec, we must measure it again before reuse.
	child.forceLayout();
}
```

可以看出，listview在对child view执行measure前，没有判断visibility为gone的情况。
再看看里面的详细逻辑：
- lpHeight>0时，说明在xml或者程序里面设置了一个确定的尺寸，这里没有问题
- else里面，设置mode为UNSPECIFIED，就是让child view自己去决定大小，child view在measure自己时，不会考虑VISIBILITY属性

如果外面包一层parent view（例如LinearLayout），并设置layout_height为wrap_content(按上面的分析，设置match_parent也是可以的），listView会调用调用额外的这个parent view的measure方法。而LinearLayout在measure时，会判断child view的visibility，如果为gone，则会返回0。最终这个额外的parent view返回给list view的尺寸就是0，从而解决了空白区域的问题

# ViewGroup没有调用onDraw方法

结论：经测试得知，在没有background背景的情况下，ViewGroup并不会执行onDraw绘制

View#draw相关代码：
```java
@CallSuper
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

		...

		// we're done...
		return;
	}

	...

	// Step 3, draw the content
	if (!dirtyOpaque) onDraw(canvas);

	// Step 4, draw the children
	dispatchDraw(canvas);

	...
}
```

可以看到想要执行onDraw，我们都需要变量dirtyOpaque为false（这个变量的具体逻辑没有追踪，大意应该是需要有内容进行绘制吧，所以我们需要添加一个背景）
又或者我们可以**重写方法dispatchDraw**，这个是一定会执行的

# ViewPager获取当前显示的Fragment

最近博主使用到了ViewPager与Fragment这一经典的组合，在此记录一下如何在ViewPager中获取当前显示的Fragment

## FragmentManager#findFragmentByTag

当我们使用Viewpager + FragmentPagerAdapter的组合时，加载过的Fragment都会被保留，因此我们可以通过 FragmentManager#findFragmentByTag 来获取相应的 Fragment
根据网上相关的资料以及翻看源码，我们可以使用 FragmentPagerAdapter#makeFragmentName 来获取相应的 tag (该方法是private的，我们可以拷贝出来使用)

具体的源码如下所示：
```java
/**
 * Return a unique identifier for the item at the given position.
 *
 * <p>The default implementation returns the given position.
 * Subclasses should override this method if the positions of items can change.</p>
 *
 * @param position Position within this adapter
 * @return Unique identifier for the item at position
 */
public long getItemId(int position) {
	return position;
}

private static String makeFragmentName(int viewId, long id) {
	return "android:switcher:" + viewId + ":" + id;
}
```

PS：FragmentPagerAdapter#makeFragmentName 这个方法是 private 的，直接调用或者copy并不是非常保险，比较好的做法是把整个FragmentPagerAdapter都拷贝出去

## 重写FragmentPagerAdapter#setPrimaryItem

这个方法在每次viewpager滑动后都会被调用，而object参数就是显示的Fragment
我们可以通过重写该方法，把object存到我们的成员变量中随时读取
不过这种方式有一个缺陷，FragmentPagerAdapter#setPrimaryItem是在 viewpager的滑动监听执行完后才会调用的，因此我们在滑动监听中读取的当前Fragment是不正确的