---
title: VMware下扩展Ubuntu根分区大小的方法
tags: [linux,操作系统]
categories: [linux]
date: 2017-12-17 12:59:54
description: VMware下扩展Ubuntu根分区大小的方法
---
刚开始设置VMware下的Ubuntu虚拟机的硬盘时，由于担心占用过多的空间，因此把容量设置得不够大，以至于后面发现容量不够用时需要对Ubutun的根分区进行一定的拓展，下面总结一下拓展的方法：
首先我们需要在VMware的虚拟机设置中选择拓展硬盘大小：
![_pic1](1.png)



修改完硬盘大小后，我们还需要使用工具对Ubutun系统进行分区，在此推荐一款叫gparted的软件：
http://nchc.dl.sourceforge.net/project/gparted/gparted-live-stable/0.8.0-5/gparted-live-0.8.0-5.iso

下载完这个软件后，我们可以在虚拟机设置的CD设置里面加载该iso文件
然后我们在开启虚拟机的时候按下ESC键，选择BIOS从光盘启动：
![_pic2](2.png)

之后选择第一项，然后一路回车即可：
![_pic3](3.png)

进入到GParted后，我们可以看到以下界面，因为我们需要拓展根分区大小，因此我们需要先把linux-swap的交换区删掉，把根分区拓展之后再重新建立起来：
![_pic4](4.png)

拓展完重启后，我们可以使用fdisk -l指令看到根分区变大了
