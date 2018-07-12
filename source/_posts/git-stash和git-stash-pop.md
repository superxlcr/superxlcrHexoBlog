---
title: git stash和git stash pop
tags: [git]
categories: [git]
date: 2017-07-30 12:12:40
description: （转载）git stash和git stash pop
---
本文转载自：http://blog.csdn.net/wh_19910525/article/details/7784901


更多git stash相关介绍可查看：
https://git-scm.com/book/zh/v1/Git-%E5%B7%A5%E5%85%B7-%E5%82%A8%E8%97%8F%EF%BC%88Stashing%EF%BC%89


git stash 可用来暂存当前正在进行的工作， 比如想pull 最新代码， 又不想加新commit， 或者另外一种情况，为了fix 一个紧急的bug,  先stash, 使返回到自己上一个commit, 改完bug之后再stash pop, 继续原来的工作。

基础命令：
```
git stash

do some work

git stash pop
```


进阶：

Git stash save "work in progress for foo feature"

当你多次使用’git stash’命令后，你的栈里将充满了未提交的代码，这时候你会对将哪个版本应用回来有些困惑，
’**git stash list**’ 命令可以将当前的Git栈信息打印出来，你只需要将找到对应的版本号，例如使用’git stash apply stash@{1}’就可以将你指定版本号为stash@{1}的工作取出来，当你将所有的栈都应用回来的时候，可以使用’**git stash clear**’来将栈清空。

```
git stash          # save uncommitted changes
# pull, edit, etc.
git stash list     # list stashed changes in this git
git show stash@{0} # see the last stash 
git stash pop      # apply last stash and remove it from the list

git stash --help   # for more info
```