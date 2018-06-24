---
title: Github 之 SSH key的创建与配置（Windows）
tags: [git,ssh]
categories: [git]
date: 2016-05-09 17:47:22
description: （翻译）Github 之 SSH key的创建与配置
---
最近配置了github的ssh key，翻找了大量资料后发现github官方就有相关的教程……在此翻译一下官方教程以加深印象
原文链接：https://help.github.com/categories/ssh/

# Generating an SSH key（生成SSH key）

SSH密钥是来识别值得信赖的电脑的方法。您可以生成一个SSH密钥，并按照本节所述的方法将公共密钥添加到您的帐户GitHub中

# Checking for existing SSH keys（检查已存在的SSH key）

在你生成一个ssh key之前，你可以检查一下你是否已经有了ssh key：
1. 打开Git Bash
2. 输入
	```
	ls -al ~/.ssh
	```
	来查看是否有ssh key存在
3. 检查/.ssh目录来查看是否存在公开的ssh key

一般而言，公开的ssh key的文件名为以下几种：
- id_dsa.pub
- id_ecdsa.pub
- id_ed25519.pub
- id_rsa.pub

# Generating a new SSH key and adding it to the ssh-agent（生成一个新的SSH key并添加到ssh-agent）

在你检查过存在的ssh key后，你可以新建一个ssh key：
1. 打开Git Bash
2. 输入这一串：
	```
	ssh-keygen -t rsa -b 4096 -C "your_email@example.com"  
	```
	生成一个新的ssh key，使用填入的邮箱地址作为ssh key的标签，并生成RSA密钥对
3. 看到如下提示时：
	```
	Enter a file in which to save the key (/Users/you/.ssh/id_rsa): [Press enter]  
	```
	按下回车，表示把ssh key放在默认地址
4. 然后为ssh key设置密码：
	```
	Enter passphrase (empty for no passphrase): [Type a passphrase]  
	Enter same passphrase again: [Type passphrase again]  
	```
	
创建完ssh key后，你需要把它添加到ssh-agent中去：
1. 首先保证ssh-agent启用了：
	```
	eval "$(ssh-agent -s)"  
	```
	该指令返回进程id则表示已经启用ssh-agent
2. 使用如下指令把ssh key添加到ssh-agent中：
	```
	ssh-add ~/.ssh/id_rsa  
	```
	
# Adding a new SSH key to your GitHub account（为你的github账号添加SSH key）

在把ssh key添加到ssh-agent后，你需要把ssh key添加到你的github账号中：
1. 打开Git Bash，使用指令把ssh key复制到剪贴板：
	```
	clip < ~/.ssh/id_rsa.pub  
	```
	如果不成功就用编辑器打开该文件直接复制内容
2. 在github右上角点击**setting**：
	![setting位置示意图](1.png)
3. 在左边选择**SSH and GPG keys**：
	![SSH and GPG keys位置示意图](2.png)
4. 点击**New SSH key**：
	![New SSH key位置示意图](3.png)
5. 在Title处为你的ssh key填入适当的标题，在Key处粘贴你复制的ssh key
	![填入ssh key](4.png)
6. 	点击**Add SSH key**：
	![Add SSH key](5.png)
7. 输入你的github账号密码确认此次操作

# Testing your SSH connection（测试你的SSH连接）

在进行完上面一系列操作后，是时候看看你的SSH连接是否成功了：
1. 打开Git Bash
2. 输入以下指令：
	```
	ssh -T git@github.com  
	```
	尝试去用ssh连接github，你可能会看到一些警告信息：
	```
	The authenticity of host 'github.com (192.30.252.1)' can't be established.  
	RSA key fingerprint is 16:27:ac:a5:76:28:2d:36:63:1b:56:4d:eb:df:a6:48.  
	Are you sure you want to continue connecting (yes/no)?  
	```
	输入yes不管他就好
3. 如果你看到一下信息：
	```
	Hi username! You've successfully authenticated, but GitHub does not provide shell access  
	```
	则表示ssh连接成功了
4. 如果你收到的信息是"access denied" ,那么你可以参考一下链接进行进一步处理：https://help.github.com/articles/error-permission-denied-publickey/

# Changing a remote's URL（改变远程仓库的URL）

在设置完ssh后，你可能需要把你的远程仓库的URL从HTTPS改为SSH（SSH好处在于不用每次push都输账号密码……）：
1. 打开GIt Bash
2. 把工作目录转到你的本地工程中
3. 查看拥有的远程仓库：
	```
	git remote -v  
	```
4. 更改远程仓库的url：
	```
	git remote set-url origin https://github.com/USERNAME/OTHERREPOSITORY.git  
	```
	origin为仓库名，后面接的是ssh仓库地址：
5. 查看拥有的远程仓库，看看是否修改成功：
	```
	git remote -v  
	```

至此Github的SSH key配置大功告成，以后push再也不用每次都输入github的账号密码了～