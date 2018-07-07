---
title: ClipboardManager Hook总结
tags: [android,源码]
categories: [android]
date: 2016-09-23 11:47:34
description: ClipboardManager Hook总结
---
最近学习了如何Hook Android中的剪贴板服务，特此写下一篇博客记录。
首先说明下什么是Hook，Hook即通过使用代理对象替换系统原有对象，达到增强或修改系统类的功能的手段。一般我们替换的对象都会选择不易改变的静态对象。
下面首先介绍Android中获取系统服务的步骤，然后再介绍如何Hook 剪贴板服务ClipboardManager。
以下代码版本为Android 4.4
首先我们调用Context的getSystemService方法，由于Context的实际功能实现类为 ContextImpl，故我们追踪ContextImpl的代码：


```java
    @Override
    public Object getSystemService(String name) {
        ServiceFetcher fetcher = SYSTEM_SERVICE_MAP.get(name);
        return fetcher == null ? null : fetcher.getService(this);
    }
```


ContextImpl去检查了一个系统服务的缓存HashMap SYSTEM_SERVICE_MAP，如下所示（以下仅截取关键代码）：



```java
    private static final HashMap<String, ServiceFetcher> SYSTEM_SERVICE_MAP =
            new HashMap<String, ServiceFetcher>();

    private static void registerService(String serviceName, ServiceFetcher fetcher) {
        if (!(fetcher instanceof StaticServiceFetcher)) {
            fetcher.mContextCacheIndex = sNextPerContextServiceCacheIndex++;
        }
        SYSTEM_SERVICE_MAP.put(serviceName, fetcher);
    }

    static {

        registerService(ALARM_SERVICE, new ServiceFetcher() {
                public Object createService(ContextImpl ctx) {
                    IBinder b = ServiceManager.getService(ALARM_SERVICE);
                    IAlarmManager service = IAlarmManager.Stub.asInterface(b);
                    return new AlarmManager(service, ctx);
                }});

        registerService(CLIPBOARD_SERVICE, new ServiceFetcher() {
                public Object createService(ContextImpl ctx) {
                    return new ClipboardManager(ctx.getOuterContext(),
                            ctx.mMainThread.getHandler());
                }});
    }
```




ServiceFetcher是ContextImpl的一个内部类，当我们初次调用getService方法时，会调用createService返回结果
SYSTEM_SERVICE_MAP通过静态代码块初始化注册完毕，其中主要的createService返回方式有两种：

1. 返回xxxManager（我们这次实验的剪贴板就是这种形式），其中一般在内部会有接口的缓存，初次获取缓存的方式与下一种方法相同
2. 调用ServiceManager的getService方法返回一个通信用的IBinder，通过IInterface（即IxxxManager）的内部类Stub的asInterface方法转换为可用的接口（可能是实体也是能是代理）

ClipboardManager的关键代码如下：

```java
public class ClipboardManager extends android.text.ClipboardManager {
    private static IClipboard sService;

    static private IClipboard getService() {
        synchronized (sStaticLock) {
            if (sService != null) {
                return sService;
            }
            IBinder b = ServiceManager.getService("clipboard");
            sService = IClipboard.Stub.asInterface(b);
            return sService;
        }
    }
}
```


ClipboardManager实际服务的提供都会使用getService来调用。


先来看看ServiceManager的getService方法：

```java
public final class ServiceManager {
    private static final String TAG = "ServiceManager";

    private static IServiceManager sServiceManager;
    private static HashMap<String, IBinder> sCache = new HashMap<String, IBinder>();

    public static IBinder getService(String name) {
        try {
            IBinder service = sCache.get(name);
            if (service != null) {
                return service;
            } else {
                return getIServiceManager().getService(name);
            }
        } catch (RemoteException e) {
            Log.e(TAG, "error in getService", e);
        }
        return null;
    }

}

```


该方法首先回去检查sCache的缓存service，如果没有则会调用IServiceManager去获取真正的Service服务。


接下来asInterface的关键代码如下（IClipboard.Stub类，实际就是AIDL自动生成的类文件）：

```java
/** Local-side IPC implementation stub class. */
public static abstract class Stub extends android.os.Binder implements android.content.IClipboard {

	private static final java.lang.String DESCRIPTOR = "android.content.IClipboard";

	/** Construct the stub at attach it to the interface. */
	public Stub() {
		this.attachInterface(this, DESCRIPTOR);
	}
	
	/**
 	* Cast an IBinder object into an android.content.IClipboard interface,
 	* generating a proxy if needed.
 	*/
	public static android.content.IClipboard asInterface(android.os.IBinder obj) {
		if ((obj==null)) {
			return null;
		}
		android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
		if (((iin!=null)&&(iin instanceof android.content.IClipboard))) {
			return ((android.content.IClipboard)iin);
		}
		return new android.content.IClipboard.Stub.Proxy(obj);
	}
}
```


在asInterface方法中，首先调用了IBinder对象的queryLocalInterface方法来查找是否本地含有此接口（如果是同一进程就含有），如果不是则返回代理对象。



总的来说，Android获取系统服务步骤如下图所示：
![Service获取_pic1](1.png)


其中我们可以进行Hook的点，主要是红色的两个部分：

1. Hook xxxManager中的sService服务缓存
2. 利用sCache缓存表Hook SystemService中getService返回的IBinder对象


方法一的代码如下：

```java
    /**
     * hook 方法1
     *
     * @throws Exception
     */
    public static void hook1() throws Exception {
        // 加载ClipboardManager类
        Class<?> clipboardManagerClazz = Class
                .forName("android.content.ClipboardManager");
        // 通过getService static方法获取真实IClipboard对象
        Method getServiceMethod = clipboardManagerClazz
                .getDeclaredMethod("getService");
        getServiceMethod.setAccessible(true);
        // 真实IClipboard对象
        Object clipboardManager = getServiceMethod.invoke(null);
        // 获取sService的IClipboard缓存
        Field sServiceFeild = clipboardManagerClazz
                .getDeclaredField("sService");
        sServiceFeild.setAccessible(true);
        // 替换sService
        sServiceFeild.set(null, Proxy
                .newProxyInstance(clipboardManager.getClass().getClassLoader(),
                                  clipboardManager.getClass().getInterfaces(),
                                  new ClipboardManagerProxyHandler(
                                          clipboardManager)));
    }
```


Java动态代理的InvocationHandler接口如下：

```java
/**
 * Created by superxlcr on 2016/9/20.
 * 剪贴板代理处理类
 */
public class ClipboardManagerProxyHandler implements InvocationHandler {

    // 真正的clipboardManager
    private Object clipboardManager;

    public ClipboardManagerProxyHandler(Object clipboardManager) {
        this.clipboardManager = clipboardManager;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws
            Throwable {
        switch (method.getName()) {
            case "getPrimaryClip": // 粘贴内容
                return ClipData.newPlainText(null, "you are hook!");
            case "hasPrimaryClip": // 剪贴板永远有粘贴内容
                return true;
        }
        // 其余情况由真实对象处理
        return method.invoke(clipboardManager, args);
    }

}
```


在这里我们替换掉了，判断剪贴板是否有内容的hasPrimaryClip方法与返回剪贴内容的getPrimaryClip方法，使剪贴板粘贴总是返回you are hook


方法二代码如下所示：

```java
    /**
     * hook方法2
     *
     * @throws Exception
     */
    public static void hook2() throws Exception {
        // 加载ServiceManager类
        Class<?> serviceManagerClazz = Class
                .forName("android.os.ServiceManager");
        // 获取getService方法
        Method getServiceMethod = serviceManagerClazz
                .getMethod("getService", String.class);
        // 获取真正的clipboardManager对象
        IBinder clipboardManagerIBinder = (IBinder) getServiceMethod
                .invoke(null, CLIPBOARD);
        // 获取sCache HashMap缓存
        Field sCacheField = serviceManagerClazz.getDeclaredField("sCache");
        // private变量
        sCacheField.setAccessible(true);
        // static变量
        HashMap<String, IBinder> sCache = (HashMap) sCacheField.get(null);
        // 把代理放入缓存
        sCache.put(CLIPBOARD, (IBinder) Proxy.newProxyInstance(
                clipboardManagerIBinder.getClass().getClassLoader(),
                clipboardManagerIBinder.getClass().getInterfaces(),
                new ClipboardManagerIBinderProxyHandler(
                        clipboardManagerIBinder)));
    }
```


其中对IBinder的动态代理如下：

```java
/**
 * Created by superxlcr on 2016/9/21.
 * 剪贴板通信代理处理类
 */
public class ClipboardManagerIBinderProxyHandler implements InvocationHandler {

    // 真正的clipboardManagerIBinder
    private IBinder clipboardManagerIBinder;

    public ClipboardManagerIBinderProxyHandler(
            IBinder clipboardManagerIBinder) {
        this.clipboardManagerIBinder = clipboardManagerIBinder;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws
            Throwable {
        if (method.getName().equals("queryLocalInterface")) { // 返回伪造代理对象
            // 加载IClipboard内部类Stub
            Class<?> IClipboardStubClazz = Class
                    .forName("android.content.IClipboard$Stub");
            // 获取asInterface方法
            Method asInterfaceMethod = IClipboardStubClazz
                    .getMethod("asInterface", IBinder.class);
            // 通过asInterface static方法，得到真正IClipboard对象
            Object clipboardManager = asInterfaceMethod
                    .invoke(null, clipboardManagerIBinder);
            return Proxy.newProxyInstance(
                    clipboardManager.getClass().getClassLoader(),
                    clipboardManager.getClass().getInterfaces(),
                    new ClipboardManagerProxyHandler(clipboardManager));
        }
        return method.invoke(clipboardManagerIBinder, args);
    }
}
```


这里我们修改了queryLocalInterface方法，使其返回我们代理的IClipboard接口对象，其余部分与方法一相同。


最后附上该工程的地址，有兴趣的同学可以下载看看：https://github.com/superxlcr/ClipboardManagerHook

