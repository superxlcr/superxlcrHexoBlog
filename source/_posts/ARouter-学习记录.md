---
title: ARouter 学习记录
tags: [android,应用]
categories: [android]
date: 2017-09-13 14:53:35
description: 框架配置与初始化、Activity基础跳转、传递参数与参数注入、使用新旧转场动画处理跳转、kotlin跳转、接口调用、获取Fragment、跨模块调用、处理跳转结果、拦截器
---
ARouter 是阿里巴巴开源的一个Android页面路由框架，它提供了url跳转、自动解析bundle数据赋值，自定义拦截跳转过程，url调用接口服务等功能
下面我们来了解下ARouter 框架的使用

# 框架配置与初始化

gradle 版本 &gt;= 2.2 的配置方法（使用annotationProcessor）：

```java
android {
    defaultConfig {
	...
	javaCompileOptions {
	    annotationProcessorOptions {
		arguments = [ moduleName : project.getName() ]
	    }
	}
    }
}

dependencies {
    // 替换成最新版本, 需要注意的是api
    // 要与compiler匹配使用，均使用最新版可以保证兼容
    compile 'com.alibaba:arouter-api:x.x.x'
    annotationProcessor 'com.alibaba:arouter-compiler:x.x.x'
    ...
}
```


gradle 版本 &lt; 2.2 的配置方法（使用apt）：


```java
apply plugin: 'com.neenbedankt.android-apt'

buildscript {
    repositories {
	jcenter()
    }

    dependencies {
	classpath 'com.neenbedankt.gradle.plugins:android-apt:1.4'
    }
}

apt {
    arguments {
	moduleName project.getName();
    }
}

dependencies {
    compile 'com.alibaba:arouter-api:x.x.x'
    apt 'com.alibaba:arouter-compiler:x.x.x'
    ...
}
```



在Application中初始化ARouter：

```java
public class MyApplication extends Application {

    @Override
    public void onCreate() {
        super.onCreate();
        if (BuildConfig.DEBUG) {           // 这两行必须写在init之前，否则这些配置在init过程中将无效
            ARouter.openLog();     // 打印日志
            ARouter.openDebug();   // 开启调试模式(如果在InstantRun模式下运行，必须开启调试模式！线上版本需要关闭,否则有安全风险)
        }
        ARouter.init(this); // 尽可能早，推荐在Application中初始化
    }
}
```





# Activity基础跳转


在Activity声明注解Route，并赋予path值（即url），值得注意的是path必须至少有两级目录，否则会在编译期报错：

```java
@Route(path = "/arouterTest/activities/jumpTestActivity")
public class JumpTestActivity extends AppCompatActivity {
```


然后调用ARouter的build方法输入url，调用navigation方法即可打开对应的Activity：

```java
ARouter.getInstance().build("/arouterTest/activities/jumpTestActivity").navigation();
```



也可以传入uri进行跳转：

```java
        Uri testUri = Uri.parse("test://test.com/arouterTest/activities/jumpTestActivity");
        ARouter.getInstance().build(testUri)
                .navigation();
```


需要接收ActivityResult的可以在navigation中传入requestCode：

```java
ARouter.getInstance().build("/arouterTest/activities/jumpTestActivity").navigation(this, 666);
```



# 传递参数与参数注入

传递参数调用ARouter的with系列方法即可：

- with各种基本类型
- withObject
- withParcelable
- with：直接传递Bundle参数

例，传入名为text的字符串参数：


```java
ARouter.getInstance().build("/arouterTest/activities/jumpTestActivity")
                .withString("text", "老子吃火锅，你吃火锅底料！")
                .navigation();
```


参数注入，即在目标Activity中自动解析传入的参数：


首先我们需要使用Autowired注解，并且把注解标记的属性设置为public的：

```java
@Autowired(name = "text")
public String text;
```


然后调用ARoute的inject方法即可：

```java
ARouter.getInstance().inject(this);
```


# 使用新旧转场动画处理跳转


旧动画：
使用withTransition方法即可：

```java
ARouter.getInstance().build(testUri).withTransition(R.anim.slide_in, R.anim.slide_out)
                .navigation();
```


新动画：
使用withOptionsCompat方法即可：

```java
ActivityOptionsCompat compat = ActivityOptionsCompat.
                makeScaleUpAnimation(view, view.getWidth() / 2, view.getHeight() / 2, 0, 0);

        ARouter.getInstance().build("/module/jumpTestActivity2").withOptionsCompat(compat)
                .navigation();
```


# kotlin跳转


与Java相似，声明Route注解并为path属性赋值：

```
@Route(path = "/kotlin/test")
class KotlinTestActivity : Activity() {
```


然后跳转即可：

```java
ARouter.getInstance()
                        .build("/kotlin/test")
                        .navigation();
```



# 接口调用


除了跳转Activity外，ARouter还允许用户进行接口调用
首先我们需要新建接口，我们的接口需要继承IProvider接口：

```java
public interface MyServiceInterface extends IProvider {

    void doSomething();
}
```


接着实现这个接口：

```java
@Route(path = "/interface/my-service")
public class MyServiceImpl implements MyServiceInterface {

    @Override
    public void doSomething() {
        Log.d("MyServiceImpl", "I just do something!");
    }

    @Override
    public void init(Context context) {
        Log.d("MyServiceImpl", "init!");
    }
}
```


我们可以在init方法中进行一些接口首次被调用时的初始化操作。


调用接口的方式有几种：
使用url调用接口：

```java
((MyServiceInterface) ARouter.getInstance().build("/interface/my-service").navigation())
                .doSomething();
```


通过类型（.class）来调用接口：

```java
ARouter.getInstance().navigation(MyServiceImpl.class).doSomething();
```


# 获取Fragment


ARouter还允许我们获取某个Fragment：
使用Route注解声明：

```java
@Route(path = "/test/fragment")
public class BlankFragment extends Fragment {
```


通过url获取：

```java
Fragment fragment = (Fragment) ARouter.getInstance().build("/test/fragment").navigation();
```


# 跨模块调用


ARouter的调用除了能在本module中运行外，也可以跨module运行，由于代码类似，在此不多研究



# 处理跳转结果

ARouter可以让我们处理跳转过程的结果：

```java
        ARouter.getInstance().build("/module/jumpTestActivity2").navigation(null,
                new NavigationCallback() {
                    @Override
                    public void onFound(Postcard postcard) {
                        // 找到目标后进行的操作
                    }

                    @Override
                    public void onLost(Postcard postcard) {
                        // 找不到目标进行的操作
                    }
                });
```


# 拦截器


ARouter的拦截器可以在navigation的过程中拦截请求，并进行一系列的处理，是一种AOP的编程模式（应用场景为检查登录状态等）
要实现拦截器，首先我们需要实现IInterceptor接口，并使用Interceptor注解标记我们的拦截器，并传入priority优先级参数（数字越小、优先级越高）
例子：这里实现了两个拦截器：

```java
@Interceptor(priority = 6)
public class MyInterceptorFirst implements IInterceptor {

    @Override
    public void process(Postcard postcard, InterceptorCallback callback) {
        Log.i("MyInterceptorFirst", postcard.toString());
        callback.onContinue(postcard);
    }

    @Override
    public void init(Context context) {
        Log.i("MyInterceptorFirst", "init");
    }
}

@Interceptor(priority = 7)
public class MyInterceptorSecond implements IInterceptor {

    @Override
    public void process(Postcard postcard, InterceptorCallback callback) {
        Log.i("MyInterceptorSecond", postcard.toString());
    }

    @Override
    public void init(Context context) {
        Log.i("MyInterceptorSecond", "init");
    }
}
```


IInterceptor接口定义了两个方法：

- init方法：在ARouter初始化的时候会调用，用于进行一系列初始化操作
- process方法：拦截请求的处理方法，分别传入Postcard（请求的具体信息），InterceptorCallback（用于控制拦截流程）两个参数

这里说一下InterceptorCallback这个参数，这个接口定义了两个方法：


- onContinue：传入Postcard参数（可以更改请求参数），表示当前拦截器放行此请求（该请求可以被更低优先级的拦截器拦截，或者没有拦截器了就执行对应操作）
- onInterrupt：拦截此次请求，不再传递下去
- 如果不调用任何方法则默认拦截此次请求

因此，上面例子中的拦截器实现的效果是，分别执行MyInterceptorFirst、MyInterceptorSecond拦截器，并拦截请求，无法跳转



以上就是本人探究的ARouter的功能，如有遗漏欢迎补充~
