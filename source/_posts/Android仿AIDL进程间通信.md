---
title: Android仿AIDL进程间通信
tags: [android,通信]
categories: [android]
date: 2016-09-14 15:09:17
description: Android仿AIDL进程间通信
---
最近了解了一些关于AIDL（Android Interface Definition Language）的知识，在此通过模仿AIDL进程间通信的原理加深印象。
首先我们定义两个进程通信使用的接口：

```java
public interface MyRemoteInterface extends IInterface {

	// 接口描述符
	static final String DESCRIPTOR = "MyRemoteInterface";

	// 方法描述符
	static final int TRANSACTION_myFunction = (IBinder.FIRST_CALL_TRANSACTION + 0);
	
	String myFunction(String s); 
	
}
```


这个接口继承自IInterface（IInterface是Binder中接口的基类，所有Binder的接口都要继承它），里面定义了一个我们的方法myFunction，并含有两个静态常量用于表示该接口与该方法。
Binder的进程间通信模型是C/S模式，接下来我们在Server进程实现该接口：

```java
public class MyIRemoteService extends Binder implements MyRemoteInterface {

	private static final String TAG = "MyIRemoteService";

	MyIRemoteService() {
		// 绑定IInterface接口
		attachInterface(this, DESCRIPTOR);
	}

	@Override
	public IBinder asBinder() {
		return this;
	}

	@Override
	protected boolean onTransact(int code, Parcel data, Parcel reply, int flags)
			throws RemoteException {
		switch (code) {
		case INTERFACE_TRANSACTION:
			Log.d(TAG, "INTERFACE_TRANSACTION");
			reply.writeString(DESCRIPTOR);
			return true;
		case TRANSACTION_myFunction:
			Log.d(TAG, "TRANSACTION_myFunction");
			data.enforceInterface(DESCRIPTOR);
			String s;
			s = data.readString();
			String result = this.myFunction(s);
			reply.writeNoException();
			reply.writeString(result);
			return true;
		}
		return super.onTransact(code, data, reply, flags);
	}

	/**
	 * 自己定义的方法，log打印字符串并返回
	 * 
	 * @param s
	 * @return
	 */
	public String myFunction(String s) {
		Log.d(TAG, s);
		return "A message from " + TAG + " : " + s;
	}

}
```


由于MyIRemoteService是Server进程中的通信对象，因此它是一个实体的对象。我们在其构造函数中进行了接口注册，实现了 MyRemoteInterface 中的 asBinder 与 myFunction 两个方法。我们重写了onTransact方法，当Client进程中的代理对象通过transact方法传输数据时，我们判断是否需要执行我们的myFunction方法并返回结果。
我们在Service的onBind方法返回我们的MyIRemoteService对象：

```java
public class MyRemoteService extends Service {

	private final String TAG = "MyService"; 
	
	@Override
	public IBinder onBind(Intent intent) {
		// 返回MyIRemoteService对象
		return new MyIRemoteService();
	}

	@Override
	public void onCreate() {
		super.onCreate();
		Log.v(TAG, "onCreate");
	}

}
```



Service的注册xml如下：

```html
<service
            android:name="com.example.test.MyRemoteService">
            <intent-filter>
                <action android:name="com.example.test.myAidl"/>
            </intent-filter>
        </service>
```




接下来我们来实现Client进程的通信对象：

```java
public class MyIRemoteServiceProxy implements MyRemoteInterface {
	
	// 代理对象
	private IBinder mRemote;
	
	public MyIRemoteServiceProxy(IBinder mRemote) {
		this.mRemote = mRemote;
	}

	@Override
	public IBinder asBinder() {
		return mRemote;
	}
	
	/**
	 * 自己定义的方法，log打印字符串并返回
	 * 
	 * @param s
	 * @return
	 */
	public String myFunction(String s) {
		Parcel data = Parcel.obtain();
		Parcel reply = Parcel.obtain();
		String result = "";
		try {
			data.writeInterfaceToken(DESCRIPTOR);
			data.writeString(s);
			mRemote.transact(TRANSACTION_myFunction, data, reply, 0);
			reply.readException();
			result = reply.readString();
		} catch (RemoteException e) {
			e.printStackTrace();
		} finally {
			reply.recycle();
			data.recycle();
		}
		return result;
	}
	
}
```


由于MyIRemoteServiceProxy是存在于Client进程中的通信对象，因此一般而言它并不是一个实体对象，而是一个代理对象。我们在其构造函数中保存了具有通信能力的IBinder对象，在调用myFunction时，通过transact方法识别要调用的接口与具体方法，并获取Server进程onTransact方法返回的结果。
在Client进程的Activity中，调用过程如下：

```java
public class MyActivity extends Activity {

	private final String TAG = "MyActivity";

	private ServiceConnection connection = new ServiceConnection() {

		@Override
		public void onServiceDisconnected(ComponentName name) {
		}

		@Override
		public void onServiceConnected(ComponentName name, IBinder service) {
			MyIRemoteServiceProxy mRemoteService = new MyIRemoteServiceProxy(service);
			Log.v(TAG, mRemoteService.myFunction("Activity Message!"));
		}
	};

	@Override
	protected void onCreate(Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);
		setContentView(R.layout.activity_main);
		Log.v(TAG, "onCreate");
		
		// 绑定远程Service
		Intent intent = new Intent();
		intent.setAction("com.example.test.myAidl");
		bindService(intent, connection, Context.BIND_AUTO_CREATE);
	}

}
```


此处我们直接给Server进程发送了一条字符串并打印，效果如下：
![仿AIDL效果图_pic1](1.png)
上图中Client进程pid为883，tid为883
Server进程pid为830，主线程tid为830，处理Binder通信子线程tid为841
