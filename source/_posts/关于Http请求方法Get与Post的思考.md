---
title: 关于HTTP请求方法GET与POST的思考
tags: [计算机网络,HTTP]
categories: [计算机网络]
date: 2018-04-15 15:22:33
description: HTTP简述、GET方法、POST方法、GET方法与POST方法对比
---
最近在做需求时处理了关于HTTP请求方法的问题，在此写篇博客做一下记录

# HTTP简述

HTTP，即超文本传输协议(HyperText Transfer Protocol)，互联网上应用最为广泛的一种网络协议，是一个以request-response(请求-回复)形式工作的应用层协议。
在客户端向服务器发起一个HTTP请求的时候，我们可以选择相应的请求方法，其中最为常用的方法，就是GET方法与POST方法。

# GET方法

在使用GET方法发送HTTP请求时，一般而言，查询的键值对参数被附加在url地址的后面：
```
/test/demo_form.asp?name1=value1&name2=value2
```
GET请求具有如下特点：
1. GET 请求可被缓存
2. GET 请求保留在浏览器历史记录中
3. GET 请求可被收藏为书签
4. GET 请求不应在处理敏感数据时使用
5. GET 请求有长度限制
6. GET 请求只应当用于取回数据

# POST方法

在使用POST方法发送HTTP请求时，一般而言，查询的键值对参数是添加在HTTP消息主体中发送的：
```
POST /test/demo_form.asp HTTP/1.1
Host: w3schools.com
name1=value1&name2=value2
```

POST请求具有如下特点：
1. POST 请求不会被缓存
2. POST 请求不会保留在浏览器历史记录中
3. POST 不能被收藏为书签
4. POST 请求对数据长度没有要求

# GET方法与POST方法对比

| 方法 | GET | POST |
| - | - | - |
| 后退按钮/刷新 | 没影响 | 数据会被重新提交（浏览器应该告知用户数据会被重新提交） |
| 书签 | 可收藏为书签 | 不可收藏为书签 |
| 缓存 | 可缓存 | 不可缓存 |
| 编码类型 | application/x-www-form-urlencoded | application/x-www-form-urlencoded 或 multipart/form-data。为二进制数据使用多重编码。 |
| 历史记录 | 参数保留在浏览器历史中 | 参数不会保留在浏览器历史中 |
| 对参数数据长度的限制 | 当发送数据时，GET 方法向 URL 添加数据；URL 的长度是受限制的（URL 的最大长度是 2048 个字符） | 无限制 |
| 对数据类型的限制 | 只允许 ASCII 字符 | 没有限制，也允许二进制数据 |
| 安全性 | 与 POST 相比，GET 的安全性较差，因为所发送的数据是 URL 的一部分 | POST 比 GET 更安全，因为参数不会被保存在浏览器历史或 web 服务器日志中 |
| 可见性 | 数据在 URL 中对所有人都是可见的 | 数据不会显示在 URL 中 |

对于两种方法安全性的思考：其实如果对于能够通过Fiddler或者Charles工具抓包的童鞋而言，两种方法其实都相当于是明文传输，都不安全。只不过POST方法由于参数在HTTP请求主体中，一般而言在浏览器上不容易看到，相对安全

对于两种方法参数传输的思考：

## GET方法和POST方法与数据如何传递没有关系

GET和POST是由HTTP协议定义的。在HTTP协议中，Method和Data（url， body， header）是正交的两个概念，也就是说，使用哪个Method与应用层的数据如何传输是没有相互关系的
HTTP没有要求，如果Method是POST数据就要放在body中。也没有要求，如果Method是GET，数据（参数）就一定要放在URL中而不能放在BODY中

## HTTP协议对GET和POST都没有对长度的限制

HTTP协议明确地指出了，HTTP头和Body都没有长度的要求。而对于url长度上的限制，有两方面的原因造成：

1. 浏览器。据说早期的浏览器会对URL长度做限制。据说IE对url长度会限制在2048个字符内。
2. 服务器。url长了，对服务器处理也是一种负担。过长的url会增加服务器的解析时间，因此多数服务器出于安全以及稳定方面的考虑，会给url长度加限制。但是这个限制是针对所有HTTP请求的，与GET、POST没有关系。