---
title: Android 图片总结
tags: [android,内存,应用]
categories: [android]
date: 2017-04-27 19:11:32
description: 图片使用场景、内存中的图片、硬盘上的图片、Android中的图片、Android中bitmap的大小
---
最近了解了一些关于Android图片相关的知识，在此写一篇博客进行总结。
# 图片使用场景
图片有两种使用的场景：


- 在硬盘上的存储格式，即JPG、PNG等
- 在内存的展开格式，即Bitmap

# 内存中的图片
因为需要知道图片的所有信息，所有在内存中，图片一般是完全展开的（未压缩的）。

对于支持透明度的BItmap来说，对于图片上的每一个像素点，我们的颜色属性由R（红）、G（绿）、B（蓝）三色组成，而透明度属性由A（alpha值）表示。由于我们的图片在内存中是完全展开的，因此图片Bitmap在内存中占用的大小为：图片宽 X 图片高 X 图片每像素大小（一般而言是4byte，ARGB分别使用8bit进行表示）。
# 硬盘上的图片
对于硬盘上的图片，我们并不会像在内存那样直接存储图片，我们会对图片进行一定的压缩以节省空间。目前来说，比较常见的压缩格式有两种：

- PNG：可移植网络图形格式(Portable Network Graphic Format)，这是一种无损的图片压缩格式，带有透明度值（A），一般而言会比较大。
- JPG：它是一种有损压缩，不带透明值，在不影响图片使用的情况下会丢弃一部分的信息，一般来说肉眼区分不出来。

一般而言，对于同一张bitmap保存为JPG会比保存为PNG小得多，因此一般网络上使用JPG格式的图片较多。


# Android中的图片
在Android系统中，图片文件以Bitmap对象的形式存在，一般而言我们都会使用BitmapFactory工厂类来创建我们的Bitmap对象。
在使用BitmapFactory的静态方法创建Bitmap时，我们都会涉及一个BitmapFactory.Options参数类用来控制解析图片文件的过程，其中的比较重要的参数有：

- inDensity：bitmap使用的像素密度
- inJustDecodeBounds：如果设置为true，BitmapFactory解析不会返回Bitmap，不会占用内存，但可以从Options获取图片的大小类型等信息，用于加载大图片时获取尺寸用
- inPreferredConfig：设置图片的解析类型，设置图片每个像素用多少字节表示，一般而言默认解析类型为ARGB_8888
- inSampleSize：设置图片的缩小倍数，若设置为4，则会返回长宽各为原图1/4，图片大小为1/16的bitmap
- outHeight、outWidth：用于设置了inJustDecodeBounds时获取图片大小尺寸
- outMimeType：用于获取图片的MIME类型

图片的解析类型，在内存中的展开类型有以下几种：

- ALPHA_8：每像素只存储alpha通道（透明度），因此每像素只占用1字节
- ARGB_4444：每像素存储alpha，red，green，blue四通道，每种通道占用4bit，因此每像素占用2字节，由于显示图片效果不佳已弃用
- ARGB_8888：与ARGB_4444相似，但每种通道占用1字节，每像素占用4字节，为默认解析类型
- RGB_565：三通道，R通道占用5bit，G通道占用6bit，B通道占用5bit，每像素占用2字节



# Android中bitmap的大小
在Android中解析的Bitmap对象，我们可以通过调用getAllocationByteCount方法来获取该图片在内存中占用的大小。


在了解如何计算bitmap内存大小前，我们需要先了解关于Android分辨率的一些知识：[Android 关于dp dip sp px dpi density解析](/2016/03/12/Android-关于dp-dip-sp-px-dpi-density解析)


下面给出例子：

```java
        BitmapFactory.Options options = new BitmapFactory.Options();
        options.inPreferredConfig = Bitmap.Config.RGB_565;
        Bitmap bitmap = BitmapFactory.decodeResource(getResources(), R.mipmap.test, options);
        Log.d("MyLog", bitmap.getWidth() + "");
        Log.d("MyLog", bitmap.getHeight() + "");
        Log.d("MyLog", bitmap.getAllocationByteCount() + "");
```



我们在MX4手机中（屏幕大小为1920 x 1152，属于超超高分辨率，xxhdpi），解析了一张test.jpg图片。
test 图片信息如下，被我们存放在mipmap的mdpi子目录中，大小为203 x 300：
![test_info_pic1](1.png)


打印的Log如下：
![log输出_pic2](2.png)



由log可以看到，bitmap占用的内存为1096200byte，该值的计算过程如下：
首先图片的原始大小为 203 x 300，但该图片是存放在mdpi目录下的，而我们的手机为xxhdpi分辨率，因此图片在解析时会有一定的扩展（mdpi ：xxhdpi = 4 ：12）：
解析出来图片的宽高为609 x 900（203 / 4 x 12，300 / 4 x 12）。
再者，由于图片本身为JPG类型，没有alpha通道，因此我们在解析图片的类型时也选择了没有alpha通道的RGB_565类型，每像素占用的内存大小为2字节：
因此，综上所述图片占用的内存为：609 x 900 x 2 = 1096200（byte）
