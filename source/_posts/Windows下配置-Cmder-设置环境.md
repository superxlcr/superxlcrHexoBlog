---
title: Windows下配置 Cmder 设置环境
tags: [操作系统,应用]
categories: [操作系统]
date: 2017-12-04 20:55:56
description: win + R 启动 Cmder、右键添加 Cmder here 选项、设置Cmder初始目录、修复ls指令中文乱码的问题
---
Cmder 是一款好用的 Console Emulator，其官网为：
http://cmder.net/

下载完后，我们可以在 Window 下配置我们的 Cmder 了


# win + R 启动 Cmder

我们可以在 Window 环境变量的 PATH 中添加我们Cmder的路径，以后就可以通过 win + R 输入相关名称来启动我们的Cmder了

# 右键添加 Cmder here 选项

我们首先需要通过原来的cmd来到Cmder的目录下，然后运行相关的指令：


```
Cmder.exe /REGISTER ALL
```

运行此命令后，我们右键菜单中就多了 Cmder here 的选项，可以快速在某个文件夹下打开Cmder
ps：如果出现错误，请尝试以管理员身份运行Cmder

# 设置Cmder初始目录

我们可以按下：win + alt + p 来开启 Cmder 的设置菜单，首先我们看到Startup里面的Specified named task选项，该选项说明了你当前使用的是哪个task
接着我们选择Startup下面的Tasks ，修改刚刚看到的对应的选项，加上：


```
-new_console:d:%your_path%
```

把%your_path%改为你需要的初始目录即可
或者我们也可以点击Startup dir...按钮进行GUI操作

# 修复ls指令中文乱码的问题

我们可以按下：win + alt + p 来开启 Cmder 的设置菜单，选择Startup 下面的Environment，添加一项：


```
set LANG=zh_CN.UTF-8
```







