---
title: Android常见问题总结（七）
tags: [android,基础知识]
categories: [android]
date: 2019-03-29 21:24:18
description: 如何判断当前网络类型、关于Android resources资源的问题、adb shell dumpsys 指令使用、ListView中getView反复调用问题、ViewTreeObserver造成内存泄漏问题
---
上一篇博客传送门：[Android常见问题总结（六）](/2017/08/08/Android常见问题总结（六）/)


# 如何判断当前网络类型

想要判断Android设备当前的网络类型，我们可以使用ConnectivityManager类

通过ConnectivityManager#getActiveNetworkInfo我们可以获取NetworkInfo类，它包含了当前网络相关的信息
我们可以通过NetworkInfo#isAvailable来判断是否连上了网络
通过NetworkInfo#getType来判断当前网络是否wifi类型

至于移动网络的类型，我们可以通过NetworkInfo#getSubtype获取网络的类型，然后通过TelephonyManager#getNetworkClass来判断当前的网络究竟是那种具体类型（不过这个方法是hide，估计是官方觉得不准确就不公开了，我们可以打开源码把该方法拷贝出来使用）
具体判断网络类型的代码如下：


```java
    public static final String NETWORK_WIFI = "Wifi";
    public static final String NETWORK_2G = "2G";
    public static final String NETWORK_3G = "3G";
    public static final String NETWORK_4G = "4G";
    public static final String NETWORK_OTHER = "Other";
    public static final String NETWORK_NONE = "None";
	
	/**
     * 获取当前网络类型
     * @param context 上下文
     * @return 网络类型
     *
     * @see #NETWORK_NONE
     * @see #NETWORK_WIFI
     * @see #NETWORK_2G
     * @see #NETWORK_3G
     * @see #NETWORK_4G
     * @see #NETWORK_OTHER
     */
    public static String getNetworkDetailType(Context context) {
        if (context == null) {
            return NETWORK_NONE;
        }
        try {
            ConnectivityManager manager = (ConnectivityManager) context.getSystemService(Context.CONNECTIVITY_SERVICE);
            NetworkInfo info = manager.getActiveNetworkInfo();
            // 判断是否无网络
            if (info == null || !info.isAvailable()) {
                return NETWORK_NONE;
            }
            // 是否wifi
            if (info.getType() == ConnectivityManager.TYPE_WIFI) {
                return NETWORK_WIFI;
            }
            /**
             * 判断移动网络类型，可见
             * @see TelephonyManager#getNetworkClass
             */
            switch (info.getSubtype()) {
                case TelephonyManager.NETWORK_TYPE_GPRS:
                case TelephonyManager.NETWORK_TYPE_EDGE:
                case TelephonyManager.NETWORK_TYPE_CDMA:
                case TelephonyManager.NETWORK_TYPE_1xRTT:
                case TelephonyManager.NETWORK_TYPE_IDEN:
                    return NETWORK_2G;
                case TelephonyManager.NETWORK_TYPE_UMTS:
                case TelephonyManager.NETWORK_TYPE_EVDO_0:
                case TelephonyManager.NETWORK_TYPE_EVDO_A:
                case TelephonyManager.NETWORK_TYPE_HSDPA:
                case TelephonyManager.NETWORK_TYPE_HSUPA:
                case TelephonyManager.NETWORK_TYPE_HSPA:
                case TelephonyManager.NETWORK_TYPE_EVDO_B:
                case TelephonyManager.NETWORK_TYPE_EHRPD:
                case TelephonyManager.NETWORK_TYPE_HSPAP:
                    return NETWORK_3G;
                case TelephonyManager.NETWORK_TYPE_LTE:
                    return NETWORK_4G;
                default:
                    return NETWORK_OTHER;
            }
        } catch (Exception e) {
            L.exception(e);
        }
        return NETWORK_NONE;
    }
```






# 关于Android resources资源的问题

可以参考官方文档解决问题：https://developer.android.com/guide/topics/resources/overview.html





# adb shell dumpsys 指令使用

该命令用于打印出当前系统信息，默认打印出设备中所有service的信息，可以在命令后面加指定的service name.

有两种方法可以查看service list:
- adb shell dumpsys：输出信息的开始部分就是所有运行的service
- adb shell service list

只要我们在指令后添加对应service name，就能查看指定service的信息：

adb shell dumpsys activity （查看activity堆栈相关信息）
adb shell dumpsys display （查看显示相关信息，可以查看分辨率）


其中，有些service还可以带上额外的参数，我们可以使用 -h 来查看帮助信息：
adb shell dumpsys activity -h （可以查到top等参数的用法）

# ListView中getView反复调用问题

最近在项目中需要在ListView中实现计时器
用TextView配合CountDownTimer实现完成后发现界面卡顿严重，通过debug发现ListView的Adapter在疯狂的调用getView方法
猜测是由于CountDownTimer定时使用TextView#setText刷新文案，而TextView大小为wrap_content导致界面需要重新测量布局，导致ListView会反复调用Adapter#getView
最终通过把TextView大小设为定值解决问题

此次顺便发现了刚进入页面时，Adapter#getView对于相同的position会调用多次的问题
猜测也是ListView需要测量反复调用Adapter#getView所致，通过把ListView在xml中的高度由wrap_content改为match_parent或定值解决问题
最终刚进入页面时，相同的position只会调用一次Adapter#getView

# ViewTreeObserver造成内存泄漏问题

ViewTreeObserver的一般用法如下：
```java
// 添加监听器
view.getViewTreeObserver().addXXXListener(...);
// 移除监听器
view.getViewTreeObserver().removeXXXListener(...);
```

如果我们添加了监听器，而没有在View detach之前执行相应的移除方法，则会造成内存泄漏
下面我们一起来看下具体的源码了解原因：
View#getViewTreeObserver
```java
public ViewTreeObserver getViewTreeObserver() {
	if (mAttachInfo != null) {
		return mAttachInfo.mTreeObserver;
	}
	if (mFloatingTreeObserver == null) {
		mFloatingTreeObserver = new ViewTreeObserver(mContext);
	}
	return mFloatingTreeObserver;
}
```

当view还没有attach到window的时候，其mAttachInfo为空，此时我们添加的监听器会被保存在mFloatingTreeObserver中

View#dispatchAttachedToWindow
```java
void dispatchAttachedToWindow(AttachInfo info, int visibility) {
	mAttachInfo = info;
	...
	if (mFloatingTreeObserver != null) {
		info.mTreeObserver.merge(mFloatingTreeObserver);
		mFloatingTreeObserver = null;
	}

	...
}
```

第5行可以看到attach到window的时候，会执行merge方法，把监听器合并到mAttachInfo中

ViewTreeObserver#merge
```java
    void merge(ViewTreeObserver observer) {
        if (observer.mOnWindowAttachListeners != null) {
            if (mOnWindowAttachListeners != null) {
                mOnWindowAttachListeners.addAll(observer.mOnWindowAttachListeners);
            } else {
                mOnWindowAttachListeners = observer.mOnWindowAttachListeners;
            }
        }

        if (observer.mOnWindowFocusListeners != null) {
            if (mOnWindowFocusListeners != null) {
                mOnWindowFocusListeners.addAll(observer.mOnWindowFocusListeners);
            } else {
                mOnWindowFocusListeners = observer.mOnWindowFocusListeners;
            }
        }

        if (observer.mOnGlobalFocusListeners != null) {
            if (mOnGlobalFocusListeners != null) {
                mOnGlobalFocusListeners.addAll(observer.mOnGlobalFocusListeners);
            } else {
                mOnGlobalFocusListeners = observer.mOnGlobalFocusListeners;
            }
        }

        if (observer.mOnGlobalLayoutListeners != null) {
            if (mOnGlobalLayoutListeners != null) {
                mOnGlobalLayoutListeners.addAll(observer.mOnGlobalLayoutListeners);
            } else {
                mOnGlobalLayoutListeners = observer.mOnGlobalLayoutListeners;
            }
        }

        if (observer.mOnPreDrawListeners != null) {
            if (mOnPreDrawListeners != null) {
                mOnPreDrawListeners.addAll(observer.mOnPreDrawListeners);
            } else {
                mOnPreDrawListeners = observer.mOnPreDrawListeners;
            }
        }

        if (observer.mOnDrawListeners != null) {
            if (mOnDrawListeners != null) {
                mOnDrawListeners.addAll(observer.mOnDrawListeners);
            } else {
                mOnDrawListeners = observer.mOnDrawListeners;
            }
        }

        if (observer.mOnTouchModeChangeListeners != null) {
            if (mOnTouchModeChangeListeners != null) {
                mOnTouchModeChangeListeners.addAll(observer.mOnTouchModeChangeListeners);
            } else {
                mOnTouchModeChangeListeners = observer.mOnTouchModeChangeListeners;
            }
        }

        if (observer.mOnComputeInternalInsetsListeners != null) {
            if (mOnComputeInternalInsetsListeners != null) {
                mOnComputeInternalInsetsListeners.addAll(observer.mOnComputeInternalInsetsListeners);
            } else {
                mOnComputeInternalInsetsListeners = observer.mOnComputeInternalInsetsListeners;
            }
        }

        if (observer.mOnScrollChangedListeners != null) {
            if (mOnScrollChangedListeners != null) {
                mOnScrollChangedListeners.addAll(observer.mOnScrollChangedListeners);
            } else {
                mOnScrollChangedListeners = observer.mOnScrollChangedListeners;
            }
        }

        if (observer.mOnWindowShownListeners != null) {
            if (mOnWindowShownListeners != null) {
                mOnWindowShownListeners.addAll(observer.mOnWindowShownListeners);
            } else {
                mOnWindowShownListeners = observer.mOnWindowShownListeners;
            }
        }

        observer.kill();
    }
```

分析过Activity的显示流程我们可以知道，这个attachInfo实际上是由ViewRootImpl来管理的，因此最终所有的监听器会被merge合并到ViewRootImpl中
由于ViewRootImpl一般而言生命周期都是长于普通的View，而View从window执行detach的时候，我们可以看到其实是没有移除这些监听器的
```java
void dispatchDetachedFromWindow() {
	AttachInfo info = mAttachInfo;
	if (info != null) {
		int vis = info.mWindowVisibility;
		if (vis != GONE) {
			onWindowVisibilityChanged(GONE);
			if (isShown()) {
				// Invoking onVisibilityAggregated directly here since the subtree
				// will also receive detached from window
				onVisibilityAggregated(false);
			}
		}
	}

	onDetachedFromWindow();
	onDetachedFromWindowInternal();

	InputMethodManager imm = InputMethodManager.peekInstance();
	if (imm != null) {
		imm.onViewDetachedFromWindow(this);
	}

	ListenerInfo li = mListenerInfo;
	final CopyOnWriteArrayList<OnAttachStateChangeListener> listeners =
			li != null ? li.mOnAttachStateChangeListeners : null;
	if (listeners != null && listeners.size() > 0) {
		// NOTE: because of the use of CopyOnWriteArrayList, we *must* use an iterator to
		// perform the dispatching. The iterator is a safe guard against listeners that
		// could mutate the list by calling the various add/remove methods. This prevents
		// the array from being modified while we iterate it.
		for (OnAttachStateChangeListener listener : listeners) {
			listener.onViewDetachedFromWindow(this);
		}
	}

	if ((mPrivateFlags & PFLAG_SCROLL_CONTAINER_ADDED) != 0) {
		mAttachInfo.mScrollContainers.remove(this);
		mPrivateFlags &= ~PFLAG_SCROLL_CONTAINER_ADDED;
	}

	mAttachInfo = null;
	if (mOverlay != null) {
		mOverlay.getOverlayView().dispatchDetachedFromWindow();
	}

	notifyEnterOrExitForAutoFillIfNeeded(false);
}
```

因此，如果我们注册了相应的监听器，在detach之前没有进行清理的话，我们的监听器会被ViewRootImpl一直持有从而导致内存泄漏
修复方案一般是在View#onDetachedFromWindow中执行相应的清理方法即可