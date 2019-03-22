---
title: okhttp源码分析——基本流程（超详细）
tags: [java,计算机网络,源码]
categories: [java]
date: 2019-03-18 19:25:42
description: （转载）前言、分析
---

本文转载自：https://www.jianshu.com/p/37e26f4ea57b

# 前言

最近算是入了源码的坑了，什么东西总想按住ctrl看看源码的模样，这段时间在研究okhttp的源码，发现okhttp的源码完全不是简简单单的几天就可以啃下来的，那就一步一步来吧。
这篇博客主要是从okhttp的总体流程分析源码的执行过程，对okhttp源码有大体上的理解，从全局上看出okhttp的设计思想。

# 分析

## OkHttpClient

既然是流程分析，使用过okhttp的都了解，首先需要初始化一个OkHttpClient对象。OkHttp支持两种构造方式

1.默认方式

```java
public OkHttpClient() {
	this(new Builder());
}
```

可以看到这种方式，不需要配置任何参数，也就是说基本参数都是默认的，调用的是下面的构造函数。

```java
OkHttpClient(Builder builder) {...}
```

2.builder模式，通过Builder配置参数，最后通过builder()方法返回一个OkHttpClient实例。

```
public OkHttpClient build() {
	return new OkHttpClient(this);
}
```

OkHttpClient基本上就这样分析完了，里面的细节基本上就是用于初始化参数和设置参数的方法。所以也必要将大量的代码放上来占内容。。。,这里另外提一点，**从OkHttpClient中可以看出什么设计模式哪**？
1. builder模式
2. 外观模式

## Request

构建完OkHttpClient后就需要构建一个Request对象，查看Request的源码你会发现，你找不多public的构造函数，唯一的一个构造函数是这样的。

```java
Request(Builder builder) {
	this.url = builder.url;
	this.method = builder.method;
	this.headers = builder.headers.build();
	this.body = builder.body;
	this.tag = builder.tag != null ? builder.tag : this;
}
```

这意味着什么，当然我们构建一个request需要用builder模式进行构建，那么就看一下builder的源码。

```java
public Builder newBuilder() {
	return new Builder(this);
}
//builder===================
public Builder() {
	this.method = "GET";
	this.headers = new Headers.Builder();
}

Builder(Request request) {
	this.url = request.url;
	this.method = request.method;
	this.body = request.body;
	this.tag = request.tag;
	this.headers = request.headers.newBuilder();
}

public Request build() {
	if (url == null) throw new IllegalStateException("url == null");
	return new Request(this);
}
```

其实忽略其他的源码，既然这篇博客只是为了从总体流程上分析OkHttp的源码，所以我们主要着重流程源码上的分析。从上面的源码我们会发现，request的构建也是基于builder模式。

## 异步请求

**这里注意一下，这里分析区分一下同步请求和异步请求，但其实实质的执行流程除了异步外，基本都是一致的。**

构建完Request后，我们就需要构建一个Call，一般都是这样的
```java
Call call = mOkHttpClient.newCall(request);
```
那么我们就返回OkHttpClient的源码看看。

```java
/**
* Prepares the {@code request} to be executed at some point in the future.
*/
@Override public Call newCall(Request request) {
//工厂模式
	return RealCall.newRealCall(this, request, false /* for web socket */);
}
```

可以看到，这里实质上调用的是RealCall中的newRealCall方法，但是这里需要注意一点，那就是方法前面的@Override注解，看到这个注解我们就要意识到，这个方法不是继承就是实现接口。

```java
public class OkHttpClient implements Cloneable, Call.Factory, WebSocket.Factory {...}

```

可以看到OkhttpClient实现了Call.Factory接口。

```java
//Call.java
interface Factory {
	Call newCall(Request request);
}

```

从接口源码我们也可以看出，这个接口其实并不复杂，仅仅是定义一个newCall用于创建Call的方法，这里其实用到了**工厂模式**的思想，**将构建的细节交给具体实现，顶层只需要拿到Call对象即可。**
回到主流程，我们继续看RealCall中的newRealCall方法。

```java
final class RealCall implements Call {
...
	static RealCall newRealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
		// Safely publish the Call instance to the EventListener.
		RealCall call = new RealCall(client, originalRequest, forWebSocket);
		call.eventListener = client.eventListenerFactory().create(call);
		return call;
	}
...
}
```

可以看到RealCall实现了Call接口，newRealCall这是一个静态方法，new了一个RealCall对象，并创建了一个eventListener对象，从名字也可以看出，这个是用来监听事件流程，并且从构建方法我们也可以看出，使用了**工厂模式**

```java
private RealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
    this.client = client;
    this.originalRequest = originalRequest;
    this.forWebSocket = forWebSocket;
    //默认创建一个retryAndFollowUpInterceptor过滤器
    this.retryAndFollowUpInterceptor = new RetryAndFollowUpInterceptor(client, forWebSocket);
  }

```

**重点来了**，可以看到，在RealCall的构造函数中，除了基本的赋值外，默认创建一个retryAndFollowUpInterceptor过滤器,**过滤器**可以说是OkHttp的巨大亮点，后续的文章我会详细分析一些过滤器吧（能力有限，尽量全看看）。
现在Call创建完了，一般就到最后一个步骤了,将请求加入调度，一般的代码是这样的。

```java
//请求加入调度
call.enqueue(new Callback() {
	@Override
	public void onFailure(Request request, IOException e) {
	}

	@Override
	public void onResponse(final Response response) throws IOException {
		//String htmlStr =  response.body().string();
	}
});  
```

可以看到这里调用了call的enqueue方法，既然这里的call-&gt;RealCall，所以我们看一下RealCall的enqueue方法。

```java
@Override public void enqueue(Callback responseCallback) {
	synchronized (this) {
		if (executed) throw new IllegalStateException("Already Executed");
		executed = true;
	}
	captureCallStackTrace();
	eventListener.callStart(this);
	client.dispatcher().enqueue(new AsyncCall(responseCallback));
}
```

1.首先利用synchronized加入了对象锁，防止多线程同时调用，这里先判断一下executed是否为true判断当前call是否被执行了，如果为ture，则抛出异常，没有则设置为true。

2.captureCallStackTrace()

```java
private void captureCallStackTrace() {
	Object callStackTrace = Platform.get().getStackTraceForCloseable("response.body().close()");
	retryAndFollowUpInterceptor.setCallStackTrace(callStackTrace);
}
```

可以看到这里大体上可以理解为为retryAndFollowUpInterceptor加入了一个用于追踪堆栈信息的callStackTrace，后面有时间再详细分析一下这部分吧，不影响总体流程理解。

3.
```java
eventListener.callStart(this);
```
可以看到前面构建的eventListener起到作用了，这里先回调callStart方法。

4.
```java
client.dispatcher().enqueue(new AsyncCall(responseCallback));
```
这里我们就需要先回到OkHttpClient的源码中。

```java
public Dispatcher dispatcher() {
	return dispatcher;
}
```

可以看出返回了一个仅仅是返回了一个DisPatcher对象，那么就追到Dispatcher源码中。

```java
synchronized void enqueue(AsyncCall call) {
	if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
		runningAsyncCalls.add(call);
		executorService().execute(call);
	} else {
		readyAsyncCalls.add(call);
	}
}
```

这里先对Dispatcher的成员变量做个初步的认识。

```java
public final class Dispatcher {
	private int maxRequests = 64;
	private int maxRequestsPerHost = 5;
	private @Nullable Runnable idleCallback;

	/** Executes calls. Created lazily. */
	private @Nullable ExecutorService executorService;

	/** Ready async calls in the order they'll be run. */
	private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();

	/** Running asynchronous calls. Includes canceled calls that haven't finished yet. */
	private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>();

	/** Running synchronous calls. Includes canceled calls that haven't finished yet. */
	private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();
	...
}
```

可以看到，这里用三个队列ArrayDeque用于保存Call对象，分为三种状态**异步等待**,**同步running**,**异步running**。
所以这里的逻辑就比较清楚了。

```java
synchronized void enqueue(AsyncCall call) {
	if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
		runningAsyncCalls.add(call);
		executorService().execute(call);
	} else {
	readyAsyncCalls.add(call);
	}
}
```

当正在执行的异步队列个数小于**maxRequest**(64)并且请求同一个主机的个数小于maxRequestsPerHost(5)时，则将这个请求加入异步执行队列**runningAsyncCall**，并用线程池执行这个call，否则加入异步等待队列。这里可以看一下runningCallsForHost方法。

```java
/** Returns the number of running calls that share a host with {@code call}. */
private int runningCallsForHost(AsyncCall call) {
	int result = 0;
	for (AsyncCall c : runningAsyncCalls) {
		if (c.host().equals(call.host())) result++;
	}
	return result;
}
```

其实也是很好理解的，遍历了runningAsyncCalls，记录同一个Host的个数。
现在来看一个AsyncCall的源码，这块基本上是核心执行的地方了。

```java
final class AsyncCall extends NamedRunnable {
	...
}
```

看一个类，首先看一下这个类的结构，可以看到AsyncCall继承了NameRunnable类。

```java
public abstract class NamedRunnable implements Runnable {
	protected final String name;

	public NamedRunnable(String format, Object... args) {
		this.name = Util.format(format, args);
	}

	@Override public final void run() {
		String oldName = Thread.currentThread().getName();
		Thread.currentThread().setName(name);
		try {
			execute();
		} finally {
			Thread.currentThread().setName(oldName);
		}
	}

	protected abstract void execute();
}
```

可以看到NamedRunnable是一个抽象类，首先了Runnable接口，这就很好理解了，接着看run方法，可以看到，这里将当前执行的线程的名字设为我们在构造方法中传入的名字，接着执行execute方法，**finally**再设置回来。所以现在我们理所当然的回到AsyCall找execute方法了。

```java
@Override protected void execute() {
	boolean signalledCallback = false;
	try {
		//异步和同步走的是同样的方式，主不过在子线程中执行
		Response response = getResponseWithInterceptorChain();
		if (retryAndFollowUpInterceptor.isCanceled()) {
			signalledCallback = true;
			responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
		} else {
			signalledCallback = true;
			responseCallback.onResponse(RealCall.this, response);
		}
	} catch (IOException e) {
		if (signalledCallback) {
			// Do not signal the callback twice!
			Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
		} else {
			eventListener.callFailed(RealCall.this, e);
			responseCallback.onFailure(RealCall.this, e);
		}
	} finally {
		client.dispatcher().finished(this);
	}
}
```

终于，找到了**Response**的身影，那么就意味着执行网络请求就在getResponseWithInterceptorChain()方法中，后面的代码其实基本上就是一些接口回调，回调当前Call的执行状态，这里就不分析了，这里我们重点看一下getResponseWithInterceptorChain()这个方法，给我的感觉这个方法就是okHttp的精髓。

```java
Response getResponseWithInterceptorChain() throws IOException {
	// Build a full stack of interceptors.
	List<Interceptor> interceptors = new ArrayList<>();
	interceptors.addAll(client.interceptors());
	//失败和重定向过滤器
	interceptors.add(retryAndFollowUpInterceptor);
	//封装request和response过滤器
	interceptors.add(new BridgeInterceptor(client.cookieJar()));
	//缓存相关的过滤器，负责读取缓存直接返回、更新缓存
	interceptors.add(new CacheInterceptor(client.internalCache()));
	//负责和服务器建立连接
	interceptors.add(new ConnectInterceptor(client));
	if (!forWebSocket) {
		//配置 OkHttpClient 时设置的 networkInterceptors
		interceptors.addAll(client.networkInterceptors());
	}
	//负责向服务器发送请求数据、从服务器读取响应数据(实际网络请求)
	interceptors.add(new CallServerInterceptor(forWebSocket));

	Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null, null, 0,
	originalRequest, this, eventListener, client.connectTimeoutMillis(),
	client.readTimeoutMillis(), client.writeTimeoutMillis());

	return chain.proceed(originalRequest);
}
```

可以看到，这里首先new了一个Interceptor的ArrayList，然后分别加入了各种各样的Interceptor，所以当我们默认创建okHttpClient时，okHttp默认会给我们实现这些过滤器，每个过滤器执行不同的任务，这个思想太屌了有木有，每个过滤器负责自己的任务，各个过滤器间相互不耦合，**高内聚，低耦合，对拓展放开巴拉巴拉**等一系列设计思想有木有，这里可以对比一下Volley源码中的思想，Volley的处理是将缓存，网络请求等一系列操作揉在一起写，导致用户对于Volley的修改只能通过修改源码方式，而修改就必须要充分阅读理解volley整个的流程，可能一部分的修改会影响全局的流程，而这里，将不同的职责的过滤器分别单独出来，用户只需要对关注的某一个功能项进行理解，并可以进行扩充修改，一对比，okHttp在这方面的优势立马体现出来了。这里大概先描述一下几个过滤器的功能：
- retryAndFollowUpInterceptor——失败和重定向过滤器
- BridgeInterceptor——封装request和response过滤器
- CacheInterceptor——缓存相关的过滤器，负责读取缓存直接返回、更新缓存
- ConnectInterceptor——负责和服务器建立连接，连接池等
- networkInterceptors——配置 OkHttpClient 时设置的 networkInterceptors
- CallServerInterceptor——负责向服务器发送请求数据、从服务器读取响应数据(实际网络请求)

添加完过滤器后，就是执行过滤器了,这里也很重要，一开始看比较难以理解。

```java
Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null, null, 0,
	originalRequest, this, eventListener, client.connectTimeoutMillis(),
	client.readTimeoutMillis(), client.writeTimeoutMillis());

return chain.proceed(originalRequest);
```

可以看到这里创建了一个RealInterceptorChain，并调用了proceed方法，这里注意一下**0**这个参数。

```java
public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,
	RealConnection connection) throws IOException {
	if (index >= interceptors.size()) throw new AssertionError();

	calls++;

	// If we already have a stream, confirm that the incoming request will use it.
	if (this.httpCodec != null && !this.connection.supportsUrl(request.url())) {
		throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
		+ " must retain the same host and port");
	}

	// If we already have a stream, confirm that this is the only call to chain.proceed().
	if (this.httpCodec != null && calls > 1) {
		throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
		+ " must call proceed() exactly once");
	}

	// Call the next interceptor in the chain.
	RealInterceptorChain next = new RealInterceptorChain(interceptors, streamAllocation, httpCodec,
		connection, index + 1, request, call, eventListener, connectTimeout, readTimeout,
		writeTimeout);
	Interceptor interceptor = interceptors.get(index);
	Response response = interceptor.intercept(next);

	// Confirm that the next interceptor made its required call to chain.proceed().
	if (httpCodec != null && index + 1 < interceptors.size() && next.calls != 1) {
		throw new IllegalStateException("network interceptor " + interceptor
		+ " must call proceed() exactly once");
	}

	// Confirm that the intercepted response isn't null.
	if (response == null) {
		throw new NullPointerException("interceptor " + interceptor + " returned null");
	}

	if (response.body() == null) {
		throw new IllegalStateException(
		"interceptor " + interceptor + " returned a response with no body");
	}

	return response;
}
```

第一眼看，脑袋可能会有点发麻，稍微处理一下。

```java
public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,
	RealConnection connection) throws IOException {
	if (index >= interceptors.size()) throw new AssertionError();

	calls++;
	...

	// Call the next interceptor in the chain.
	RealInterceptorChain next = new RealInterceptorChain(interceptors, streamAllocation, httpCodec,
		connection, index + 1, request, call, eventListener, connectTimeout, readTimeout,
		writeTimeout);
	Interceptor interceptor = interceptors.get(index);
	Response response = interceptor.intercept(next);

	...

	return response;
}
```

这样就很清晰了，这里index就是我们刚才的0，也就是从0开始，如果index超过了过滤器的个数抛出异常，后面会再new一个RealInterceptorChain，而且会将参数传递，并且**index+1**了，接着获取index的interceptor,并调用intercept方法，传入新new的next对象，这里可能就有点感觉了，这里用了**递归**的思想来完成遍历，为了验证我们的想法，随便找一个interceptor，看一下intercept方法。

```java
public final class ConnectInterceptor implements Interceptor {
	public final OkHttpClient client;

	public ConnectInterceptor(OkHttpClient client) {
		this.client = client;
	}

	@Override public Response intercept(Chain chain) throws IOException {
		RealInterceptorChain realChain = (RealInterceptorChain) chain;
		...暂时没必要看...

		return realChain.proceed(request, streamAllocation, httpCodec, connection);
	}
}

```

可以看到这里我们拿了一个ConnectInterceptor的源码，这里得到chain后，进行相应的处理后，继续调用proceed方法，那么接着刚才的逻辑，index+1,获取下一个interceptor,重复操作，所以现在就很清楚了，这里利用**递归循环**，也就是okHttp最经典的**责任链模式**。

## 同步请求

异步看完，同步其实就很好理解了。

```java
/**
* 同步请求
*/
@Override public Response execute() throws IOException {
	//检查这个call是否运行过
	synchronized (this) {
		if (executed) throw new IllegalStateException("Already Executed");
		executed = true;
	}
	captureCallStackTrace();
	//回调
	eventListener.callStart(this);
	try {
		//将请求加入到同步队列中
		client.dispatcher().executed(this);
		//创建过滤器责任链，得到response
		Response result = getResponseWithInterceptorChain();
		if (result == null) throw new IOException("Canceled");
		return result;
	} catch (IOException e) {
		eventListener.callFailed(this, e);
		throw e;
	} finally {
		client.dispatcher().finished(this);
	}
}
```

可以看到基本上流程都一致，除了是同步执行，核心方法走的还是getResponseWithInterceptorChain()方法。

![pic1](1.png)

okHttp流程

到这里okHttp的流程基本上分析完了，接下来就是对Inteceptor的分析了，这里献上一张偷来的图（原图来源-&gt;[拆轮子系列：拆 OkHttp](https://blog.piasy.com/2016/07/11/Understand-OkHttp/)）便于理解流程，希望能分析完所有的Inteceptor吧！

[OkHttp源码](https://github.com/sdfdzx/okhttp)
