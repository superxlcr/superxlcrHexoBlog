---
title: Android常见问题总结（九）
tags: [android,基础知识]
categories: [android]
date: 2019-08-07 18:50:20
description: addFooterView与setAdapter顺序问题、查看应用是否已安装、gradle 插件版本号与gradle 版本号对应关系、处理安装出现INSTALL_FAILED_TEST_ONLY错误、Activity启动流程图总结
---

上一篇博客传送门：[Android常见问题总结（八）](/2019/05/06/Android常见问题总结（八）/)

# addFooterView与setAdapter顺序问题

最近博主遇到了一个 cannot be cast to android.widget.HeaderViewListAdapter 的崩溃，在网上查阅了相关资料，发现是ListView的addFooterView与setAdapter调用顺序不当导致的（addHeaderView同理）

结论：在Android 4.4之前，如果在调用setAdapter之后调用addFooterView，则会在调用removeFooterView的时候抛出异常

原因：

先贴上Android 4.2的listView源码：
http://androidxref.com/4.2_r1/xref/frameworks/base/core/java/android/widget/ListView.java
再贴上Android 4.4的listView源码：
http://androidxref.com/4.4.4_r1/xref/frameworks/base/core/java/android/widget/ListView.java

我们先看下抛出异常的方法removeFooterView：
```java
/**
* Removes a previously-added footer view.
*
* @param v The view to remove
* @return
* true if the view was removed, false if the view was not a footer view
*/
public boolean removeFooterView(View v) {
	if (mFooterViewInfos.size() > 0) {
		boolean result = false;
		if (mAdapter != null && ((HeaderViewListAdapter) mAdapter).removeFooter(v)) {
			if (mDataSetObserver != null) {
				mDataSetObserver.onChanged();
			}
			result = true;
		}
		removeFixedViewInfo(v, mFooterViewInfos);
		return result;
	}
	return false;
}
```

如果我们addFooterView与setAdapter调用顺序不当，当我们调用removeFooterView时，在上面的源码中的第11行，会由于mAdapter强制转换为HeaderViewListAdapter类型而抛出异常

接下来我们来看下setAdapter方法：
```java
/**
 * Sets the data behind this ListView.
 *
 * The adapter passed to this method may be wrapped by a {@link WrapperListAdapter},
 * depending on the ListView features currently in use. For instance, adding
 * headers and/or footers will cause the adapter to be wrapped.
 *
 * @param adapter The ListAdapter which is responsible for maintaining the
 *        data backing this list and for producing a view to represent an
 *        item in that data set.
 *
 * @see #getAdapter()
 */
@Override
public void setAdapter(ListAdapter adapter) {
	...

	if (mHeaderViewInfos.size() > 0|| mFooterViewInfos.size() > 0) {
		mAdapter = new HeaderViewListAdapter(mHeaderViewInfos, mFooterViewInfos, adapter);
	} else {
		mAdapter = adapter;
	}

	...
}
```

在setAdapter方法第19行我们可以看到，如果我们在setAdapter之前添加了headerView或者是footerView的话，则会把我们的adapter包装为HeaderViewListAdapter，因此在这种情况下当我们removeFooterView时，强制转换并不会抛出异常

接下来我们看一下4.2版本的addFooterView方法：
```java
public void addFooterView(View v, Object data, boolean isSelectable) {

	// NOTE: do not enforce the adapter being null here, since unlike in
	// addHeaderView, it was never enforced here, and so existing apps are
	// relying on being able to add a footer and then calling setAdapter to
	// force creation of the HeaderViewListAdapter wrapper

	FixedViewInfo info = new FixedViewInfo();
	info.view = v;
	info.data = data;
	info.isSelectable = isSelectable;
	mFooterViewInfos.add(info);

	// in the case of re-adding a footer view, or adding one later on,
	// we need to notify the observer
	if (mAdapter != null && mDataSetObserver != null) {
		mDataSetObserver.onChanged();
	}
}
```

然后再对比下4.4版本的addFooterView方法：
```java
/**
 * Add a fixed view to appear at the bottom of the list. If addFooterView is
 * called more than once, the views will appear in the order they were
 * added. Views added using this call can take focus if they want.
 * <p>
 * Note: When first introduced, this method could only be called before
 * setting the adapter with {@link #setAdapter(ListAdapter)}. Starting with
 * {@link android.os.Build.VERSION_CODES#KITKAT}, this method may be
 * called at any time. If the ListView's adapter does not extend
 * {@link HeaderViewListAdapter}, it will be wrapped with a supporting
 * instance of {@link WrapperListAdapter}.
 *
 * @param v The view to add.
 * @param data Data to associate with this view
 * @param isSelectable true if the footer view can be selected
 */
public void addFooterView(View v, Object data, boolean isSelectable) {
	final FixedViewInfo info = new FixedViewInfo();
	info.view = v;
	info.data = data;
	info.isSelectable = isSelectable;
	mFooterViewInfos.add(info);

	// Wrap the adapter if it wasn't already wrapped.
	if (mAdapter != null) {
		if (!(mAdapter instanceof HeaderViewListAdapter)) {
			mAdapter = new HeaderViewListAdapter(mHeaderViewInfos, mFooterViewInfos, mAdapter);
		}

		// In the case of re-adding a footer view, or adding one later on,
		// we need to notify the observer.
		if (mDataSetObserver != null) {
			mDataSetObserver.onChanged();
		}
	}
}
```

我们可以看到，为什么4.4及其以上版本就允许addFooterView在setAdapter以后调用，关键的地方在于第27行，发现如果此时mAdapter不为HeaderViewListAdapter，则会将其包装为HeaderViewListAdapter
因此，在4.4及其以上版本，我们并不需要关心setAdapter与addFooterView的调用顺序（headerView同理）

# 查看应用是否已安装

最近博主遇到一个需求，需要查看应用是否已安装
经过一番查找后，找到了如下方案：
```java
    public static boolean isInstallApp(String appPackage) {
        try {
            PackageManager manager = KGCommonApplication.getContext().getPackageManager();
            List<PackageInfo> pkgList = manager.getInstalledPackages(0);
            for (int i = 0; i < pkgList.size(); i++) {
                PackageInfo pI = pkgList.get(i);
                if (pI.packageName.equalsIgnoreCase(appPackage))
                    return true;
            }
        } catch (NullPointerException e) {
            e.printStackTrace();
        }

        return false;
    }
```

原理是使用 android.content.pm.PackageManager#getInstalledPackages 方法，获取所有已安装的apk，然后从返回的列表中，查找是否包含特定包名来判断特定应用是否安装
然而，该方案在高版本的Android中，也许是因为系统权限的限制，我们并不能拿到所有的已安装应用
在没有权限的情况下，该方法只返回了部分系统的应用

因此再通过一番查到，博主找到了一个新的方案：
```java
    public static boolean isInstallApp(String appPackage) {
        try {
            PackageManager manager = KGCommonApplication.getContext().getPackageManager();
            PackageInfo info = manager.getPackageInfo(appPackage, PackageManager.GET_ACTIVITIES);
            return info != null;
        } catch (Exception e) {
            // ignore
        }
        return false;
    }
```

通过 android.content.pm.PackageManager#getPackageInfo(java.lang.String, int) 方法，我们可以获取特定的应用信息，如果用户并没有安装该应用，则会抛出 NameNotFoundException 异常
由于该方法没有系统权限的限制，所以我们可以放心使用

# gradle 插件版本号与gradle 版本号对应关系

gradle 插件版本号：指build.gradle文件里，classpath ‘com.android.tools.build:gradle:3.1.2’，应用的插件的版本号
gradle 版本号：指“gradle-wrapper.properties”中，配置的gradle 的版本号

其中，gradle 插件的版本号和 gradle 的版本号是有关联的，关系如下：

| 插件版本号 | gradle版本号 |
| - | - |
| 1.0.0 - 1.1.3 | 2.2.1 - 2.3 |
| 1.2.0 - 1.3.1 | 2.2.1 - 2.9 |
| 1.5.0 | 2.2.1 - 2.13 |
| 2.0.0 -2.1.2 | 2.10 - 2.13 |
| 2.1.3 - 2.2.3 | 2.14.1+ |
| 2.3.0+ | 3.3+ |
| 3.0.0+ | 4.1+ |
| 3.1.0+ | 4.4+ |
| 3.2.0 - 3.2.1 | 4.6+ |
| 3.3.0 - 3.3.2 | 4.10.1+ |
| 3.4.0 - 3.4.1 | 5.1.1+ |
| 3.5.0+ | 5.4.1 - 5.6.4 |

详细gradle插件版本与gradle版本更新日志如下：

https://developer.android.google.cn/studio/releases/gradle-plugin#updating-plugin

# 处理安装出现INSTALL_FAILED_TEST_ONLY错误

某次在本地debug模式编译打包apk后，通过adb install指令发现竟然无法正常安装，提示了INSTALL_FAILED_TEST_ONLY错误
在网上查询资料后发现，原来是Android Studio 3.0会在debug类型apk的manifest文件application标签里自动添加 android:testOnly="true"属性

官方文档如下：
https://developer.android.google.cn/guide/topics/manifest/application-element#testOnly

解决方案有两个：

我们可以在项目中的gradle.properties全局配置中设置：
```
android.injected.testOnly=false
```

除此以外，我们还可以安装apk时，增加上-t的参数：
```
adb install -t app-debug.apk
```

# Activity启动流程图总结

Activity启动流程图如下所示：
![Activity启动流程图](1.jpg)

流程主要分为以下几步：
1. 当前Activity把需要启动的目标Activity相关信息包装成Intent的形式，发送给ActivityManagerService(AMS)
2. AMS检验信息合法后，保存信息，并通知当前Activity进入中止状态（onPaused），并通过Handler监控是否超时
3. 当前Activity进入中止状态后，通知AMS
4. AMS检查目标Activity在Manifest中设定启动的进程启动了没有，如果进程没有启动，则启动进程（也通过Handler计算超时），有则跳到第7步
5. AMS通知zygote进程fork一个子进程，以ActivityThread的main函数作为进程执行入口
6. 新进程初始化Handler消息循环机制，初始化Application，完成启动工作后，发送ApplicationThread给AMS（以后AMS就通过ApplicationThread找到这个进程）
7. AMS通过进程启动后传回的ApplicationThread找到进程，发送命令启动目标Activity，同时开始Handler计算超时
8. 目标进程接到IPC指令后，通过消息循环机制，在主线程中反射实例化Activity，并执行Activity的生命周期，完成后通知AMS
9. AMS收到Activity启动的消息后，Activity启动流程结束