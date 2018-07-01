---
title: Android常见问题总结（五）
tags: [android,基础知识]
categories: [android]
date: 2016-05-17 12:36:13
description: Android内存优化方法、Android中弱引用与软引用的应用场景、View与View Group分类，自定义View过程、Touch事件分发机制
---
上一篇文章的传送门：[Android常见问题总结（四）](/2016/05/09/Android常见问题总结（四）/)

# Android内存优化方法

首先有一些与内存泄漏相关的点：[Android防止内存泄露小结](/2016/04/13/Android防止内存泄露小结/)，在此不再赘述了。

还有一些内存优化的方法，个人认为有以下几个方面：
- Java引用的灵活使用
- 图片缓存
- Bitmap压缩加载

## Java引用的灵活使用

众所周知，在Java中有强软弱虚四种引用，合理的使用软引用和弱引用有利于GC工作的进行，回收不需要的内存。

## 图片缓存

使用图片缓存可以有效防止内存中同时载入过多的图片，也可以减少经常载入图片所耗费的时间。
一般而言，设计图片缓存的思路如下，可以把图片的缓存分为以下几个层级：
- 强引用 HashMap （用于存储最常用的图片）
- 软引用 HashMap （用于存储比较常用的图片）
- SD卡 （用于存储使用过的图片）
- 下载 （第一次使用）

一般而言我们都是用LRU算法（最近最少使用），最近使用的图片一般缓存于强引用的HashMap中，随着使用次数的减少慢慢移动至软引用HashMap、SD卡中。
除了我们自己设计图片缓存外，我们还可以直接使用Google提供的LruCache（内部由LinkedHashMap实现缓存队列），以及DiskLruCache（用于实现硬盘缓存的，需要自行下载，[代码地址](https://android.googlesource.com/platform/libcore/+/jb-mr2-release/luni/src/main/java/libcore/io/DiskLruCache.java)）。

## Bitmap压缩加载

对于某些原尺寸特别大的图片，我们可以选择压缩后再加载以节省我们的内存空间：
```java
// 创建解析参数
Options options = new Options();
// 只解析大小，并不生成实际图片
options.inJustDecodeBounds = true;
BitmapFactory.decodeFile(pathName, options);
// 计算缩放大小
int widthScale = options.outWidth/width_we_need;
int heightScale = options.outHeight/height_we_need;
// 选取小的为缩放尺寸
options.inSampleSize = Math.min(widthScale, heightScale);
// 生成Bitmap
options.inJustDecodeBounds = false; // 记得关掉
Bitmap bitmap = BitmapFactory.decodeFile(pathName, options);
```

压缩到合适的尺寸后再加载就不必担心原来的bitmap太大而占用内存空间了。

# Android中弱引用与软引用的应用场景

软引用与弱引用一般用于：需要某个对象，但又不关心它的死活的时候（就是该对象存在就用，不存在就算了）。
在Android中的应用场景主要有：
- Handler获取Activity的引用：使用静态变量和软弱引用保证不干扰Activity的回收
- 缓存设计：使用软弱应用保证内存紧张时GC可以回收使用较少的缓存资源

# View与View Group分类，自定义View过程

View是Android中基本的UI单元，占据屏幕的一块矩形区域，可用于绘制并能处理事件，而ViewGroup是View的子类，他能包含多个View，并让他们在其中按照一定的规则排列。View与ViewGroup的设计使用了组合模式。

自定义View我们一般需要重写一下三个方法：
- onMeasure：用于测量自定义View的大小，在方法中必须调用setMeasureDimension方法
- onLayout：用于确定自定义View布局
- onDraw：用于绘制自定义View本身

对于measure、layout、draw的更详细流程可参考我的另一篇博客：[一个Activity的显示过程总结（四）](/2016/05/17/一个Activity的显示过程总结（四）/)

# Touch事件分发机制

可参考：[Android View 与 ViewGroup 事件分发总结](/2016/03/06/Android-View-与-ViewGroup-事件分发总结/)