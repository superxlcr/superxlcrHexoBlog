---
title: Android源码下载repo以及repo init总结
tags: [android,编译,源码]
categories: [android]
date: 2017-12-31 16:07:19
description: Android源码下载repo以及repo init总结
---
最近下载了一波Android的源码，由于google源码被墙，以及编译环境等一系列问题，搞得头皮发麻，在此写下一篇博客记录一下由于看的书是《Android系统源代码情景分析》，我们下载的目标源码是Android 2.3.4，因此我们需要使用的是Ubutun 12.04版本的系统，否则编译源码时会出现一堆奇怪的问题首先，我们需要安装java1.6，然后再安装git和一系列的编译一堆工具：
```
sudo apt-get install git-core gnupg
```

```
sudo apt-get install flex bison gperf libsdl-dev libesd0-dev libwxgtk3.0-dev build-essential zip curl valgrind
```

然后下载google提供的下载源码的repo工具，在合适的目录下执行命令下载repo，并把它改为可执行文件即可：
```
curl "http://php.webtutor.pl/en/wp-content/uploads/2011/09/repo" > repo
chmod a+x repo
```

然后由于google背墙的原因，我们需要修改以下参数：把repo中的 REPO_URL 改为 REPO_URL='http://code.google.com/p/git-repo/'然后再repo的目录下执行init指令即可：
```java
repo init -u git://Android.git.linaro.org/platform/manifest.git -b android-2.3.4_r1
```

初始化完成后，我们再修改下init目录下的隐藏文件.repo中的manifest.xml文件：把fetch="git://Android.git.kernel.org/"改为fetch="git://Android.git.linaro.org/"

然后执行 repo sync 指令开始下载Android源代码由于下载源码的指令经常会由于网络原因莫名奇妙中断掉，在此分享一个中断后会自动重新执行repo sync的sh脚本：
```
#!/bin/sh
repo sync
while [ $? -ne 0 ]
do
repo sync
done
```

我们保存改脚本并改为可执行文件后，执行sh xxx.sh开始同步源码即可，在源码下载完成后脚本会自行终止
在下载完Android源码后，我们还可以下载Android的内核源码，由于内核源码也是被墙了，原地址根本下不动，因此我们需要通过国内镜像来下载：

| 名称 | Google GIT地址 | 清华服务器地址 | 
| - | - | - |
| common | https://android.googlesource.com/kernel/common.git | https://aosp.tuna.tsinghua.edu.cn/kernel/common.git | 
| exynos | https://android.googlesource.com/kernel/exynos.git | https://aosp.tuna.tsinghua.edu.cn/kernel/exynos.git | 
| goldfish | https://android.googlesource.com/kernel/goldfish.git | https://aosp.tuna.tsinghua.edu.cn/kernel/goldfish.git | 
| hikey-linaro | https://android.googlesource.com/kernel/hikey-linaro | https://aosp.tuna.tsinghua.edu.cn/kernel/hikey-linaro.git | 
| lk |   | https://aosp.tuna.tsinghua.edu.cn/kernel/lk.git | 
| omap | https://android.googlesource.com/kernel/omap.git | https://aosp.tuna.tsinghua.edu.cn/kernel/omap.git | 
| samsung | https://android.googlesource.com/kernel/samsung.git | https://aosp.tuna.tsinghua.edu.cn/kernel/samsung.git | 
| tegra | https://android.googlesource.com/kernel/tegra.git | https://aosp.tuna.tsinghua.edu.cn/kernel/tegra.git | 
| x86_64 | https://android.googlesource.com/kernel/x86_64.git | https://aosp.tuna.tsinghua.edu.cn/kernel/x86_64.git | 
| msm | https://android.googlesource.com/kernel/msm.git | https://aosp.tuna.tsinghua.edu.cn/kernel/msm.git | 

 