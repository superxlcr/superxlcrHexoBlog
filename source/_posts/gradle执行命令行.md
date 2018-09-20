---
title: gradle执行命令行
tags: [gradle]
categories: [gradle]
date: 2018-09-20 21:13:11
description: 如何执行命令行、区分操作系统
---

最近博主在编写gradle相关脚本时，需要执行相关的命令行指令，在此记录一下踩过的坑

# 如何执行命令行

首先，需要执行命令行，我们有两种不同的方式达成：

可以通过定义type类型为Exec的task来完成：
```
task execTask(type: Exec) {
	workingDir './'
	commandLine 'cmd', '/c', ...
}
```

也可以通过调用Project#exec来完成：
```
exec {
	workingDir = './'
	def commands = []
	commands << 'cmd'
	commands << '/c'
	...
	commandLine = commands
}
```

# 区分操作系统

值得注意的是，一般而言我们并不能直接把需要执行的指令输入给commandLine变量，而是需要在前面加入'cmd /c'参数才行（这里代表的应该是指重新打开一个命令行窗口执行相关指令）
当然这只是在windows系统上的做法，在其他操作系统上，我们应该加入参数'bash -c'

因此我们应该补上这样一段代码：
```
def commands = []
if (System.getProperty("os.name").toLowerCase().startsWith("windows")) {
	commands << 'cmd'
	commands << '/c'
} else {
	commands << 'bash'
	commands << '-c'
}
commands << realCommands
commandLine = commands
```