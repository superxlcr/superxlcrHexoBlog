---
title: TCP/IP常见问题总结（三）
tags: [计算机网络,TCP,Http]
categories: [计算机网络]
date: 2016-04-15 11:19:14
description: Http怎么处理长连接、Cookie与Session的作用与原理、电脑上访问一个网页的整个过程、电脑上访问一个网页的整个过程
---
上一篇文章的传送门：[TCP/IP常见问题总结（二）](/2016/04/07/TCP-IP常见问题总结（二）/)

# Http怎么处理长连接

长连接，指在一个连接上可以连续发送多个数据包，在连接保持期间，如果没有数据包发送，需要双方发链路检测包。
Http协议是一个短连接、无状态的基于TCP的协议，默认的流程是：建立连接-->发送请求-->响应请求-->断开连接。
在Http1.1中加入了保持长连接的功能，可以通过Http头域属性Connection:keep-alive开启。
另外，我们也可以使用Http轮询来模拟长连接的过程。

# Cookie与Session的作用与原理

由于Http协议是一种短连接的、无状态的协议，因此如果我们一般无法获取客户端以前的请求信息与状态信息。
为了解决这个问题，保存请求的状态，Cookie和Session出现了。简单来说，Cookie存于客户端，而Session存于服务器，它们都保存和用户历史请求相关的信息。在发送Http请求时Cookie或SessionId会随被加入请求参数中一起发给服务器，服务器根据Cookie的信息或SessionId查找到的Session信息来得知客户端的状态信息，进而进行进一步的相关操作。

## Cookie

1. cookie被存储于客户端
2. cookie在RFC有定义
3. 与cookie相关的头域参数有cookie和set-cookie

## Session

1. session被存储与服务器
2. session并没有在Http协议中有明确的定义
3. session通过sessionId来区分不同的用户
4. 与cookie相比，存储在服务器的session显得更为安全
5. session通常的实现方法有两种：
	- 通过cookie实现，在cookie中存储sessionId
	- 回写URL实现，在网页的连接中加入sessionId
	
# 电脑上访问一个网页的整个过程是怎么样的：DNS、HTTP、TCP、OSPF、IP、ARP

在电脑上访问一个网页主要有一下步骤：
1. 我们在浏览器中输入网址，但电脑并不认识如此复杂的字符串，因此电脑使用DNS服务解析出对应的IP地址
2. 访问网页的过程基于Http协议，因此我们访问网页的过程实际上就是Http协议发送请求和响应请求的过程
3. Http协议是应用层协议，它基于传输层的TCP协议，TCP协议描述了进程与进程间如何通信交流，把具体的传输过程交给下层的网络层
4. 网络层由IP协议控制数据包的路由选择和存储转发，但由于网络上的主机太多，IP协议管理不过来，因此又把网络主机分为多块，在外部使用EGP外部网关协议，在内部使用IGP内部网关协议（包括RIP协议与OSPF协议）
5. 当网络层的数据传输到以太网后，就通过ARP协议获取目标主机的MAC地址，最后把数据传输的任务交给数据链路层处理

# ICMP报文与Ping的整个过程

ICMP即Internet Control Message Protocol，网络控制报文协议，是一个网络层的用于在IP主机、路由器之间传递控制消息的协议。控制消息是指网络通不通、主机是否可达、路由是否可用等网络本身的消息。这些控制消息虽然并不传输用户数据，但是对于用户数据的传递起着重要的作用。
其中Ping是ICMP协议中的指令之一，其主要作用的是检查网络的连通情况和检测网络的速度。
Ping的过程主要有如下步骤：
1. 检查本地ARP缓存，查找目标主机的MAC地址
2. 如果没有找到MAC地址，使用ARP协议获取目标主机MAC地址，并放入ARP缓存
3. 发出ICMP echo request包，接收ICMP echo reply包