---
title: Android CountDownTimer的使用
tags: [android]
categories: [android]
date: 2018-01-07 20:21:43
description: Android CountDownTimer的使用
---
最近博主需要实现一个倒计时相关的功能，被推荐了Android的CountDownTimer工具类，在此说一下CountDownTimer的使用以及源码的解读
以下是一个总计10秒倒计时，每间隔1秒进行回调的例子：


```java
        CountDownTimer timer = new CountDownTimer(10000, 1000) {
            @Override
            public void onTick(long millisUntilFinished) {
                // do something
            }

            @Override
            public void onFinish() {
                // do something
            }
        };
        timer.start();
```
每间隔1秒，CountDownTimer便会调用onTick回调方法执行相应操作

当倒计时结束后，CountDownTimer会调用onFinish回调方法执行相应的操作

看完CountDownTimer的例子后，我们可以看一下CountDownTimer的源码以加深对该工具类的理解，源码如下：


```java
public abstract class CountDownTimer {

    /**
     * Millis since epoch when alarm should stop.
     */
    private final long mMillisInFuture;

    /**
     * The interval in millis that the user receives callbacks
     */
    private final long mCountdownInterval;

    private long mStopTimeInFuture;
    
    /**
    * boolean representing if the timer was cancelled
    */
    private boolean mCancelled = false;

    /**
     * @param millisInFuture The number of millis in the future from the call
     *   to {@link #start()} until the countdown is done and {@link #onFinish()}
     *   is called.
     * @param countDownInterval The interval along the way to receive
     *   {@link #onTick(long)} callbacks.
     */
    public CountDownTimer(long millisInFuture, long countDownInterval) {
        mMillisInFuture = millisInFuture;
        mCountdownInterval = countDownInterval;
    }

    /**
     * Cancel the countdown.
     */
    public synchronized final void cancel() {
        mCancelled = true;
        mHandler.removeMessages(MSG);
    }

    /**
     * Start the countdown.
     */
    public synchronized final CountDownTimer start() {
        mCancelled = false;
        if (mMillisInFuture <= 0) {
            onFinish();
            return this;
        }
        mStopTimeInFuture = SystemClock.elapsedRealtime() + mMillisInFuture;
        mHandler.sendMessage(mHandler.obtainMessage(MSG));
        return this;
    }


    /**
     * Callback fired on regular interval.
     * @param millisUntilFinished The amount of time until finished.
     */
    public abstract void onTick(long millisUntilFinished);

    /**
     * Callback fired when the time is up.
     */
    public abstract void onFinish();


    private static final int MSG = 1;


    // handles counting down
    private Handler mHandler = new Handler() {

        @Override
        public void handleMessage(Message msg) {

            synchronized (CountDownTimer.this) {
                if (mCancelled) {
                    return;
                }

                final long millisLeft = mStopTimeInFuture - SystemClock.elapsedRealtime();

                if (millisLeft <= 0) {
                    onFinish();
                } else {
                    long lastTickStart = SystemClock.elapsedRealtime();
                    onTick(millisLeft);

                    // take into account user's onTick taking time to execute
                    long lastTickDuration = SystemClock.elapsedRealtime() - lastTickStart;
                    long delay;

                    if (millisLeft < mCountdownInterval) {
                        // just delay until done
                        delay = millisLeft - lastTickDuration;

                        // special case: user's onTick took more than interval to
                        // complete, trigger onFinish without delay
                        if (delay < 0) delay = 0;
                    } else {
                        delay = mCountdownInterval - lastTickDuration;

                        // special case: user's onTick took more than interval to
                        // complete, skip to next interval
                        while (delay < 0) delay += mCountdownInterval;
                    }

                    sendMessageDelayed(obtainMessage(MSG), delay);
                }
            }
        }
    };
}

```


源码并不算长，CountDownTimer作为一个抽象类，其主要方法有如下几个：


- start：开始进行倒计时
- cancel：取消倒计时
- onTick：抽象方法，用于倒计时间隔回调
- onFinish：抽象方法，用于倒计时结束时回调

看过CountDownTimer的源码后，有几个细节我们需要稍微注意一下：

1. 在源码第38行中，CountDownTimer会判断是否倒计时已结束，如果是则调用onFinish方法，否则调用onTick方法。因此，在倒计时的最后一秒时，我们并不会收到onTick的回调，取而代之的是onFinish的回调。
2. 从源码可以看出，CountDownTimer其实与Timer完全没有任何关系，它的倒计时实现是使用Handler机制实现的，因此当我们在非UI线程使用该工具时，需要先初始化Looper
3. 同上，由于CountDownTimer是基于Handler实现的，其处理以及发送message以及回调onTick处于同一线程，因此当我们在回调方法onTick耗时过多时，可能会影响CountDownTimer预估的回调次数（见源码144行）





