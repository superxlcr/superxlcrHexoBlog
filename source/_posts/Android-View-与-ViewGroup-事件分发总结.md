---
title: Android View 与 ViewGroup 事件分发总结
date: 2016-03-06 20:16:43
tags: [android,view,基础知识]
categories: [android]
description: View 点击事件分发、ViewGroup点击事件分发
---
# View 点击事件分发
当一个控件被点击时，该控件的dispatchTouchEvent方法就会被调用，由于View的子类没有重写该方法（ViewGroup等会再讨论），故该控件的类会不断往基类查找，最终点击事件的分发由View里的 dispatchTouchEvent方法开始进行。
```java
public boolean dispatchTouchEvent(MotionEvent event) {  
        ...  
            //noinspection SimplifiableIfStatement  
            ListenerInfo li = mListenerInfo;  
            if (li != null && li.mOnTouchListener != null && (mViewFlags & ENABLED_MASK) == ENABLED  
                    && li.mOnTouchListener.onTouch(this, event)) {  
                return true;  
            }  
  
            if (onTouchEvent(event)) {  
                return true;  
            }  
        ...  
    return false;  
}  
```
以上为View的dispatchTouchEvent方法的代码，dispatchTouchEvent传入一个带有点击信息的event参数，返回一个布尔值，表示这个点击事件是否已经被消费处理。代码中比较关键的是第5行的if语句：其中判断了mOnTouchListener是否为null，而mOnTouchListener顾名思义就是我们平时经常注册的touch事件监听器了
```java
public void setOnTouchListener(OnTouchListener l) {  
    getListenerInfo().mOnTouchListener = l;  
}
```
以上为View中的注册touch事件监听器代码。
发现touch事件监听器不为null时，View就会调用listener的onTouch方法了，如果onTouch方法返回了true，那么该方法就不会往下继续执行，dispatchTouchEvent就会返回true代表事件被消费掉了。
但如果我们没有注册touch事件监听器，或者在onTouch方法中返回了false，方法就会继续执行进入到onTouchEvent中
```java
	public boolean onTouchEvent(MotionEvent event) {  
        final int viewFlags = mViewFlags;  
  
  
        if ((viewFlags & ENABLED_MASK) == DISABLED) {  
            if (event.getAction() == MotionEvent.ACTION_UP && (mPrivateFlags & PFLAG_PRESSED) != 0) {  
                setPressed(false);  
            }  
            // A disabled view that is clickable still consumes the touch  
            // events, it just doesn't respond to them.  
            return (((viewFlags & CLICKABLE) == CLICKABLE ||  
                    (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE));  
        }  
  
  
        if (mTouchDelegate != null) {  
            if (mTouchDelegate.onTouchEvent(event)) {  
                return true;  
            }  
        }  
  
  
        if (((viewFlags & CLICKABLE) == CLICKABLE ||  
                (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)) {  
            switch (event.getAction()) {  
                case MotionEvent.ACTION_UP:  
                    boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;  
                    if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {  
                        // take focus if we don't have it already and we should in  
                        // touch mode.  
                        boolean focusTaken = false;  
                        if (isFocusable() && isFocusableInTouchMode() && !isFocused()) {  
                            focusTaken = requestFocus();  
                        }  
  
  
                        if (prepressed) {  
                            // The button is being released before we actually  
                            // showed it as pressed.  Make it show the pressed  
                            // state now (before scheduling the click) to ensure  
                            // the user sees it.  
                            setPressed(true);  
                       }  
  
  
                        if (!mHasPerformedLongPress) {  
                            // This is a tap, so remove the longpress check  
                            removeLongPressCallback();  
  
  
                            // Only perform take click actions if we were in the pressed state  
                            if (!focusTaken) {  
                                // Use a Runnable and post this rather than calling  
                                // performClick directly. This lets other visual state  
                                // of the view update before click actions start.  
                                if (mPerformClick == null) {  
                                    mPerformClick = new PerformClick();  
                                }  
                                if (!post(mPerformClick)) {  
                                    performClick();  
                                }  
                            }  
                        }  
  
  
                        if (mUnsetPressedState == null) {  
                            mUnsetPressedState = new UnsetPressedState();  
                        }  
  
  
                        if (prepressed) {  
                            postDelayed(mUnsetPressedState,  
                                    ViewConfiguration.getPressedStateDuration());  
                        } else if (!post(mUnsetPressedState)) {  
                            // If the post failed, unpress right now  
                            mUnsetPressedState.run();  
                        }  
                        removeTapCallback();  
                    }  
                    break;  
  
  
                case MotionEvent.ACTION_DOWN:  
                    mHasPerformedLongPress = false;  
  
  
                    if (performButtonActionOnTouchDown(event)) {  
                        break;  
                    }  
  
  
                    // Walk up the hierarchy to determine if we're inside a scrolling container.  
                    boolean isInScrollingContainer = isInScrollingContainer();  
  
  
                    // For views inside a scrolling container, delay the pressed feedback for  
                    // a short period in case this is a scroll.  
                    if (isInScrollingContainer) {  
                        mPrivateFlags |= PFLAG_PREPRESSED;  
                        if (mPendingCheckForTap == null) {  
                            mPendingCheckForTap = new CheckForTap();  
                        }  
                        postDelayed(mPendingCheckForTap, ViewConfiguration.getTapTimeout());  
                    } else {  
                        // Not inside a scrolling container, so show the feedback right away  
                        setPressed(true);  
                        checkForLongClick(0);  
                    }  
                    break;  
  
  
                case MotionEvent.ACTION_CANCEL:  
                    setPressed(false);  
                    removeTapCallback();  
                    removeLongPressCallback();  
                    break;  
  
  
                case MotionEvent.ACTION_MOVE:  
                    final int x = (int) event.getX();  
                    final int y = (int) event.getY();  
  
  
                    // Be lenient about moving outside of buttons  
                    if (!pointInView(x, y, mTouchSlop)) {  
                        // Outside button  
                        removeTapCallback();  
                        if ((mPrivateFlags & PFLAG_PRESSED) != 0) {  
                            // Remove any future long press/tap checks  
                            removeLongPressCallback();  
  
  
                            setPressed(false);  
                        }  
                    }  
                    break;  
            }  
            return true;  
        }  
  
  
        return false;  
    }
```
以上是View的onTouchEvent的代码。由于代码很长，我们只看关键的部分，从第9行的注释我们可以得知，如果被点击的控件是disable的，那么该方法不会执行任何动作，但如果该控件是clickable的，dispatchTouchEvent方法依然会返回true，该点击事件依然会被消费掉。
接着看到第23行，如果该控件是可点击的，那么代码就进入if语句里面，执行第60行的performClick方法
```java
	public boolean performClick() {  
        sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_CLICKED);  
  
        ListenerInfo li = mListenerInfo;  
        if (li != null && li.mOnClickListener != null) {  
            playSoundEffect(SoundEffectConstants.CLICK);  
            li.mOnClickListener.onClick(this);  
            return true;  
        }  
  
        return false;  
    }  
```
以上就是View中performClick方法的代码，第7行就是我们调用了再熟悉不过的onClickListener的onClick方法了。
综上所述，View的事件分发如下图所示：
![view事件分发示意图](1.png)

# ViewGroup点击事件分发
讲完了View的事件分发机制，我们再来讲讲ViewGroup的事件分发机制。ViewGroup继承自View，然而ViewGroup重写了View中的dispatchTouchEvent方法，代码如下：
```java
	@Override  
    public boolean dispatchTouchEvent(MotionEvent ev) {  
        ...  
  
        boolean handled = false;  
        if (onFilterTouchEventForSecurity(ev)) {  
            ...  
  
            // Check for interception.  
            final boolean intercepted;  
            if (actionMasked == MotionEvent.ACTION_DOWN  
                    || mFirstTouchTarget != null) {  
                final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;  
                if (!disallowIntercept) {  
                    intercepted = onInterceptTouchEvent(ev);  
                    ev.setAction(action); // restore action in case it was changed  
                } else {  
                    intercepted = false;  
                }  
            } else {  
                // There are no touch targets and this action is not an initial down  
                // so this view group continues to intercept touches.  
                intercepted = true;  
            }  
  
            ...  
  
            // Update list of touch targets for pointer down, if needed.  
            final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;  
            TouchTarget newTouchTarget = null;  
            boolean alreadyDispatchedToNewTouchTarget = false;  
            if (!canceled && !intercepted) {  
                ...  
  
                        final boolean customOrder = isChildrenDrawingOrderEnabled();  
                        for (int i = childrenCount - 1; i >= 0; i--) {  
                            final int childIndex = customOrder ?  
                                    getChildDrawingOrder(childrenCount, i) : i;  
                            final View child = children[childIndex];  
                            if (!canViewReceivePointerEvents(child)  
                                    || !isTransformedTouchPointInView(x, y, child, null)) {  
                                continue;  
                            }  
  
                            newTouchTarget = getTouchTarget(child);  
                            if (newTouchTarget != null) {  
                                // Child is already receiving touch within its bounds.  
                                // Give it the new pointer in addition to the ones it is handling.  
                                newTouchTarget.pointerIdBits |= idBitsToAssign;  
                                break;  
                            }  
  
                            resetCancelNextUpFlag(child);  
                            if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {  
                                // Child wants to receive touch within its bounds.  
                                mLastTouchDownTime = ev.getDownTime();  
                                mLastTouchDownIndex = childIndex;  
                                mLastTouchDownX = ev.getX();  
                                mLastTouchDownY = ev.getY();  
                                newTouchTarget = addTouchTarget(child, idBitsToAssign);  
                                alreadyDispatchedToNewTouchTarget = true;  
                                break;  
                            }  
                        }  
                    ...  
            }  
  
            // Dispatch to touch targets.  
            if (mFirstTouchTarget == null) {  
                // No touch targets so treat this as an ordinary view.  
                handled = dispatchTransformedTouchEvent(ev, canceled, null,  
                        TouchTarget.ALL_POINTER_IDS);  
            }...  
        }  
  
        if (!handled && mInputEventConsistencyVerifier != null) {  
            mInputEventConsistencyVerifier.onUnhandledEvent(ev, 1);  
        }  
        return handled;  
    }  
```
我们依然是找关键代码来看，在第15行有一个非常关键的方法：onInterceptTouchEvent，该方法的返回值赋值给了布尔变量intercepted，而在第32行intercepted的取值决定了是否进入该if语句，而该if语句就是ViewGroup分发touchEvent给他的子View的关键所在。
ViewGroup中的onInterceptTouchEvent方法如下：
```java
	/** 
     * Implement this method to intercept all touch screen motion events.  This 
     * allows you to watch events as they are dispatched to your children, and 
     * take ownership of the current gesture at any point. 
     * 
     * <p>Using this function takes some care, as it has a fairly complicated 
     * interaction with {@link View#onTouchEvent(MotionEvent) 
     * View.onTouchEvent(MotionEvent)}, and using it requires implementing 
     * that method as well as this one in the correct way.  Events will be 
     * received in the following order: 
     * 
     * <ol> 
     * <li> You will receive the down event here. 
     * <li> The down event will be handled either by a child of this view 
     * group, or given to your own onTouchEvent() method to handle; this means 
     * you should implement onTouchEvent() to return true, so you will 
     * continue to see the rest of the gesture (instead of looking for 
     * a parent view to handle it).  Also, by returning true from 
     * onTouchEvent(), you will not receive any following 
     * events in onInterceptTouchEvent() and all touch processing must 
     * happen in onTouchEvent() like normal. 
     * <li> For as long as you return false from this function, each following 
     * event (up to and including the final up) will be delivered first here 
     * and then to the target's onTouchEvent(). 
     * <li> If you return true from here, you will not receive any 
     * following events: the target view will receive the same event but 
     * with the action {@link MotionEvent#ACTION_CANCEL}, and all further 
     * events will be delivered to your onTouchEvent() method and no longer 
     * appear here. 
     * </ol> 
     * 
     * @param ev The motion event being dispatched down the hierarchy. 
     * @return Return true to steal motion events from the children and have 
     * them dispatched to this ViewGroup through onTouchEvent(). 
     * The current target will receive an ACTION_CANCEL event, and no further 
     * messages will be delivered here. 
     */  
    public boolean onInterceptTouchEvent(MotionEvent ev) {  
        return false;  
    } 
```
注释非常长，而代码仅仅只是返回了一个false。由注释可知，该方法的作用是返回当前ViewGroup是否拦截点击的touchEvent，如果该方法返回false，则dispatchTouchEvent方法将会进入第32行的if语句，执行第54行的dispatchTransformedTouchEvent方法：
```java
	private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,  
            View child, int desiredPointerIdBits) {  
        final boolean handled;  
  
        // Canceling motions is a special case.  We don't need to perform any transformations  
        // or filtering.  The important part is the action, not the contents.  
        final int oldAction = event.getAction();  
        if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {  
            event.setAction(MotionEvent.ACTION_CANCEL);  
            if (child == null) {  
                handled = super.dispatchTouchEvent(event);  
            } else {  
                handled = child.dispatchTouchEvent(event);  
            }  
            event.setAction(oldAction);  
            return handled;  
        }  
  
        ...  
    } 
```
该方法为ViewGroup的一个private方法，他的返回值表示该点击事件是否被消费掉，方法的第1个参数即是我们的点击事件，我们重点注意方法的第3个参数child，在第10行的if语句中，如果child为null，则把touchEvent交给基类处理（就是把当前ViewGroup当成一个普通的View进行事件分发处理），如果child不为null，则把touchEvent交给child（子View）来处理。
在dispatchTouchEvent的第54行我们传入了child的参数，故我们在尝试把touchEvent交给ViewGroup的子View处理。然而当onInterceptTouchEvent返回了true或子View处理点击事件失败时，我们就会执行dispatchTouchEvent方法的第71行，此时我们在此调用了dispatchTransformedTouchEvent方法，但是这次我们传入的child参数为null，故该方法会把点击事件交给ViewGroup的基类处理。
综上所述，ViewGroup的点击事件分发流程如下：
![viewGroup点击事件分发示意图](2.png)