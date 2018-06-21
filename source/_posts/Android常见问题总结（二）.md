---
title: Android常见问题总结（二）
tags: [android,基础知识]
categories: [android]
date: 2016-04-24 21:51:06
description: 广播的两种注册方法、Intent的使用、ContentProvider使用方法、Thread、AsycTask、IntentService的使用场景与特点、五种Layout各自特点
---
上一篇文章的传送门：[Android常见问题总结（一）](/2016/04/22/Android常见问题总结（一）/)

# 广播的两种注册方法

Android四大组件之一的broadcast（广播）拥有两种不同的注册方法：静态注册与动态注册

## 静态注册

广播静态注册指把广播相应的信息写在AndroidManifest.xml中，例子如下：
```xml
<receiver android:name=".StaticReceiver">    
   <intent-filter>    
      <action android:name="XXX" />   
   </intent-filter>    
</receiver>  
```

静态广播的好处在于：不需程序启动即可接收，可用作自动启动程序

## 动态注册

广播动态注册指在程序中调用registerReceiver方法来注册广播，例子如下：
```java
IntentFilter filter = new IntentFilter();  
filter.addAction("XXX");  
DynamicReceiver receiver = new DynamicReceiver();  
registerReceiver(receiver,  filter);  
// 不使用后记得取消注册  
unregisterReceiver(receiver);  
```

动态广播的好处在于：程序适应系统变化做操作，但在程序运行状态才能接收到

# Intent的使用

Intent即意图，是Android中连接四大组件的枢纽，Android中的Activity、Service和BroadcastReceiver都依靠Intent来启动
Intent对象的属性大致包含七种，分别是Component、Action、Category、Data、Type、Extra、Flag

## Component

Component用于明确指定需要启动的目标组件，使用方法如下：
```java
Intent intent = new Intent();  
// 方法一：传入上下文参数与class参数  
intent.setComponent(new ComponentName(context, XXX.class));  
// 方法二：传入包名与类名  
intent.setComponent(new ComponentName(pkg, cls));  
```

指定Component属性的Intent已经明确了它将要启动的组件，因此这种Intent也被称为显示Intent，没有指定Component属性的Intent被称为隐试Intent

## Action、Category

Action代表Intent所要完成的一个抽象“动作”，而Category则用于为Action增加额外的附加类别信息，它们的使用方法如下：
```java
Intent intent = new Intent();  
// 设置一个字符串代表Action  
intent.setAction(action);  
// 添加一个字符串代表category  
intent.addCategory(category1);  
intent.addCategory(category2); 
```

值得注意的是Action属性是唯一的，但Category属性可以有多个。通常设置了Action和Category来启动组件的Intent就不指定Component属性了，因此这种Intent被称为隐试Intent

## Data、Type

Data属性接受一个Uri对象，Data属性通常用于向Action属性提供操作的数据。Type属性用于指定该Data属性所指定Uri对应的MIME类型。它们的使用方法如下：
```java
Intent intent = new Intent();  
// 设置Data属性  
intent.setData(new Uri());  
// 设置Type属性  
intent.setType(type);  
// 同时设置Data和Type属性  
intent.setDataAndType(data, type);  
```

值得注意的是Data属性和Type属性会互相覆盖，如果需要同时设置Data属性和Type属性需要使用setDataAndType

## Extra

Intent的Extra属性用于进行数据交换，Intent的Extra属性值应该是一个Bundle对象（一个类似Map的对象，可以存入多个键值对，但存入的对象必须是基本类型或可序列化的），用法如下：
```java
Intent intent = new Intent();  
// 直接往Intent添加基本类型，在方法内也是把数据存入Bundle  
// 该方法有多种重载  
intent.putExtra(name, value);  
// 新建Bundle  
Bundle bundle = new Bundle();  
// 往Bundle添加数据，XXX为基本类型  
bundle.putXXX(key, value);  
bundle.putXXXArray(key, value);  
// 把Bundle添加进Intent  
intent.putExtras(bundle);  
```

## Flag

Intent的Flag属性用于为该Intent添加一些额外的控制旗标，Intent可调用addFlags方法来添加控制旗标

# ContentProvider使用方法

为了在应用程序之间交换数据，Android提供了ContentProvider。当一个应用程序需要把自己的数据暴露给其他程序使用时，该应用程序就可以通过ContentProvider来实现，而其他程序则使用ContentResolver来操作ContentProvider暴露的数据。
实现ContentProvider的Java代码如下：
```java
public class MyProvider extends ContentProvider {  
  
    @Override  
    public boolean onCreate() {  
        // 第一次创建时调用，如果创建成功则返回true  
        // 可以在这里打开数据库什么的  
        return true;  
    }  
  
    @Override  
    public String getType(Uri uri) {  
        // 返回ContentProvider所提供数据的MIME类型  
        return null;  
    }  
  
    @Override  
    public Cursor query(Uri uri, String[] projection, String selection,  
            String[] selectionArgs, String sortOrder) {  
        // 实现查询方法  
        return null;  
    }  
      
    @Override  
    public Uri insert(Uri uri, ContentValues values) {  
        // 实现插入方法，返回插入条数  
        return null;  
    }  
  
    @Override  
    public int delete(Uri uri, String selection, String[] selectionArgs) {  
        // 实现删除方法，返回删除条数  
        return 0;  
    }  
  
    @Override  
    public int update(Uri uri, ContentValues values, String selection,  
            String[] selectionArgs) {  
        // 实现更新方法，返回更新条数  
        return 0;  
    }  
  
}  
```

当我们实现了自己的ContentProvider后，还需要去AndroidManifest.xml中注册才行：
```java
<provider   
    android:name=".MyProvider"  
    android:authorities="com.example.test.provider"  
    android:exported="true" />  
```

配置ContentProvider时我们需要设置如下属性：
- name：类名
- authorities：为ContentProvider指定一个对应的Uri，其他程序通过这个Uri来找到该ContentProvider
- exported：允许ContentProvider被其他应用调用

当其他应用需要访问我们提供的ContentProvider的使用，只需使用ContentResolver并传入相应的Uri即可：
```java
public class MyActivity extends Activity {  
  
    private static String TAG = "MyActivity";  
      
    @Override  
    protected void onCreate(Bundle savedInstanceState) {  
        super.onCreate(savedInstanceState);  
        setContentView(R.layout.activity_main);  
        ContentResolver resolver = getContentResolver();  
        // 传入对应的Uri进行增删查改操作  
        resolver.query(uri, projection, selection, selectionArgs, sortOrder);  
        resolver.insert(url, values);  
        resolver.delete(url, where, selectionArgs);  
        resolver.update(uri, values, where, selectionArgs);  
    }  
      
}  
```

除此之外，我们还可以使用ContentObserver来为ContentProvider添加观察者。我们在其他程序的ContentResolver中注册ContentObserver，当ContentProvider发生改变时，我们在数据改动的使用调用：
```java
getContext().getContentResolver().notifyChange(uri, null);  
```

那么此时ContentObserver中的onChange回调方法就会被调用，我们就可以简单地监听ContentProvider的数据改变了

# Thread、AsycTask、IntentService的使用场景与特点

Thread、AsyncTask和IntentService都与多线程有关，当我们在Android中涉及并发编程时（进行网络请求、加载较大的文件等操作）就需要使用

## Thread

Java中的子线程，可以通过传入Runnable接口或继承Thread重写run方法新建

## AsyncTask

由Java线程池改造的异步任务工具

## IntentService

Android四大组件之一的Service默认是在主线程中运行的，IntentService是Service的子类，有如下特点：
1. IntentService会创建单独的worker线程来处理所有的Intent请求
2. 当所有请求处理完成后，IntentService会自动停止
3. 为Service的onBind方法提供了默认实现，返回null
4. 为Service的onStartCommand方法提供了默认实现，把请求Intent添加到队列中

实现IntentService的例子如下：
```java
public class MyIntentService extends IntentService {  
      
    public MyIntentService(String name) {  
        super(name);  
    }  
  
    @Override  
    protected void onHandleIntent(Intent intent) {  
        // IntentService会创建单独的worker线程来处理此处的代码  
  
    }  
  
} 
```

我们只需重写onHandleIntent方法即可，该方法的代码会在一个子线程中运行

# 五种Layout各自特点

## FrameLayout

FrameLayout即单帧布局，在该布局中的所有空间都会被置于布局的左上角

## LinearLayout

LinearLayout即线性布局，使用该布局必须为其指定orientation属性（排列方向属性，可以设置为水平或垂直的），其中的空间就会根据设置的属性呈水平或垂直排列

## AbsoluteLayout

AbsoluteLayout即绝对布局，对布局中的控件我们使用x，y坐标值进行定位

## RelativeLayout

RelativeLayout即相对布局，在其中的控件可以相对于父布局或布局中别的控件的位置进行定位布局

## TableLayout

TableLayout即表格布局，TableLayout中的每一行的控件由TableRow包含，最终的布局效果会呈现成表格状