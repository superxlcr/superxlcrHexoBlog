---
title: Linux Java程序debug总结
tags: [java,linux,通信]
categories: [java]
date: 2017-04-28 12:35:52
description: 使用Linux shell进行debug、使用jvisualvm进行远程debug
---
最近博主debug了远程Linux服务器上的Java程序，在此对过程中所使用的工具进行一番总结。
# 使用Linux shell进行debug
通过putty登录到Linux服务器，我们可以使用Linux上的shell命令进行debug。


首先我们可以使用

```
ps -e
```

指令来确认Java进程的pid：
![Java_pid_get_pic1](1.png)


然后我们可以使用

```
top -p pid
```
来查看该进程所消耗的内存，与cpu占用情况：
![top_pid_pic2](2.png)


在确认我们Java程序所在进程出现问题后，我们使用如下指令来查看进程中线程的时间消耗情况：

```
ps -mp pid -o THREAD,tid,time
```
![ps_tid_get_pic3](3.png)


通过对比各个线程消耗CPU时间的多少我们可以大致判定出问题所在线程，首先我们先把线程号转化为16进制格式：

```
printf "%x\n" tid
```
然后通过jstack指令打印线程运行信息：

```
jstack pid |grep tid -A 30
```
上述指令的意思为：打印pid进程的Java运行信息，截取带有tid线程号的行并输出其后30行的信息

# 使用jvisualvm进行远程debug

首先我们需要修改Linux 服务器上的/etc/hosts文件，把对应的hostname改为机器的外网ip：

```
vi /etc/hosts
```
![etc_hosts_pic4](4.png)


然后在我们启动Java进程时加上以下配置参数：

```
-Djava.rmi.server.hostname=xxx.xxx.xxx.xxx # Linux主机外网ip
-Dcom.sun.management.jmxremote
-Dcom.sun.management.jmxremote.port=28888 # 通信端口，在jvisualvm中需要填入
-Dcom.sun.management.jmxremote.authenticate=false # 不需要用户名密码登录
-Dcom.sun.management.jmxremote.ssl=false
```



然后我们启动jvisualvm，该工具为JDK自带，在JDK安装目录/bin目录下。
打开后我们使用远程监控：
![远程_pic5](5.png)



右键添加远程主机，输入ip添加完成。
然后再右键添加JMX连接，输入对应的ip:port即可。


通过jvisualvm，我们既可以查看Java虚拟机总体的资源使用情况：
![总览_pic6](6.png)



又可以查看其中每个线程的状况，必要时还可以dump内存进行debug：
![线程_pic7](7.png)

