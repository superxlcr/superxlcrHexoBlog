---
title: RxJava2源码阅读
tags: [android,源码]
categories: [android]
date: 2019-01-14 11:28:57
description: 前言、例子引入、源码分析、总结
---

# 前言

ReactiveX，是一套用可观察流来完成异步编程的api，它的官网是：
http://reactivex.io/
而RxJava，则是ReactiveX为Java语言平台提供的api，目前最新版本为2.2.5，github地址为：
https://github.com/ReactiveX/RxJava

博主也在项目中使用了RxJava相关的api，惊叹于其神奇的链式调用，消除了复杂的异步编程层层嵌套导致的回调地狱，并把逻辑流程梳理清晰
最近刚好有空，在阅读了RxJava的源码后，在此写下一篇博客记录一下体会

博主阅读的是RxJava2版本的源码，与RxJava1的版本源码有较大出入
本文旨在分析RxJava2源码中流程相关的部分，基础的使用方式以及一些操作符说明请自行查询相关文档

# 例子引入

假设我们这里有这么一个场景：
有一系列的学生数据，我们需要把他们的所有id打印到logCat，然后再把其中的偶数id再次打印，如果熟悉RxJava的童鞋肯定会写出下面相似的代码：
```java
Observable.create(new ObservableOnSubscribe<Student>() {
	@Override
	public void subscribe(ObservableEmitter<Student> emitter) {
		List<Student> list = getStudentList();
		for (Student student : list) {
			emitter.onNext(student);
		}
	}
})
		.map(new Function<Student, Integer>() {
			@Override
			public Integer apply(Student student) {
				return student.id;
			}
		})
		.observeOn(Schedulers.newThread())
		.doOnNext(new Consumer<Integer>() {
			@Override
			public void accept(Integer integer) {
				Log.i(TAG, String.valueOf(integer));
			}
		})
		.filter(new Predicate<Integer>() {
			@Override
			public boolean test(Integer integer) {
				return integer % 2 == 0;
			}
		})
		.subscribeOn(Schedulers.io())
		.observeOn(AndroidSchedulers.mainThread())
		.subscribe(new Consumer<Integer>() {
			@Override
			public void accept(Integer integer) {
				Log.i(TAG, String.valueOf(integer));
			}
		});
```

简单介绍下上面代码完成的事情：
1. 我们通过Observable#create方法，创建了一个发射学生Student数据的数据源
2. 通过map操作符，我们从Student中获取他们的id
3. 通过observeOn操作符，我们定义下面的工作将在一个新线程中执行
4. 通过doOnNext操作符，我们打印出所有学生的id
5. 通过filter操作符，我们过滤出其中的偶数学生id
6. 通过subscribeOn操作符，我们定义数据源的发射操作在io线程中执行
7. 通过observeOn操作符，我们定义下面的工作将在Android主线程中执行
8. 通过subscribe方法，我们打印出剩下的偶数学生id

# 源码分析

值得注意的是，博主在RxJava的github[例子](https://github.com/ReactiveX/RxJava#simple-background-computation)上发现了一段话，它告诉了我们RxJava的流程设计是怎么样子的，相信当我们浏览完源码后再回来看会有更深的体会
原文如下：

>This style of chaining methods is called a fluent API which resembles the builder pattern. However, RxJava's reactive types are immutable; each of the method calls returns a new Flowable with added behavior.

这段话告诉了我们RxJava中的响应类型是不变的，每次方法的调用都是通过生成一个新的对象来添加新的行为
这很容易就让人联想到了使用装饰器模式实现的javaIo，看完源码之后我们会发现两者确实有共通之处

接下来我们一起来阅读下其中的源码，我们分为三个阶段，来看下RxJava真正的流程是怎样的

## 创建阶段

首先我们来看下响应类型对象的创建流程
就例子而言，我们经过一系列链式调用后（除去subscribe方法）得到的是一个Observable对象

首先我们先来看下Observable#create方法
Observable#create:
```java
public static <T> Observable<T> create(ObservableOnSubscribe<T> source) {
	ObjectHelper.requireNonNull(source, "source is null");
	return RxJavaPlugins.onAssembly(new ObservableCreate<T>(source));
}
```

其中，第2行的ObjectHelper#requireNonNull用于检测传入的数据是否为null，因为在RxJava的数据流中是不允许出现null的，后续类似的代码我们将忽略
第3行的RxJavaPlugins是一个全局的工具类，它允许我们设置一些监听器，在相应的hook点执行我们自己的操作，这里我们默认什么也不做，只返回原本的对象，后续类似的代码我们也将忽略
综上，Observable#create方法返回了一个ObservableCreate对象，并把我们写的ObservableOnSubscribe匿名内部类当做构造参数传入持有

接下来我们来看看Observable#map方法
Observable#map:
```java
public final <R> Observable<R> map(Function<? super T, ? extends R> mapper) {
	ObjectHelper.requireNonNull(mapper, "mapper is null");
	return RxJavaPlugins.onAssembly(new ObservableMap<T, R>(this, mapper));
}
```
map方法也很简单，返回了一个ObservableMap对象，并把Observable#create返回的ObservableCreate对象，以及我们写的Function匿名内部类当做构造参数传入持有
值得注意的是，这里返回的Observable的泛型参数已经改变了

接下来我们来看看Observable#observeOn方法
Observable#observeOn:
```java
public final Observable<T> observeOn(Scheduler scheduler) {
	return observeOn(scheduler, false, bufferSize());
}
public final Observable<T> observeOn(Scheduler scheduler, boolean delayError, int bufferSize) {
	ObjectHelper.requireNonNull(scheduler, "scheduler is null");
	ObjectHelper.verifyPositive(bufferSize, "bufferSize");
	return RxJavaPlugins.onAssembly(new ObservableObserveOn<T>(this, scheduler, delayError, bufferSize));
}
```
该方法返回了一个ObservableObserveOn对象，持有了map返回的ObservableMap对象，以及我们传入的调度器Schedulers.newThread()

接下来我们来看看Observable#doOnNext方法
Observable#doOnNext:
```java
public final Observable<T> doOnNext(Consumer<? super T> onNext) {
	return doOnEach(onNext, Functions.emptyConsumer(), Functions.EMPTY_ACTION, Functions.EMPTY_ACTION);
}
private Observable<T> doOnEach(Consumer<? super T> onNext, Consumer<? super Throwable> onError, Action onComplete, Action onAfterTerminate) {
	ObjectHelper.requireNonNull(onNext, "onNext is null");
	ObjectHelper.requireNonNull(onError, "onError is null");
	ObjectHelper.requireNonNull(onComplete, "onComplete is null");
	ObjectHelper.requireNonNull(onAfterTerminate, "onAfterTerminate is null");
	return RxJavaPlugins.onAssembly(new ObservableDoOnEach<T>(this, onNext, onError, onComplete, onAfterTerminate));
}
```
该方法返回了一个ObservableDoOnEach对象，持有了observeOn返回的ObservableObserveOn对象，我们传入的Consumer对象以及一些我们并不关心的其他东西

接下来我们来看看Observable#filter方法
Observable#filter:
```java
public final Observable<T> filter(Predicate<? super T> predicate) {
	ObjectHelper.requireNonNull(predicate, "predicate is null");
	return RxJavaPlugins.onAssembly(new ObservableFilter<T>(this, predicate));
}
```
该方法返回了一个ObservableFilter对象，持有了doOnNext返回的ObservableDoOnEach对象，以及我们传入的Predicate对象

接下来我们来看看Observable#subscribeOn方法
Observable#subscribeOn:
```java
public final Observable<T> subscribeOn(Scheduler scheduler) {
	ObjectHelper.requireNonNull(scheduler, "scheduler is null");
	return RxJavaPlugins.onAssembly(new ObservableSubscribeOn<T>(this, scheduler));
}
```
该方法返回了一个ObservableSubscribeOn对象，持有了filter返回的ObservableFilter对象，以及我们传入的调度器Schedulers.io()

接下来又调用了Observable#observeOn方法，返回了一个ObservableObserveOn对象，持有了subscribeOn返回的ObservableSubscribeOn对象，以及我们传入的调度器AndroidSchedulers.mainThread()

可见，在创建阶段，我们获得了一个“层层包裹”的ObservableObserveOn对象
就像是在使用java.io中的stream一样，通过装饰器模式“层层包裹”来增强相应的功能：
```java
new BufferedReader(new InputStreamReader(new ByteArrayInputStream()))
```

创建阶段具体流程可用下图表示：
![创建流程](1.png)

## 订阅阶段

接下来，我们对创建阶段得到的ObservableObserveOn对象调用了subscribe方法，这是在父类Observable中的一个final方法，我们一起来看下
Observable#subscribe:
```java
public final Disposable subscribe(Consumer<? super T> onNext) {
	return subscribe(onNext, Functions.ON_ERROR_MISSING, Functions.EMPTY_ACTION, Functions.emptyConsumer());
}
public final Disposable subscribe(Consumer<? super T> onNext, Consumer<? super Throwable> onError,
		Action onComplete, Consumer<? super Disposable> onSubscribe) {
	ObjectHelper.requireNonNull(onNext, "onNext is null");
	ObjectHelper.requireNonNull(onError, "onError is null");
	ObjectHelper.requireNonNull(onComplete, "onComplete is null");
	ObjectHelper.requireNonNull(onSubscribe, "onSubscribe is null");

	LambdaObserver<T> ls = new LambdaObserver<T>(onNext, onError, onComplete, onSubscribe);

	subscribe(ls);

	return ls;
}
public final void subscribe(Observer<? super T> observer) {
	ObjectHelper.requireNonNull(observer, "observer is null");
	try {
		observer = RxJavaPlugins.onSubscribe(this, observer);

		ObjectHelper.requireNonNull(observer, "Plugin returned null Observer");

		subscribeActual(observer);
	} catch (NullPointerException e) { // NOPMD
		throw e;
	} catch (Throwable e) {
		Exceptions.throwIfFatal(e);
		// can't call onError because no way to know if a Disposable has been set or not
		// can't call onSubscribe because the call might have set a Subscription already
		RxJavaPlugins.onError(e);

		NullPointerException npe = new NullPointerException("Actually not, but can't throw other exceptions due to RS");
		npe.initCause(e);
		throw npe;
	}
}
```

在代码第11行，我们可以看到我们传入的监听器onNext被包装成为了一个LambdaObserver对象
接着在第24行，以该对象作为参数调用了ObservableObserveOn#subscribeActual
ObservableObserveOn#subscribeActual:
```java
@Override
protected void subscribeActual(Observer<? super T> observer) {
	if (scheduler instanceof TrampolineScheduler) {
		source.subscribe(observer);
	} else {
		Scheduler.Worker w = scheduler.createWorker();

		source.subscribe(new ObserveOnObserver<T>(observer, w, delayError, bufferSize));
	}
}
```

在subscribeActual方法中，我们传入的scheduler是AndroidSchedulers.mainThread()，它是一个HandlerScheduler，因此走的是分支语句的else逻辑
在这里，我们的LambdaObserver对象被包装成为一个ObserveOnObserver对象，并与scheduler创建的Worker绑定在了一起
接着调用了source的subscribe方法，这里的source即是我们构造ObservableObserveOn时传入的“上游”数据源————ObservableSubscribeOn对象

由于Observable#subscribe是一个final方法，无法被子类重写，并且它最终的逻辑都会调用子类的subscribeActual方法，这里我们直接看ObservableSubscribeOn#subscribeActual
ObservableSubscribeOn#subscribeActual:
```java
@Override
public void subscribeActual(final Observer<? super T> s) {
	final SubscribeOnObserver<T> parent = new SubscribeOnObserver<T>(s);

	s.onSubscribe(parent);

	parent.setDisposable(scheduler.scheduleDirect(new SubscribeTask(parent)));
}
final class SubscribeTask implements Runnable {
	private final SubscribeOnObserver<T> parent;

	SubscribeTask(SubscribeOnObserver<T> parent) {
		this.parent = parent;
	}

	@Override
	public void run() {
		source.subscribe(parent);
	}
}
```

首先在第3行，我们的ObserveOnObserver对象被包装成为了SubscribeOnObserver对象
接着在第5行，调用了ObserveOnObserver#onSubscribe
ObserveOnObserver#onSubscribe:
```java
@Override
public void onSubscribe(Disposable s) {
	if (DisposableHelper.validate(this.s, s)) {
		this.s = s;
		if (s instanceof QueueDisposable) {
			@SuppressWarnings("unchecked")
			QueueDisposable<T> qd = (QueueDisposable<T>) s;

			int m = qd.requestFusion(QueueDisposable.ANY | QueueDisposable.BOUNDARY);

			if (m == QueueDisposable.SYNC) {
				sourceMode = m;
				queue = qd;
				done = true;
				actual.onSubscribe(this);
				schedule();
				return;
			}
			if (m == QueueDisposable.ASYNC) {
				sourceMode = m;
				queue = qd;
				actual.onSubscribe(this);
				return;
			}
		}

		queue = new SpscLinkedArrayQueue<T>(bufferSize);

		actual.onSubscribe(this);
	}
}
```

经过一系列检查后，最终会调用actual对象的onSubscribe方法，而actual对象即是我们构造ObserveOnObserver对象时传入的LambdaObserver对象
LambdaObserver#onSubscribe:
```java
@Override
public void onSubscribe(Disposable s) {
	if (DisposableHelper.setOnce(this, s)) {
		try {
			onSubscribe.accept(this);
		} catch (Throwable ex) {
			Exceptions.throwIfFatal(ex);
			s.dispose();
			onError(ex);
		}
	}
}
```

因为我们再例子中没有传入相应的监听器，所以这里onSubscribe是由Functions.emptyConsumer()创建的，什么也不会做
如果我们传入了相应的监听器，则会在当前线程收到开始订阅的回调通知（onSubscribe）

回到subscribeActual这里，接着第7行调用了传入的scheduler的scheduleDirect方法，执行了特定的任务SubscribeTask
这里我们不去深究具体的代码，由于我们传入的是Schedulers.io()，因此容易得知SubscribeTask#run是在io线程中执行的
在第18行，此时我们已经处于io线程中，同上，我们以SubscribeOnObserver对象作为参数，调用了“上游”数据源source————ObservableFilter的subscribe方法
ObservableFilter#subscribeActual:
```java
@Override
public void subscribeActual(Observer<? super T> s) {
	source.subscribe(new FilterObserver<T>(s, predicate));
}
```

逻辑比较简单，把传入的SubscribeOnObserver对象与构造时传入的Predicate对象绑定在一起，包装成FilterObserver对象后，调用了“上游”数据源source————ObservableDoOnEach的subscribe方法
ObservableDoOnEach#subscribeActual:
```java
@Override
public void subscribeActual(Observer<? super T> t) {
	source.subscribe(new DoOnEachObserver<T>(t, onNext, onError, onComplete, onAfterTerminate));
}
```

逻辑类似，把传入的FilterObserver对象与构造时传入的Consumer对象绑定在一起，包装成DoOnEachObserver对象后，调用了“上游”数据源source————ObservableObserveOn的subscribe方法

与上面提到的一样，ObservableObserveOn#subscribeActual会把DoOnEachObserver对象与scheduler（Schedulers.newThread()）创建的Worker绑定在了一起，包装成ObserveOnObserver对象，交给“上游”数据源————ObservableMap
ObservableMap#subscribeActual:
```java
@Override
public void subscribeActual(Observer<? super U> t) {
	source.subscribe(new MapObserver<T, U>(t, function));
}
```

逻辑类似，把传入的ObserveOnObserver对象与构造时传入的Function对象绑定在一起，包装成MapObserver对象后，交给了“上游”数据源source————ObservableCreate
ObservableCreate#subscribeActual:
```java
@Override
protected void subscribeActual(Observer<? super T> observer) {
	CreateEmitter<T> parent = new CreateEmitter<T>(observer);
	observer.onSubscribe(parent);

	try {
		source.subscribe(parent);
	} catch (Throwable ex) {
		Exceptions.throwIfFatal(ex);
		parent.onError(ex);
	}
}
```

在第3行，传入的MapObserver对象被包装成了CreateEmitter
然后在第4行调用了MapObserver#onSubscribe，不过由于我们前面已经回调了订阅事件，因此这里实际上并不会回调到我们的监听器中，有兴趣的童鞋可以进一步看下相关源码
在第7行，调用了“上游”数据源source————我们构造的ObservableOnSubscribe匿名内部类的subscribe方法

至此，订阅阶段结束，这次被“层层包裹”的是我们的监听器Observer
订阅阶段的流程图如下所示：
![订阅阶段流程图](2.png)

## 发射数据阶段

在订阅阶段的最后，我们调用了ObservableOnSubscribe匿名内部类的subscribe方法，进入了发射数据的阶段
首先，让我们一起来回顾下例子中的代码做了什么：
```java
Observable.create(new ObservableOnSubscribe<Student>() {
	@Override
	public void subscribe(ObservableEmitter<Student> emitter) {
		List<Student> list = getStudentList();
		for (Student student : list) {
			emitter.onNext(student);
		}
	}
})
```

在第4行，我们首先通过getStudentList方法获得了学生数据列表，然后在第6行调用了emitter对象的onNext方法把数据发送了出去
根据上文分析，这里的emitter对象实际类型为CreateEmitter，让我们来看下相关的代码
CreateEmitter#onNext
```java
final Observer<? super T> observer;

CreateEmitter(Observer<? super T> observer) {
	this.observer = observer;
}

@Override
public void onNext(T t) {
	if (t == null) {
		onError(new NullPointerException("onNext called with null. Null values are generally not allowed in 2.x operators and sources."));
		return;
	}
	if (!isDisposed()) {
		observer.onNext(t);
	}
}
```

逻辑比较简单，首先检查了发送的数据是否为null（因为在RxJava数据流中不允许任何null出现）
然后如果是还没Disposed的情况下，把数据传递给了observer对象
从构造函数可以看出，observer对象实际类型为订阅阶段传入的MapObserver对象，我们接着看下相关代码
MapObserver#onNext
```java
final Function<? super T, ? extends U> mapper;

MapObserver(Observer<? super U> actual, Function<? super T, ? extends U> mapper) {
	super(actual);
	this.mapper = mapper;
}

@Override
public void onNext(T t) {
	...

	U v;

	try {
		v = ObjectHelper.requireNonNull(mapper.apply(t), "The mapper function returned a null value.");
	} catch (Throwable ex) {
		fail(ex);
		return;
	}
	actual.onNext(v);
}
```

在第15行，调用了我们在订阅阶段时绑定的Function对象进行映射（map）转换
在进行了非空检查后，调用了actual对象，即ObserveOnObserver对象的onNext方法，继续数据的传递
ObserveOnObserver#onNext
```java
@Override
public void onNext(T t) {
	...

	if (sourceMode != QueueDisposable.ASYNC) {
		queue.offer(t);
	}
	schedule();
}

void schedule() {
	if (getAndIncrement() == 0) {
		worker.schedule(this);
	}
}

@Override
public void run() {
	if (outputFused) {
		drainFused();
	} else {
		drainNormal();
	}
}

void drainNormal() {
	int missed = 1;

	final SimpleQueue<T> q = queue;
	final Observer<? super T> a = actual;

	for (;;) {
		if (checkTerminated(done, q.isEmpty(), a)) {
			return;
		}

		for (;;) {
			boolean d = done;
			T v;

			try {
				v = q.poll();
			} catch (Throwable ex) {
				Exceptions.throwIfFatal(ex);
				s.dispose();
				q.clear();
				a.onError(ex);
				worker.dispose();
				return;
			}
			boolean empty = v == null;

			if (checkTerminated(d, empty, a)) {
				return;
			}

			if (empty) {
				break;
			}

			a.onNext(v);
		}

		missed = addAndGet(-missed);
		if (missed == 0) {
			break;
		}
	}
}
```

在第6行，ObserveOnObserver首先把数据加入了一个队列中，然后再第8行调用了schedule方法开启线程调度
在第13行中，我们看到调用了worker对象的schedule方法，并传入了参数this
这里我们不去深究Scheduler调度器的具体实现，根据上下文的含义，不难推断出，由于我们传入的调度器是Schedulers.newThread()，ObserveOnObserver实现的run方法将会在一个新的线程中被调用
在run方法中，由于我们没有设置结果合并到一起输出，因此进入的是第22行的drainNormal逻辑
最终在第61行，我们会把从队列中取出的数据，在新的线程中，继续传递给“下游”的监听器————DoOnEachObserver

DoOnEachObserver#onNext
```java
DoOnEachObserver(
		Observer<? super T> actual,
		Consumer<? super T> onNext,
		Consumer<? super Throwable> onError,
		Action onComplete,
		Action onAfterTerminate) {
	this.actual = actual;
	this.onNext = onNext;
	this.onError = onError;
	this.onComplete = onComplete;
	this.onAfterTerminate = onAfterTerminate;
}

@Override
public void onNext(T t) {
	if (done) {
		return;
	}
	try {
		onNext.accept(t);
	} catch (Throwable e) {
		Exceptions.throwIfFatal(e);
		s.dispose();
		onError(e);
		return;
	}

	actual.onNext(t);
}
```

逻辑比较简单，交给了绑定的Consumer处理之后，继续把数据传递给“下游”的监听器————FilterObserver
FilterObserver#onNext
```java
final Predicate<? super T> filter;

FilterObserver(Observer<? super T> actual, Predicate<? super T> filter) {
	super(actual);
	this.filter = filter;
}

@Override
public void onNext(T t) {
	if (sourceMode == NONE) {
		boolean b;
		try {
			b = filter.test(t);
		} catch (Throwable e) {
			fail(e);
			return;
		}
		if (b) {
			actual.onNext(t);
		}
	} else {
		actual.onNext(null);
	}
}
```

这里我们没有设置sourceMode，因此默认值为NONE，进入上半部分分支
FilterObserver使用订阅阶段绑定的Predicate对象，通过其test方法的返回值判断哪些数据允许往下传递
在例子中我们过滤条件为数据为偶数类型，因此test方法只有数据为偶数时才返回true，只有偶数数据才会被传递到“下游”————SubscribeOnObserver

SubscribeOnObserver#onNext
```java
@Override
public void onNext(T t) {
	actual.onNext(t);
}
```

啥也没干，直接把数据往“下游”（ObserveOnObserver）传递
ObserveOnObserver上面我们已经分析过了，这里就不再重复分析了，它会在把线程切换到Android主线程后（这里的调度器是AndroidSchedulers.mainThread()），把数据传递给“下游”————LambdaObserver

LambdaObserver#onNext
```java
public LambdaObserver(Consumer<? super T> onNext, Consumer<? super Throwable> onError,
		Action onComplete,
		Consumer<? super Disposable> onSubscribe) {
	super();
	this.onNext = onNext;
	this.onError = onError;
	this.onComplete = onComplete;
	this.onSubscribe = onSubscribe;
}

@Override
public void onNext(T t) {
	if (!isDisposed()) {
		try {
			onNext.accept(t);
		} catch (Throwable e) {
			Exceptions.throwIfFatal(e);
			get().dispose();
			onError(e);
		}
	}
}
```

最终，LambdaObserver会把数据回调给我们一开始实现的Consumer匿名内部类
至此，发射数据阶段结束，其流程图如下所示：
![发射数据阶段流程图](3.png)

# 总结

总而言之，RxJava的执行流程可以分为三个阶段：
1. 创建阶段：使用装饰器模式“层层包裹”创建出一个 reactive type 对象（这里是Observable）
2. 订阅阶段：使用装饰器模式“层层包裹”我们传入的监听器，不断调用“上游”的subscribe；回调开始订阅事件（onSubscribe），切换订阅线程
3. 发射数据阶段：数据由顶层监听器向“下游”逐级传递，传递数据的同时执行相应的变换操作