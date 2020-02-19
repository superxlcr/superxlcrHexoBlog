---
title: 记一次aapt编译错误追踪
tags: [android,编译]
categories: [android]
date: 2020-02-19 21:09:19
description: 记一次aapt编译错误追踪
---

在最近的一次编译中，博主的gradle编译出错了，并报出了以下错误：
```
AAPT2 error: check logs for details
```

网上找了一些资料，基本没有怎么说到原因，都只是建议加上属性来关闭aapt2：
```
android.enableAapt2=false
```

不过在博主的项目中已经关闭了，因此博主只能尝试追踪一下错误的原因

aapt的编译跟Android的资源文件有关，通过git查看本次资源文件中修改带有点九图，通过经验判断怀疑是点九图出了问题

这里博主找了下网上手动使用aapt编译点九图的资料：
https://blog.csdn.net/waterseason/article/details/84749463

通过手动使用aapt编译一新一旧两张点九图，博主确认了这次的aapt编译错误应该就是点九图的问题导致的：
```
aapt s -i new.9.png -o test
Crunching single PNG file: new.9.png
        Output file: test
ERROR: 9-patch image new.9.png malformed.
       Frame pixels must be either solid or transparent (not intermediate alphas).
       Found at pixel #21 along top edge.
	   
	   
aapt s -i old.9.png -o test
Crunching single PNG file: old.9.png
        Output file: test
```

综上所述，当我们发现gradle报aapt的编译问题的时候，由于gradle没有导出aapt的编译日志，因为我们可以尝试手动使用aapt编译，去查看日志寻找编译错误原因
尤其值得注意的是当我们在项目中改动了点九图的时候