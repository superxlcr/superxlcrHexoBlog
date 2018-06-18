---
title: Intellij / Android Studio 调试 Gradle Plugin
tags: [gradle]
categories: [gradle]
date: 2018-06-18 17:44:55
description: （转载）Intellij / Android Studio 调试 Gradle Plugin
---
本文转载自：https://blog.csdn.net/ceabie/article/details/55271161

网上搜了很久，没发现一篇靠谱的，很多都是版本比较老的Intellij和gradle版本，和现在的都不合适。
这里的教程是指 Intellij 2017，以及Android Studio 2.2以上，gradle 2.14.1以后的版本。

当我们需要调试gradle插件时，我们需要执行以下几步：

# 创建remote调试任务

选择 Edit Configurations
![Edit Configurations示意图](1.png)

点左上角的 + 号，选择 remote。Name可以随意命名，其他配置可以不用动，端口就5005，点ok关闭
![任务参数图](2.png)

# 使用Terminal窗口（一般在底下的工具栏上）执行任务

执行相应的gradle task，并输入debug参数（下面以assembleDebug task 为例）：
```
gradlew assembleDebug -Dorg.gradle.daemon=false -Dorg.gradle.debug=true
```

注意，*.gradle脚本是无法调试的
这里最重要的是被调试的构建过程不使用当前的IDE直接运行，最简单就是使用Terminal
在Terminal的命令中点回车后，会出现 To honour the JVM settings for this build a new JVM will be forked. 这行提示，并且会一直停在这里，说明在等待调试
![Terminal调试示意图](3.png)

# 开始调试

这时候选择第二步中创建的remote任务，并使用调试启动（下图最右边的调试按钮），而不是make或直接运行：
![调试示意图](4.png)