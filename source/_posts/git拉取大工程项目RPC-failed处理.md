---
title: git拉取大工程项目RPC failed处理
tags: [git]
categories: [git]
date: 2019-09-03 18:15:35
description: 加大缓存区、换协议、只拷贝部分代码
---

最近博主在拉取某个大项目的时候遇到了下面的错误：
```
error: RPC failed; curl 18 transfer closed with outstanding read data remaining
```

在网上找了一波资料后，发现问题出在项目文件太多太大了
网上总结的解决方案主要有以下三种：

# 加大缓存区 

```
git config --global http.postBuffer 524288000
```

这个大约是500M，不过博主试了更大的数值，依旧没有任何效果，估计是项目太大了

# 换协议

```
git clone https://github.com/flutter/flutter.git 
git clone git://github.com/flutter/flutter.git
```

即clone http方式换成SSH的方式，即 https:// 改为 git:// 
可惜的是，博主拉取代码的gitlab并不支持ssh的方式

# 只拷贝部分代码

```
git clone https://github.com/flutter/flutter.git --depth 1
```

我们可以在clone指令上使用depth参数，来控制此次拷贝的深度
--depth 1的含义是复制深度为1，就是每个文件只取最近一次提交，不是整个历史版本
官方说明如下：
```
--depth <depth>
Create a shallow clone with a history truncated to the specified number of commits. Implies --single-branch unless --no-single-branch is given to fetch the histories near the tips of all branches. If you want to clone submodules shallowly, also pass --shallow-submodules.
```

不过值得注意的是这种方式拷贝下来的仓库是个shallow版本的，即浅拷贝版本的
这种shallow版本的仓库是不允许我们进行push代码的
因此当我们想要把仓库变回一个平时正常的仓库时，我们需要使用unshallow指令，把剩余的历史记录拉下来：
```
git fetch --unshallow
```

值得注意的是fetch指令也能使用depth参数，因此当我们直接unshallow拉取剩余历史记录依旧出现RPC-failed失败时，我们不妨逐步增大depth参数：
```
git fetch --depth 50
```

在多次拉取部分历史记录后，再使用unshallow拉取剩余历史记录，可以使得我们的仓库完全变成正常的样子

值得注意的是，在使用了unshallow指令之后，虽然仓库变成了完整的，但是我们会发现只能拉取一个主分支相关的代码，并不能获取其他远程分支相关的代码，这个时候，我们还需要执行以下命令，把仓库的目标分支改为所有分支：
```
git config remote.origin.fetch "+refs/heads/*:refs/remotes/origin/*"
```


更多关于shallow clone的讨论可以参考：
https://stackoverflow.com/questions/6802145/how-to-convert-a-git-shallow-clone-to-a-full-clone/17937889#17937889
https://stackoverflow.com/questions/23708231/git-shallow-clone-clone-depth-misses-remote-branches