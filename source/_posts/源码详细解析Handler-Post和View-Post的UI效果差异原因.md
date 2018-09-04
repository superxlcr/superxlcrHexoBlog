---
title: '源码详细解析Handler#Post和View#Post的UI效果差异原因'
tags: [android,源码,view]
categories: [android]
date: 2018-09-03 20:05:15
description: 前言、源码、总结
---

# 前言

一般有些需求中，会出现需要我们在Activity启动中获取UI控件相关大小或者在界面绘制完成之后刷新数据
我们都知道，在UI绘制完成之后执行相应的操作时机最好，不会阻塞主线程导致卡顿或者UI控件参数获取失败
也许大家使用过或知道Handler(MainLooper)#Post和View#Post都是把Runnable封装成Message再push到主线程中的looper的MessageQueue中，会发现在Activity的生命周期中执行这两种方式效果不同，前者不满足我们的需求，而后者却能做到
接下来，本文就从Activity启动流程以及UI刷新和绘制流程原理以及消息循环机制、同步障碍机制来剖析

下面是一个获取UI控件大小为例子：
```java

class MyActivity extends Activity {
	.....

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        myCustomView = findViewById(R.id.custom);
        Log.i("test", "onCreate init myCustomView  width=" + myCustomView.getWidth());
    }

    @Override
    protected void onResume() {
        super.onResume();
        Log.i("test", "Main onResume");
        new Handler().post(new Runnable() {
            @Override
            public void run() {
                Log.i("test", "onResume Handler post runnable button width=" + myCustomView.getWidth());
            }
        });
        myCustomView.post(new Runnable() {
            @Override
            public void run() {
                Log.i("test", "onResume myCustomView post runnable button width=" + myCustomView.getWidth());
            }
        });
    }

    public class MyView extends View {
        @Override
        public void layout(int l, int t, int r, int b) {
            super.layout(l, t, r, b);
            Log.i("test", "myView layout");
        }

        @Override
        protected void onAttachedToWindow() {
            Log.i("test", "myView onAttachedToWindow with" + getWidth());
            try {
                Object mAttachInfo = ReflectUtils.getDeclaredField(this, View.class, "mAttachInfo");
                Log.i("test", "myView onAttachedToWindow mAttachInfo=null?" + (mAttachInfo == null));
                Object mRunQueue = ReflectUtils.getDeclaredField(this, View.class, "mRunQueue");
                Log.i("test", "myView onAttachedToWindow mRunQueue=null?" + (mRunQueue == null));
            } catch (Exception e) {

            }
            super.onAttachedToWindow();

        }

        @Override
        public boolean post(Runnable action) {
            try {
                Object mAttachInfo = ReflectUtils.getDeclaredField(this, View.class, "mAttachInfo");
                Log.i("test", "myView post mAttachInfo=null?" + (mAttachInfo == null));
            } catch (Exception e) {

            }

            return super.post(action);
        }

        @Override
        protected void onDraw(Canvas canvas) {
            super.onDraw(canvas);
            Log.i("test", "myView onDraw");
        }
    }
}
```

日志显示结果：

```
onCreate init myCustomView  width=0
Main onResume
myView post mAttachInfo=null?true
onResume Handler post runnable button width=0
myView onAttachedToWindow width=0
myView onAttachedToWindow mAttachInfo=null?false
myView layout
myView onDraw
onResume myCustomView post runnable button width=854 
```

从日志中可以看出:

- 在Activity可交互之前的生命周期中UI直接操作是失效的，即使通过handler把UI操纵任务post到onResume生命周期之后，也依然获失效，日志可以看到此时UI界面都没有绘制
- View#post会让runnable在该View完成了measure、layout、draw之后再执行，这个时候当然就可以获取到UI相关参数了

# 源码

先看下两者的源码实现：

handler#post

```java
public final boolean post(Runnable r) {
	return  sendMessageDelayed(getPostMessage(r), 0);
}

public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
	MessageQueue queue = mQueue;
	if (queue == null) {
		RuntimeException e = new RuntimeException(this + " sendMessageAtTime() called with no mQueue");
		Log.w("Looper", e.getMessage(), e);
		return false;
	}
	return enqueueMessage(queue, msg, uptimeMillis);
}

private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
	msg.target = this;
	if (mAsynchronous) {
		msg.setAsynchronous(true);
	}
	return queue.enqueueMessage(msg, uptimeMillis);
}
```

代码简单，可以看到就是把runnable封装成Message然后加入当前Looper的MessageQueue队列中

再看下View#post

```java
public boolean post(Runnable action) {
	final AttachInfo attachInfo = mAttachInfo;
	// 先是尝试通过attachInfo.mHandler.post来实现，实际上就是用Handler.post和上述一样
	if (attachInfo != null) {
		return attachInfo.mHandler.post(action);
	}
	// 若attachInfo == null时，维护一个mRunQueue队列，然后在dispatchAttachedToWindow通过mRunQueue.executeActions(info.mHandler)把action加入队列
	getRunQueue().post(action);
	return true;
}

void dispatchAttachedToWindow(AttachInfo info, int visibility) {
	mAttachInfo = info;
	if (mOverlay != null) {
		mOverlay.getOverlayView().dispatchAttachedToWindow(info, visibility);
	}
	...
	// Transfer all pending runnables.
	if (mRunQueue != null) {
		mRunQueue.executeActions(info.mHandler);
		mRunQueue = null;
	}
	performCollectViewAttributes(mAttachInfo, visibility);
	onAttachedToWindow();

	...
}

private HandlerActionQueue getRunQueue() {
	if (mRunQueue == null) {
		mRunQueue = new HandlerActionQueue();
	}
	return mRunQueue;
}
```

HandlerActionQueue.class

```java
// 实际也是通过handler来post到主线程
public void executeActions(Handler handler) {
	synchronized (this) {
		final HandlerAction[] actions = mActions;
		for (int i = 0, count = mCount; i < count; i++) {
			final HandlerAction handlerAction = actions[i];
			handler.postDelayed(handlerAction.action, handlerAction.delay);
		}
		mActions = null;
		mCount = 0;
	}
}
```

重点来了,通过源码调用发现最终都是通过handler#post方式来加入到主线程队列中，api调用一样为何效果不一样，下面就从如下几个知识点来分析：

1. Activity生命周期启动流程
2. Message消息发送和执行原理机制
3. UI绘制刷新触发原理机制
4. MessageQueue同步障碍机制   

## Activity启动流程

这个流程不清楚的，可以网上搜，一大堆。但这里讲的是，ApplicationThread收到AMS的scheduleLaunchActivity的Binder消息之后，所在的binder线程，会通过ActivityThread中的mH（Handler）来sendMessage

```java
private class ApplicationThread extends ApplicationThreadNative {

    @Override
    public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
                                             ActivityInfo info, Configuration curConfig, Configuration overrideConfig,
                                             CompatibilityInfo compatInfo, String referrer, IVoiceInteractor voiceInteractor,
                                             int procState, Bundle state, PersistableBundle persistentState,
                                             List<ResultInfo> pendingResults, List<ReferrerIntent> pendingNewIntents,
                                             boolean notResumed, boolean isForward, ProfilerInfo profilerInfo) {

        updateProcessState(procState, false);

        ActivityClientRecord r = new ActivityClientRecord();
		
		...

        sendMessage(H.LAUNCH_ACTIVITY, r);
    }

    private void sendMessage(int what, Object obj, int arg1, int arg2, boolean async) {
        if (DEBUG_MESSAGES) Slog.v(
                TAG, "SCHEDULE " + what + " " + mH.codeToString(what)
                        + ": " + arg1 + " / " + obj);
        Message msg = Message.obtain();
        msg.what = what;
        msg.obj = obj;
        msg.arg1 = arg1;
        msg.arg2 = arg2;
        if (async) {
            msg.setAsynchronous(true);
        }
        mH.sendMessage(msg);
    }
}
```

mH（Handler）会把这个异步消息加入到MainLooper中MessageQueue，等到执行时候回调handleLaunchActivity

```java
public void handleMessage(Message msg) {
	if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
	switch (msg.what) {
		case LAUNCH_ACTIVITY: 
			final ActivityClientRecord r = (ActivityClientRecord) msg.obj;
			r.packageInfo = getPackageInfoNoCheck(r.activityInfo.applicationInfo, r.compatInfo);
			handleLaunchActivity(r, null);
			Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
		break;
	...
```

handleLaunchActivity方法会执行很多方法，这个是入口，简单来说会创建Activity对象，调用其启动生命周期，attach、onCreate、onStart、onResume，以及添加到WindowManager中，重点看下本文中onResume生命周期是如何回调的。在Activity可见之后，紧接着就是要触发绘制界面了，会走到handleResumeActivity方法，会在performResumeActivity中调用activity的onResume方法

```java
	final void handleResumeActivity(IBinder token, boolean clearHide, boolean isForward, boolean reallyResume) {
        // 调用activity的onResume方法
        ActivityClientRecord r = performResumeActivity(token, clearHide);
        ...
        if (r != null) {
            final Activity a = r.activity;
            final int forwardBit = isForward ? WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION : 0;
			...
            if (r.window == null && !a.mFinished && willBeVisible) {
                r.window = r.activity.getWindow();
                View decor = r.window.getDecorView();
                // decorView先暂时隐藏
                decor.setVisibility(View.INVISIBLE);
                ViewManager wm = a.getWindowManager();
                WindowManager.LayoutParams l = r.window.getAttributes();
                a.mDecor = decor;
                l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
                l.softInputMode |= forwardBit;
                if (a.mVisibleFromClient) {
                    a.mWindowAdded = true;
                    // 关键函数 添加到window触发UI测量、布局、绘制
                    wm.addView(decor, l);
                }
				...
                r.activity.mVisibleFromServer = true;
                mNumVisibleActivities++;
                if (r.activity.mVisibleFromClient) {
                    // 添加decorView之后，设置可见，从而显示了activity的界面
                    r.activity.makeVisible();
                }
            }
        }
    }

    public final ActivityClientRecord performResumeActivity(IBinder token, boolean clearHide) {
        ActivityClientRecord r = mActivities.get(token);

        if (r != null && !r.activity.mFinished) {
			...
            try {
                r.activity.onStateNotSaved();
                r.activity.mFragments.noteStateNotSaved();
                // 调用activity的onResume方法
                r.activity.performResume();
				...
            } catch (Exception e) {
				...
            }
        }
        return r;
    }

```

由此可见：从handleResumeActivity执行流程来看，到onResume调用的时候，Activity中的UI界面并没有经过measure、layout、draw等流程，所以**直接在onResume或者之前的onCreate中执行UI操纵都是无用的，因为这个时候UI界面是不可见的，并没有进行绘制**
那为何通过hander#post让UI操作执行发生在handleLaunchActivity这个Message之后，还是不行呢？
Message消息发送和执行原理机制这里就不阐述了，hander#post能让执行发生在handleLaunchActivity这个Message之后，就是因为这个Message循环机制原理，可以让任务通常依据Message加入的先后顺序依次执行，所以我们在onResume中push的Message，就会在handleLaunchActivity这个Message之后执行
但是为何onResume中使用hander#post还不能UI操作呢，我们猜测其实handleLaunchActivity之后还没有同步完成UI绘制，那么我们接下来一起分析下UI绘制刷新触发原理机制

## UI绘制刷新触发原理机制

我们直接分析触发条件，从上文中的wm.addView开始：
WindowManager会通过其子类WindowManagerImpl来实现，其内部又通过WindowManagerGlobal的单实例来实现addView方法，源码如下 

WindowManagerGlobal.class

```java
public void addView(View view, ViewGroup.LayoutParams params, Display display, Window parentWindow) {
	...
	ViewRootImpl root;
	View panelParentView = null;
	...
	// ViewRootImpl实际控制着整个UI操作流程
	root = new ViewRootImpl(view.getContext(), display);
	view.setLayoutParams(wparams);
	mViews.add(view);
	mRoots.add(root);
	mParams.add(wparams);
	...
	// do this last because it fires off messages to start doing things
	try {
	// 绑定decorView，并触发开始绘制
		root.setView(view, wparams, panelParentView);
	} catch (RuntimeException e) {

	}
}
```

在addView中，我们调用了ViewRootImpl#setView，具体源码如下： 

ViewRootImpl.class

```java
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
	synchronized (this) {
		if (mView == null) {
			mView = view;
			...
			// 触发刷新绘制的关键
			requestLayout();
			if ((mWindowAttributes.inputFeatures & WindowManager.LayoutParams.INPUT_FEATURE_NO_INPUT_CHANNEL) ==
					0) {
				mInputChannel = new InputChannel();
			}
			...
			try {
				mOrigWindowType = mWindowAttributes.type;
				mAttachInfo.mRecomputeGlobalAttributes = true;
				collectViewAttributes();
				// 通过binder call添加到Display
				res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
						getHostVisibility(), mDisplay.getDisplayId(),
						mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
						mAttachInfo.mOutsets, mInputChannel);
			} catch (RemoteException e) {

			} finally {
				if (restore) {
					attrs.restore();
				}
			}
			...
			// decorView添加父类ViewParent，其实就是ViewRootImpl
			view.assignParent(this);
		}
	}
}
```

setView完成了上述几个重要步骤，我们来看看requestLayout的实现是如何触发刷新绘制的：

```java
@Override
public void requestLayout() {
	if (!mHandlingLayoutInLayoutRequest) {
		checkThread();
		mLayoutRequested = true;
		// 安排刷新请求
		scheduleTraversals();
	}
｝
final TraversalRunnable mTraversalRunnable = new TraversalRunnable();
final class TraversalRunnable implements Runnable {
	@Override
	public void run() {
		doTraversal();
	}
}  

void scheduleTraversals() {
	// 一个刷新周期只执行一次即可，屏蔽其他的刷新请求
	if (!mTraversalScheduled) {
		mTraversalScheduled = true;
		// 设置同步障碍Message
		mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
		// 屏幕刷新信号VSYNC 监听回调把mTraversalRunnable（执行doTraversal()） push到主线程，它是个异步Message会优先得到执行，具体看下Choreographer的实现
		mChoreographer.postCallback(Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
		if (!mUnbufferedInputDispatch) {
			scheduleConsumeBatchedInput();
		}
		notifyRendererOfFramePending();
		pokeDrawLockIfNeeded();
	}
}

void doTraversal() {
	if (mTraversalScheduled) {
		mTraversalScheduled = false;
		// 移除同步障碍Message
		mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);

		if (mProfile) {
			Debug.startMethodTracing("ViewAncestor");
		}
		// 真正执行decorView的绘制
		performTraversals();

		if (mProfile) {
			Debug.stopMethodTracing();
			mProfile = false;
		}
	}
}

private void performTraversals() {
	// cache mView since it is used so much below...
	final View host = mView;
	...
	Rect frame = mWinFrame;
	if (mFirst) {
		mFullRedrawNeeded = true;
		mLayoutRequested = true;
		...
		mAttachInfo.mUse32BitDrawingCache = true;
		mAttachInfo.mHasWindowFocus = false;
		mAttachInfo.mWindowVisibility = viewVisibility;
		mAttachInfo.mRecomputeGlobalAttributes = false;
		// performTraversals 第一次调用时候decorView dispatch mAttachInfo变量
		host.dispatchAttachedToWindow(mAttachInfo, 0);
		mAttachInfo.mTreeObserver.dispatchOnWindowAttachedChange(true);
		dispatchApplyInsets(host);

		desiredWindowWidth = frame.width();
		desiredWindowHeight = frame.height();
		if (desiredWindowWidth != mWidth || desiredWindowHeight != mHeight) {
			if (DEBUG_ORIENTATION) Log.v(TAG, "View " + host + " resized to: " + frame);
			mFullRedrawNeeded = true;
			mLayoutRequested = true;
			windowSizeMayChange = true;
		}
	}
	...
	// 会根据状态判断是否执行，对mView(decorView)执行view的测量、布局、绘制
	perforMeasure()
	perforLayout()
	perforDraw()
	mFirst=false;
}
```

从上述代码可以发现在addView之后同步执行到requestLayout，再到scheduleTraversals中设置了同步障碍消息
这是个简单的阐述，我们来看下源码实现： 

MessageQueue.class

```java
private int postSyncBarrier(long when) {
	synchronized (this) {
		final int token = mNextBarrierToken++;
		final Message msg = Message.obtain();
		msg.markInUse();
		msg.when = when;
		msg.arg1 = token;

		Message prev = null;
		Message p = mMessages;
		if (when != 0) {
			while (p != null && p.when <= when) {
				prev = p;
				p = p.next;
			}
		}
		if (prev != null) { // invariant: p == prev.next
			msg.next = p;
			prev.next = msg;
		} else {
			msg.next = p;
			mMessages = msg;
		}
		return token;
	}
}

// 根据token移动这个Message
public void removeSyncBarrier(int token) {
	// Remove a sync barrier token from the queue.
	// If the queue is no longer stalled by a barrier then wake it.
	synchronized (this) {
		Message prev = null;
		Message p = mMessages;
		while (p != null && (p.target != null || p.arg1 != token)) {
			prev = p;
			p = p.next;
		}
		if (p == null) {
			throw new IllegalStateException("The specified message queue synchronization "
					+ " barrier token has not been posted or has already been removed.");
		}
		final boolean needWake;
		if (prev != null) {
			prev.next = p.next;
			needWake = false;
		} else {
			mMessages = p.next;
			needWake = mMessages == null || mMessages.target != null;
		}
		p.recycleUnchecked();

		// If the loop is quitting then it is already awake.
		// We can assume mPtr != 0 when mQuitting is false.
		if (needWake && !mQuitting) {
			nativeWake(mPtr);
		}
	}
}
```

MessageQueue同步障碍机制： 可以发现就是把一条Message（注意这个Message是没有设置target的）直接手动循环移动链表插入到合适time的Message之后的即可
然后消息队列是如何识别这个障碍消息的呢，我们可以看下Looper#loop循环中获取MessageQueue#next获取下一个message是如何实现的

```java
Message next() {

	final long ptr = mPtr;
	if (ptr == 0) {
		return null;
	}

	int pendingIdleHandlerCount = -1; // -1 only during first iteration
	int nextPollTimeoutMillis = 0;
	for (;;) {
		if (nextPollTimeoutMillis != 0) {
			Binder.flushPendingCommands();
		}
		// 查询是否有下一个消息，没有就阻塞
		nativePollOnce(ptr, nextPollTimeoutMillis);

		synchronized (this) {

			final long now = SystemClock.uptimeMillis();
			Message prevMsg = null;
			Message msg = mMessages;
			// 关键地方，首先识别 msg.target == null情况就是同步障碍消息，如果该消息是同步障碍消息的话，就会循环查询下一个消息是否是isAsynchronous状态，异步Message，专门给刷新UI消息使用的
			if (msg != null && msg.target == null) {
				// Stalled by a barrier.  Find the next asynchronous message in the queue.
				do {
					prevMsg = msg;
					msg = msg.next;
				} while (msg != null && !msg.isAsynchronous());
			}
			// 如果查到异步消息或者没有设置同步障碍的消息，直接返回执行
			if (msg != null) {
				if (now < msg.when) {
					// Next message is not ready.  Set a timeout to wake up when it is ready.
					nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
				} else {
					// Got a message.
					mBlocked = false;
					if (prevMsg != null) {
						prevMsg.next = msg.next;
					} else {
						mMessages = msg.next;
					}
					msg.next = null;
					if (DEBUG) Log.v(TAG, "Returning message: " + msg);
					msg.markInUse();
					return msg;
				}
			} else {
				// No more messages.
				nextPollTimeoutMillis = -1;
			}
		}
		...
		nextPollTimeoutMillis = 0;
	}
}
```

可以看到scheduleTraversals中设置了同步障碍消息，就是相当于在MessageQueue中插入了一个Message，并且是在onResume之后插入的，所以在onResume中使用handler#post加入的消息会在同步障碍Message之前，会先被执行，这个时候依然没有刷新绘制界面，所以在onResume中handler#post发送的消息中进行UI操作是失效的
待消息队列查询到同步障碍Message时候，会等待下个异步Message（刷新Message）出现
那么为何View#post就可以了呢，再回过头来看下其源码： 

View.class

```java
public boolean post(Runnable action) {
	final AttachInfo attachInfo = mAttachInfo;
	if (attachInfo != null) {
		return attachInfo.mHandler.post(action);
	}

	// Postpone the runnable until we know on which thread it needs to run.
	// Assume that the runnable will be successfully placed after attach.
	getRunQueue().post(action);
	return true;
}
```

由于在onResume中执行，这个时候ViewRootImpl还没有初始化（在WindowManager#addView中才初始化），而mAttachInfo是在ViewRootImpl构造函数中初始化的，此时mAttachInfo == null
从上文我们知道，此时的消息都被添加进了View里面的mRunQueue队列中
然后在dispatchAttachedToWindow中，我们通过mRunQueue.executeActions(info.mHandler)这段代码，把mRunQueue中任务全部push到主线程中
那这个方法dispatchAttachedToWindow什么会被调用，回顾上文中ViewRootImpl第一次收到Vsync同步刷新信号之后会执行performTraversals，这个函数内部做了个判断，当第一次mFirst时候会调用
```
host.dispatchAttachedToWindow(mAttachInfo, 0);
```
然后把全局mAttachInfo下发给所有子View，其源码如下： 

View.class

```java
void dispatchAttachedToWindow(AttachInfo info, int visibility) {
	mAttachInfo = info;
	if (mOverlay != null) {
	// 向下分发info,其实现在ViewGroup中
		mOverlay.getOverlayView().dispatchAttachedToWindow(info, visibility);
	}
	mWindowAttachCount++;
	// We will need to evaluate the drawable state at least once
	...
	// Transfer all pending runnables.
	if (mRunQueue != null) {
		mRunQueue.executeActions(info.mHandler);
		mRunQueue = null;
	}
	performCollectViewAttributes(mAttachInfo, visibility);
	onAttachedToWindow();
	...

}
```

可以看到这个函数同时执行了 
```
mRunQueue.executeActions(info.mHandler);
```
从上文可知就是通过hander把mRunQueue中任务全部push到主线程中。由此可以知道在performTraversals中push Message到主线中，肯定会这个performTraversals之后再执行，并且在doTraversals中移除了同步障碍消息，故会依次执行。所以onResume中View.post的Message就会在performTraversals之后执行，而performTraversals就是完成了View整个测量、布局和绘制
另外，当View的mAttachInfo != null时消息是直接post到主线程中的，但因为mAttachInfo不为空，也说明了肯定完成过UI绘制

# 总结

本文主要通过四个知识点，分析了Handler#Post和View#Post的UI效果差异原因
说明了为何Handler#Post无法保证在UI绘制后执行，而View#Post却可以
1. Activity生命周期启动流程 
2. Message消息发送和执行原理机制 
3. UI绘制刷新触发原理机制 
4. MessageQueue同步障碍机制