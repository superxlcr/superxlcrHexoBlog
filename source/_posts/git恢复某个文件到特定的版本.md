---
title: git恢复某个文件到特定的版本
tags: [git]
categories: [git]
date: 2019-07-30 19:29:09
description: git恢复某个文件到特定的版本
---

最近博主遇到一个需要git回滚代码的问题

一般而言，当我们需要把代码回滚到某次提交的情况下，我们会使用如下指令：
```
# 首先获取提交的哈希字符串
git log
# 然后重置回该次提交，其中**代表提交的哈希字符串
git reset --hard **
```

但是这样子会把所有文件都重置回该次提交的状态，如果我们只需要重置特定的文件呢？

经过搜索，博主发现了stackoverflow上的回答：
https://stackoverflow.com/questions/215718/how-can-i-reset-or-revert-a-file-to-a-specific-revision

最高票的回答告诉我们，只需要执行以下指令即可：
```
# 其中**代表提交的哈希字符串
git checkout ** -- file1/to/restore file2/to/restore
```

除此以外，我们还可以通过checkout指令把文件切换到指定分支的版本：
```
# 把file1 file2切换到develop分支的版本
git checkout develop -- file1/to/restore file2/to/restore
```