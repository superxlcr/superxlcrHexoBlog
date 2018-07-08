---
title: Android APK 安装包
tags: [android,应用]
categories: [android]
date: 2017-05-02 11:50:17
description: APK总览、APK内容、APK编译过程、APK安装过程
---
最近本人了解了一些关于Android APK安装包的知识，在此写下一篇博客进行总结。
# APK总览
APK是AndroidPackage的缩写，即Android安装包(apk)。APK文件其实是zip格式，但后缀名被修改为apk，我们可以通过更改后缀名的方式来解压缩apk文件，解压后的内容如下图所示：

![apk文件目录_pic1](1.png)


# APK内容
apk文件解压后包含以下内容：


（1）res目录：用于存放Android资源文件的目录，其中的文件均经过编译，有多个子目录用于区分具体的资源文件类型：

- anim：补间动画和逐帧动画文件
- animator：属性动画文件
- color：颜色文件
- drawable：图片文件
- layout：布局文件
- menu：菜单文件
- mipmap：理论上仅用于存放应用icon图标，用于主屏幕图片展示，其余图片文件应存放在drawable中
- raw：原生视频、音频资源
- transition：转场动画效果
- xml：原生xml文件

值得注意的是，每个子目录都可以拥有标识相应分辨率与机型大小作为后缀，如“xxhdpi”，的子目录。



（2）assets目录：用于存放需要打包到APK中的静态文件，文件没有进行编译，访问时需要使用AssetManager类


（3）META-INF目录：即Metadata infomation元数据（又称中介数据、中继数据，用于描述数据的数据）信息目录，该目录下主要包含以下三个文件：

- MANIFEST.MF：摘要文件，列出了apk的所有文件，以及这些文件内容所对应的base64-encoded SHA1 哈希值，用于验证apk文件的完整性，判断apk是否被篡改
- CERT.SF：摘要签名文件，它列出了MANIFEST.MF这个文件中每条信息的hash值，用于验证摘要文件是否被篡改
- CERT.RSA：该文件保存了解密用的公钥，以及加密算法等信息

（4）AndroidManifest.xml：Android的应用配置文件，描述了应用的总体信息，需要使用的用户权限以及组件等等，已被编译



（5）classes.dex：能用于dalvik执行的字节码，由多个.class的Java字节码文件组合而成


（6）resources.arsc：资源配置文件，用于记录资源文件与资源id之间的映射关系，res/values中的信息大部分被编译在此处


（7）lib目录：Android依赖的Native库的目录，其中会根据不同的cpu架构分为多个子目录


# APK编译过程
APK的大致编译过程如下图所示：
![Android apk编译过程_pic2](2.png)




1. 编译器将源代码转换成 DEX（Dalvik Executable) 文件（其中包括运行在 Android 设备上的字节码），将所有其他内容通过aapt（Android Asset Packaging Tool） 转换成已编译资源，编译成二进制文件
2. APK 打包器将 DEX 文件和已编译资源合并成单个 APK
3. APK 打包器使用调试或发布密钥库签署您的 APK
4. 在生成最终 APK 之前，打包器会使用 zipalign 工具对应用进行优化，减少其在设备上运行时的内存占用

官方链接：https://developer.android.google.cn/studio/build/index.html



# APK安装过程
Android的应用apk安装涉及如下几个目录：

- /data/app：存放用户安装的apk目录，安装时会把apk复制到这里
- /data/data：应用安装完成后，会在该目录下生成与apk包名（packagename）一样的文件夹，用于存放应用数据
- /data/dalvik-cache：存放apk的odex文件，便于应用启动时直接执行

具体的安装过程如下：

首先，系统会把apk复制到/data/app下，然后通过META-INF下的文件校验apk的签名是否正确，检查apk的结构是否正常，进而解压并校验dex文件。如果dex文件没有被破坏，则会把dex文件优化为odex文件，使得程序的启动时间加快，同时，在/data/data目录下建立与apk包名相同的文件夹。如果apk中有Native库，lib目录的话，系统会判断so库的名字是否正确，并根据系统cpu架构解压对应的so库到/data/data/packagename/lib下。


附，odex的内容如下所示：
![odex_pic3](3.jpg)

odex文件在原来的dex文件头添加了一些数据，在文件尾部添加了程序运行时需要的依赖库和辅助数据，使得程序运行速度加快。
