---
title: 理解Maven中的SNAPSHOT版本和正式版本
tags: [应用,编译,gradle]
categories: [杂项]
date: 2020-05-26 20:49:08
description: （转载）理解Maven中的SNAPSHOT版本和正式版本
---
本文转载自：https://www.cnblogs.com/huang0925/p/5169624.html

Maven中建立的依赖管理方式基本已成为Java语言依赖管理的事实标准，Maven的替代者Gradle也基本沿用了Maven的依赖管理机制。在Maven依赖管理中，唯一标识一个依赖项是由该依赖项的三个属性构成的，分别是groupId、artifactId以及version。这三个属性可以唯一确定一个组件（Jar包或者War包）。

其实在Nexus仓库中，一个仓库一般分为public(Release)仓和SNAPSHOT仓，前者存放正式版本，后者存放快照版本。如果在项目配置文件中（无论是build.gradle还是pom.xml）指定的版本号带有’-SNAPSHOT’后缀，比如版本号为’Junit-4.10-SNAPSHOT’，那么打出的包就是一个快照版本。

快照版本和正式版本的主要区别在于，本地获取这些依赖的机制有所不同。假设你依赖一个库的正式版本，构建的时候构建工具会先在本次仓库中查找是否已经有了这个依赖库，如果没有的话才会去远程仓库中去拉取。所以假设你发布了Junit-4.10.jar到了远程仓库，有一个项目依赖了这个库，它第一次构建的时候会把该库从远程仓库中下载到本地仓库缓存，以后再次构建都不会去访问远程仓库了。所以如果你修改了代码，向远程仓库中发布了新的软件包，但仍然叫Junit-4.10.jar，那么依赖这个库的项目就无法得到最新更新。你只有在重新发布的时候升级版本，比如叫做Junit-4.11.jar，然后通知依赖该库的项目组也修改依赖版本为Junit-4.11,这样才能使用到你最新添加的功能。

这种方式在团队内部开发的时候会变的特别蛋痛。假设有两个小组负责维护两个组件，example-service和example-ui,其中example-ui项目依赖于example-service。而这两个项目每天都会构建多次，如果每次构建你都要升级example-service的版本，那么你会疯掉。这个时候SNAPSHOT版本就派上用场了。每天日常构建时你可以构建example-service的快照版本，比如example-service-1.0-SNAPSHOT.jar，而example-ui依赖该快照版本。每次example-ui构建时，会优先去远程仓库中查看是否有最新的example-service-1.0-SNAPSHOT.jar，如果有则下载下来使用。即使本地仓库中已经有了example-service-1.0-SNAPSHOT.jar，它也会尝试去远程仓库中查看同名的jar是否是最新的。有的人可能会问，这样不就不能充分利用本地仓库的缓存机制了吗？别着急，Maven比我们想象中的要聪明。在配置Maven的Repository的时候中有个配置项，可以配置对于SNAPSHOT版本向远程仓库中查找的频率。频率共有四种，分别是always、daily、interval、never。当本地仓库中存在需要的依赖项目时，always是每次都去远程仓库查看是否有更新，daily是只在第一次的时候查看是否有更新，当天的其它时候则不会查看；interval允许设置一个分钟为单位的间隔时间，在这个间隔时间内只会去远程仓库中查找一次，never是不会去远程仓库中查找（这种就和正式版本的行为一样了）。

Maven版本的配置方式为：

```
<repository>
    <id>myRepository</id>
    <url>...</url>
    <snapshots>
        <enabled>true</enabled>
        <updatePolicy>XXX</updatePolicy>
    </snapshots>
</repository>
```

其中updatePolicy就是那4种类型之一。如果配置间隔时间更新，可以写作interval:XX(XX是间隔分钟数)。daily配置是默认值。

而在Gradle，可以设置本地缓存的更新策略。

```
configurations.all {
	// check for updates every build
	resolutionStrategy.cacheChangingModulesFor  0,'seconds'
}
```

当然也可以按照分钟或者小时来设置.

```
configurations.all {
	resolutionStrategy.cacheChangingModulesFor  10, ‘minutes'
}
configurations.all {
	resolutionStrategy.cacheChangingModulesFor  4, ‘hours'
}
```

所以一般在开发模式下，我们可以频繁的发布SNAPSHOT版本，以便让其它项目能实时的使用到最新的功能做联调；当版本趋于稳定时，再发布一个正式版本，供正式使用。当然在做正式发布时，也要确保当前项目的依赖项中不包含对任何SNAPSHOT版本的依赖，保证正式版本的稳定性。