---
title: Android Looper和Handler小结
tags: [android,thread,handler]
categories: [android]
date: 2016-03-08 17:17:39
description: Looper类分析、Handler类分析、HandlerThread的使用
---
Android系统中的Java应用程序和其他系统上类似，都是靠消息驱动来工作的。他们都拥有一个消息队列，可以往队列中投递新消息，他们还有一个消息循环，用于不断地从队列中取出消息来处理。在Android系统中，这两项工作主要由Looper和Handler来实现。

# Looper类分析
我们先从官方的一个例子来研究：
```java
  class LooperThread extends Thread {  
      public Handler mHandler;  
  
      public void run() {  
          Looper.prepare();  
  
          mHandler = new Handler() {  
              public void handleMessage(Message msg) {  
                  // process incoming messages here  
              }  
          };  
  
          Looper.loop();  
      }  
  }  
```
根据官方文档的说明，在使用Looper的时候，我们得先调用Looper的prepare方法（第5行所示），然后再调用Looper的loop方法（第13行所示）。
我们先来看看Looper的prepare方法：
```java
	/** Initialize the current thread as a looper. 
      * This gives you a chance to create handlers that then reference 
      * this looper, before actually starting the loop. Be sure to call 
      * {@link #loop()} after calling this method, and end it by calling 
      * {@link #quit()}. 
      */  
    public static void prepare() {  
        prepare(true);  
    }  
  
    private static void prepare(boolean quitAllowed) {  
        if (sThreadLocal.get() != null) {  
            throw new RuntimeException("Only one Looper may be created per thread");  
        }  
        sThreadLocal.set(new Looper(quitAllowed));  
    }  
```
从注释我们可以得知这个方法是用来为当前线程初始化一个looper的。在第12行的sThreadLocal变量是一个当前线程局部变量，用于存储当前线程的某些值，拥有get和set两个重要方法。从代码我们可以得知，如果当前线程没有设置looper这个变量，那么prepare方法就会为其设置一个，如果有就会报错。接下来我们来看看Looper的构造函数：
```java
	private Looper(boolean quitAllowed) {  
        mQueue = new MessageQueue(quitAllowed);  
        mThread = Thread.currentThread();  
    }  
```
在构造函数中，looper首先构造了一个消息队列，然后得到了当前线程的对象。
总的来说Looper.prepare方法判断当前线程是否绑定了一个looper对象，如果没有就构造一个含有消息队列looper对象并绑定到当前线程上。
接下来我们来看看Looper的loop方法：
```java
	public static void loop() {  
        final Looper me = myLooper();  
        if (me == null) {  
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");  
        }  
        final MessageQueue queue = me.mQueue;  
  
        ...  
  
        for (;;) {  
            Message msg = queue.next(); // might block  
            if (msg == null) {  
                // No message indicates that the message queue is quitting.  
                return;  
            }  
  
            ...  
  
            msg.target.dispatchMessage(msg);  
  
            ...  
  
            msg.recycleUnchecked();  
        }  
    }  
```
方法的第2行调用了myLooper方法，这个方法返回了当前线程持有的looper对象：
```java
	public static @Nullable Looper myLooper() {  
        return sThreadLocal.get();  
    } 
```
取得当前线程的looper对象后，方法获取了looper对象里面的消息队列，并进入一个循环开始处理消息。
综上所述，Looper的作用是：
	1. 封装了一个消息队列
	2. 使用Looper.prepare方法把looper对象与当前线程绑定在一起
	3. 线程调用Looper.loop方法后，处理消息队列的消息
	
# Handler类分析
接下来我们来看看Handler：
```java
	public Handler(Callback callback, boolean async) {  
        ...  
        mLooper = Looper.myLooper();  
        if (mLooper == null) {  
            throw new RuntimeException(  
                "Can't create handler inside thread that has not called Looper.prepare()");  
        }  
        mQueue = mLooper.mQueue;  
        mCallback = callback;  
        mAsynchronous = async;  
    }  
  
    public Handler(Looper looper, Callback callback, boolean async) {  
        mLooper = looper;  
        mQueue = looper.mQueue;  
        mCallback = callback;  
        mAsynchronous = async;  
    }  
```
```java
	public interface Callback {  
        public boolean handleMessage(Message msg);  
    }  
```
Handler类拥有多个构造函数，但最后都会调用上面这两个构造函数。第一个构造函数获取当前线程的looper对象，第二个构造函数获取传入参数的looper对象，然后获取保存looper对象的消息队列，同时保存了一个callback的回调接口，和一个布尔变量值表示是否异步。
由上面的方法我们知道，为什么调用Handler()之前一定要调用Looper.prepare方法了。
根据官方的文档所知，Handler类其实就是一个用来处理消息的辅助类，里面主要提供了这些方法：
```java
public void handleMessage(Message msg)  
public final Message obtainMessage()  
public final boolean post(Runnable r)  
public final boolean sendMessage(Message msg)  
public final void removeMessages(int what)  
```
对上面的方法稍作分析，我们就能明白其他方法，我们取sendMessage方法为例：
```java
	public final boolean sendMessage(Message msg)  
    {  
        return sendMessageDelayed(msg, 0);  
    }  
	
	public final boolean sendMessageDelayed(Message msg, long delayMillis)  
    {  
        if (delayMillis < 0) {  
            delayMillis = 0;  
        }  
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);  
    }  
	
	public boolean sendMessageAtTime(Message msg, long uptimeMillis) {  
        MessageQueue queue = mQueue;  
        if (queue == null) {  
            RuntimeException e = new RuntimeException(  
                    this + " sendMessageAtTime() called with no mQueue");  
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
从上面的方法我们可以发现，如果没有Handler的辅助，由我们自己来操作消息队列，是多么麻烦的一件事情！
Handler把Message的target设为自己，是因为Handler不仅封装了消息的发送，还封装了消息的处理。看回Looper.loop方法我们发现，最终消息的处理是在第19行：交给了Message对象的target的dispatchMessage方法实现，该方法如下所示：
```java
	public void dispatchMessage(Message msg) {  
        if (msg.callback != null) {  
            handleCallback(msg);  
        } else {  
            if (mCallback != null) {  
                if (mCallback.handleMessage(msg)) {  
                    return;  
                }  
            }  
            handleMessage(msg);  
        }  
    }  
```
该方法定义了一套消息处理的优先级机制：
	1. 如果Message自带了callback，则交给callback处理
	2. 如果Handler设置了全局的mCallback，则交给mCallback处理
	3. 如果上述都没有，则交给Handler子类实现的handleMessage方法处理
一般情况我们都会使用第三种处理方式，因此我们在构造handler对象的时候都会重写里面的handleMessage方法。
看完了消息的投递和处理，我们再来看看如何获取消息：
```java
	public final Message obtainMessage()  
    {  
        return Message.obtain(this);  
    }
	
	public static Message obtain(Handler h) {  
        Message m = obtain();  
        m.target = h;  
  
        return m;  
    } 
	
	/** 
	* Return a new Message instance from the global pool. Allows us to 
	* avoid allocating new objects in many cases. 
	*/  
	public static Message obtain() {  
        synchronized (sPoolSync) {  
            if (sPool != null) {  
                Message m = sPool;  
                sPool = m.next;  
                m.next = null;  
                m.flags = 0; // clear in-use flag  
                sPoolSize--;  
                return m;  
            }  
        }  
        return new Message();  
    }  
```
Handler类中的obtainMessage最终调用了Message类中的obtain方法，该方法首先判断在消息池中是否有可以回收的消息对象，如果有则使用可回收的消息对象，否则就新创建一个消息对象。通过调用该方法我们可以在多种情况下避免创建过多的消息对象，节省了内存空间。
综上所述，Looper类掌握着线程的消息队列，封装了消息循环；而Handler类则是消息处理的辅助类，里面封装了消息的投递、处理和获取等一系列操作。

# HandlerThread的使用
我们考虑如下一段代码：
```java
public class MyActivity extends Activity {  
  
    private static String TAG = "MyActivity";  
  
    private Looper looper;  
      
    @Override  
    protected void onCreate(Bundle savedInstanceState) {  
        super.onCreate(savedInstanceState);  
        setContentView(R.layout.activity_main);  
        Thread myThread = new Thread() {  
            @Override  
            public void run() {  
                Looper.prepare();  
                looper = Looper.myLooper();  
                Looper.loop();  
            }  
        };  
        myThread.start();  
        Handler handler = new Handler(looper); // 有问题！  
    }  
      
}  
```
在代码中我们新建了一个新的线程，并通过为其设置绑定Looper从而在主线程使用Handler与该线程进行通信。
然而我们的代码是有错误的，了解多线程编程的同学应该知道，在并发编程中线程执行的顺序是不可预测的，因此在注释那一行构造Handler时传入的looper参数极有可能为null。
那么我们该如何解决这个问题呢？针对这个问题，Android官方已经为我们提供了一个HandlerThread类供我们使用，我们来看一下其中的关键代码：
```java
@Override  
public void run() {  
    mTid = Process.myTid();  
    Looper.prepare();  
    synchronized (this) {  
        mLooper = Looper.myLooper();  
        notifyAll();  
    }  
    Process.setThreadPriority(mPriority);  
    onLooperPrepared();  
    Looper.loop();  
    mTid = -1;  
}  
  
/** 
 * This method returns the Looper associated with this thread. If this thread not been started 
 * or for any reason is isAlive() returns false, this method will return null. If this thread  
 * has been started, this method will block until the looper has been initialized.   
 * @return The looper. 
 */  
public Looper getLooper() {  
    if (!isAlive()) {  
        return null;  
    }  
      
    // If the thread has been started, wait until the looper has been created.  
    synchronized (this) {  
        while (isAlive() && mLooper == null) {  
            try {  
                wait();  
            } catch (InterruptedException e) {  
            }  
        }  
    }  
    return mLooper;  
}  
```
HandlerThread提供了getLooper方法让我们获取构造线程的Looper对象，从上可以看出Google巧妙地使用了一对wait/notifyAll解决了我们的问题。因此以后要是我们的子线程需要使用Looper与Handler通信时，记得使用官方提供的HandlerThread即可。