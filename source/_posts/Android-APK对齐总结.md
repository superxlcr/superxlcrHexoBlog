---
title: Android APK对齐总结
tags: [android,资源文件]
categories: [android]
date: 2017-04-24 22:23:10
description: 什么是字节对齐、什么是Zipalign、如何使用Zipalign
---
本人最近了解了一些关于Android APK对齐的知识，在此写篇博客总结一下：
# 什么是字节对齐
所谓的字节对齐，就是各种类型的数据按照一定的规则在空间上排列，而不是顺序的一个接一个的排放，这个就是对齐。我们经常听说的对齐在N上（N字节对齐），它的含义就是：数据的存放起始地址%min（N，数据字节大小）== 0。

需要字节对齐的根本原因在于CPU访问数据的效率问题，数据字节对齐后可以减少CPU访问内存的次数，但相应的，字节对齐也增加了内存空间的消耗（存在某些空内存没被使用）。



# 什么是Zipalign
首先给出官方链接：https://developer.android.google.cn/studio/command-line/zipalign.html
zipalign是Android SDK中的一个用于优化APK的新工具，它提高了优化后的Applications与Android系统的交互效率，从而可以使整个系统的运行速度有了较大的提升。
根据官方文档的描述，Android系统中应用的数据都保存在它的APK文件中，这些文件经常会被多个进程访问：


- 安装程序通过每个apk的manifest文件获取与当前应用程序相关联的permissions信息
- Home程序读取当前APK的名称和图标等信息
- System server读取一些与应用运行相关信息
- APK所包含的内容不仅限于当前应用所使用，而且可以被其它的应用通过内容提供器调用

zipalign优化的最根本目的是帮助操作系统更高效率的根据请求索引资源，通过将apk中的未压缩数据进行字节对齐（一般为4字节对齐），允许系统使用mmap方法直接映射文件至内存空间，降低内存的消耗。


# 如何使用Zipalign
一般来说，Android Studio会自动帮你进行zipalign相关的优化。

手动进行优化时，zipalign所在的位置为：sdk目录/build-tools/对应版本号，不同的Android版本对应着不同的zipalign工具。


对齐一个apk文件的方法如下：
对齐infile.apk并输出为outfile.apk

```
zipalign [-f] [-v] <alignment> infile.apk outfile.apk
```


验证一个apk文件是否对齐的方法如下：

```
zipalign -c -v <alignment> existing.apk
```


&lt;alignment&gt;指的是字节对齐参数，一般来说这个参数的值均为4，否则它起不到任何作用。

其他参数：

- -f：输出覆盖已存在的outfile.zip文件
- -v：输出详细的日志
- -p：outfile.zip should use the same page alignment for all shared object files within infile.zip（这句并非相当理解……）
- -c：确认apk是否对齐



