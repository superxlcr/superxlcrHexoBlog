---
title: Apk反编译
tags: [android,编译]
categories: [android]
date: 2016-09-08 12:29:55
description: Apk反编译
---
作为一名Android开发人员，由于debug或者是某些别的需要，我们经常要去反编译一些apk，在这里讲述一下如何反编译一个apk文件。
首先我们需要下载三个反编译工具：

- apktool（用于反编译apk资源文件）
- dex2jar（用于把dex文件转为jar文件）
- jd-gui（jar反编译工具）

这里以一个decompile.apk的反编译过程为例：


这里有一个名为decompile的apk，我们想看看里面究竟有什么东西
首先我们需要下载apktool.jar。值得注意的是，要使用jar，我们的Java环境需要事先配置好
使用java -jar apktool.jar使用apktool工具，使用命令d表示反编译，-f 参数为反编译的apk文件，-o 参数为文件输出地址：
![apktool_pic1](1.png)


然后在decompile文件夹中可以看到，所有的资源文件一目了然：
![资源文件_pic2](2.png)



接下来我们来反编译apk的代码：
把apk文件后缀改为zip：
![更改后缀_pic3](3.png)



利用解压工具解压apk，从解压出的文件中找到class.dex：

![找到class_pic4](4.png)



利用dex2jar把它转化为jar文件：
![转为jar_pic5](5.png)



然后在使用jd-gui打开得到的jar文件：
![jd-gui_pic6](6.png)



大功告成！这里博主反编译的apk只是随便生成的，并没有经过代码混淆，可读性还是非常强的。然而一般发布的apk都会有代码混淆的流程，变量都会被一些毫无意义的字符代替，反编译出来后能看懂多少就看各位的造化了~


值得注意的是直接解压出来的xml文件会是一团乱码，大伙也可以不使用apktool反编译，而是使用AXMLprinter2.jar来反编译xml文件：
直接解压后：
![xml before_pic7](7.png)



使用AXMLprinter2.jar：
```
java -jar AXMLPrinter2.jar AndroidManifest.xml > AndroidManifest.txt

```

得到的txt文件：
![xml after_pic8](8.png)

补充一下，我们也可以直接使用github上的jadx开源工具，直接反编译整个apk：
https://github.com/skylot/jadx