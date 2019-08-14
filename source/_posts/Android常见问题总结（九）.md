---
title: Android常见问题总结（九）
tags: [android,基础知识]
categories: [android]
date: 2019-08-07 18:50:20
description: addFooterView与setAdapter顺序问题
---

上一篇博客传送门：[Android常见问题总结（八）](/2019/05/06/Android常见问题总结（八）/)

# addFooterView与setAdapter顺序问题

最近博主遇到了一个 cannot be cast to android.widget.HeaderViewListAdapter 的崩溃，在网上查阅了相关资料，发现是ListView的addFooterView与setAdapter调用顺序不当导致的（addHeaderView同理）

结论：在Android 4.4之前，如果在调用setAdapter之后调用addFooterView，则会在调用removeFooterView的时候抛出异常

原因：

先贴上Android 4.2的listView源码：
http://androidxref.com/4.2_r1/xref/frameworks/base/core/java/android/widget/ListView.java
再贴上Android 4.4的listView源码：
http://androidxref.com/4.4.4_r1/xref/frameworks/base/core/java/android/widget/ListView.java

我们先看下抛出异常的方法removeFooterView：
```java
/**
* Removes a previously-added footer view.
*
* @param v The view to remove
* @return
* true if the view was removed, false if the view was not a footer view
*/
public boolean removeFooterView(View v) {
	if (mFooterViewInfos.size() > 0) {
		boolean result = false;
		if (mAdapter != null && ((HeaderViewListAdapter) mAdapter).removeFooter(v)) {
			if (mDataSetObserver != null) {
				mDataSetObserver.onChanged();
			}
			result = true;
		}
		removeFixedViewInfo(v, mFooterViewInfos);
		return result;
	}
	return false;
}
```

如果我们addFooterView与setAdapter调用顺序不当，当我们调用removeFooterView时，在上面的源码中的第11行，会由于mAdapter强制转换为HeaderViewListAdapter类型而抛出异常

接下来我们来看下setAdapter方法：
```java
/**
 * Sets the data behind this ListView.
 *
 * The adapter passed to this method may be wrapped by a {@link WrapperListAdapter},
 * depending on the ListView features currently in use. For instance, adding
 * headers and/or footers will cause the adapter to be wrapped.
 *
 * @param adapter The ListAdapter which is responsible for maintaining the
 *        data backing this list and for producing a view to represent an
 *        item in that data set.
 *
 * @see #getAdapter()
 */
@Override
public void setAdapter(ListAdapter adapter) {
	...

	if (mHeaderViewInfos.size() > 0|| mFooterViewInfos.size() > 0) {
		mAdapter = new HeaderViewListAdapter(mHeaderViewInfos, mFooterViewInfos, adapter);
	} else {
		mAdapter = adapter;
	}

	...
}
```

在setAdapter方法第19行我们可以看到，如果我们在setAdapter之前添加了headerView或者是footerView的话，则会把我们的adapter包装为HeaderViewListAdapter，因此在这种情况下当我们removeFooterView时，强制转换并不会抛出异常

接下来我们看一下4.2版本的addFooterView方法：
```java
public void addFooterView(View v, Object data, boolean isSelectable) {

	// NOTE: do not enforce the adapter being null here, since unlike in
	// addHeaderView, it was never enforced here, and so existing apps are
	// relying on being able to add a footer and then calling setAdapter to
	// force creation of the HeaderViewListAdapter wrapper

	FixedViewInfo info = new FixedViewInfo();
	info.view = v;
	info.data = data;
	info.isSelectable = isSelectable;
	mFooterViewInfos.add(info);

	// in the case of re-adding a footer view, or adding one later on,
	// we need to notify the observer
	if (mAdapter != null && mDataSetObserver != null) {
		mDataSetObserver.onChanged();
	}
}
```

然后再对比下4.4版本的addFooterView方法：
```java
/**
 * Add a fixed view to appear at the bottom of the list. If addFooterView is
 * called more than once, the views will appear in the order they were
 * added. Views added using this call can take focus if they want.
 * <p>
 * Note: When first introduced, this method could only be called before
 * setting the adapter with {@link #setAdapter(ListAdapter)}. Starting with
 * {@link android.os.Build.VERSION_CODES#KITKAT}, this method may be
 * called at any time. If the ListView's adapter does not extend
 * {@link HeaderViewListAdapter}, it will be wrapped with a supporting
 * instance of {@link WrapperListAdapter}.
 *
 * @param v The view to add.
 * @param data Data to associate with this view
 * @param isSelectable true if the footer view can be selected
 */
public void addFooterView(View v, Object data, boolean isSelectable) {
	final FixedViewInfo info = new FixedViewInfo();
	info.view = v;
	info.data = data;
	info.isSelectable = isSelectable;
	mFooterViewInfos.add(info);

	// Wrap the adapter if it wasn't already wrapped.
	if (mAdapter != null) {
		if (!(mAdapter instanceof HeaderViewListAdapter)) {
			mAdapter = new HeaderViewListAdapter(mHeaderViewInfos, mFooterViewInfos, mAdapter);
		}

		// In the case of re-adding a footer view, or adding one later on,
		// we need to notify the observer.
		if (mDataSetObserver != null) {
			mDataSetObserver.onChanged();
		}
	}
}
```

我们可以看到，为什么4.4及其以上版本就允许addFooterView在setAdapter以后调用，关键的地方在于第27行，发现如果此时mAdapter不为HeaderViewListAdapter，则会将其包装为HeaderViewListAdapter
因此，在4.4及其以上版本，我们并不需要关心setAdapter与addFooterView的调用顺序（headerView同理）