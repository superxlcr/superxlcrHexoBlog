---
title: Android compileSdkVersion 23 导致apache网络库HttpClient过期问题
tags: [android]
categories: [android]
date: 2018-01-03 20:05:33
description: Android compileSdkVersion 23 导致apache网络库HttpClient过期问题
---
博主最近在升级项目compileSdkVersion的时候遇到了一个尴尬的问题：
当compileSdkVersion设置为23后，即就是android6.0版本更新以后，Android就不再提供org.apache.http.x包了，而是改为使用HttpUrlConnection了
然后博主所在的项目比较旧，并不能轻易从apache网络库(即HttpClient)切换至HttpUrlConnection
根据官方文档，解决办法是，只要在gradle的编译文件中添加相应参数就好了：


```
android {
	...
	
	useLibrary 'org.apache.http.legacy'
	
	...
}
```


官方文档可参见下面地址：

https://developer.android.com/about/versions/marshmallow/android-6.0-changes.html




