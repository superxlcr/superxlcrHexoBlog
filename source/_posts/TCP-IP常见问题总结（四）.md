---
title: TCP/IP常见问题总结（四）
tags: [计算机网络,TCP,Http]
categories: [计算机网络]
date: 2016-04-17 21:29:06
description: C/S模式下使用socket通信、IP地址分类、路由器与交换机区别
---
上一篇文章的传送门：[TCP/IP常见问题总结（三）](/2016/04/15/TCP-IP常见问题总结（三）/)

# C/S模式下使用socket通信

客户端的Java代码如下所示：
```java
public class Main {  
    public static void main(String[] args) throws Exception {  
        String host = "";  
        int port = 0;  
        Socket socket = new Socket(host, port); // 分别填入目标主机ip和端口  
        try {  
            // 获取输出流  
            OutputStream os = socket.getOutputStream();  
            // 获取输入流  
            InputStream is = socket.getInputStream();  
        } finally {  
            // 关闭socket  
            socket.close();  
        }  
    }  
}  
```

建立连接后，获取输入输出流进行对应的输入输出即可。
服务器的Java代码如下所示：
```java
public class Main {  
    public static void main(String[] args) throws Exception {  
        int port = 0;  
        ServerSocket serverSocket = new ServerSocket(port); // 填入监听的端口号  
        try {  
            // accept是一个阻塞的方法，阻塞直到返回一个socket连接  
            Socket socket = serverSocket.accept();  
            try {  
                // 获取输入输出流进行对应操作  
            } finally {  
                socket.close();  
            }  
        } finally {  
            serverSocket.close();  
        }  
    }  
}  
```

通过accept获取一个socket连接后类似客户端获取输入输出处理即可。
客户端Java NIO代码：
```java
public class Main {  
    public static void main(String[] args) throws Exception {  
        String hostname = "";  
        int port = 0;  
        SocketChannel socketChannel = SocketChannel.open();  
        // 设置成非阻塞IO  
        socketChannel.configureBlocking(false);  
        try {  
            // 非阻塞模式下可能没建立连接就返回了  
            while (!socketChannel.finishConnect()) {   
                // 传入目标主机ip和端口号建立连接  
                socketChannel.connect(new InetSocketAddress(hostname, port));  
            }  
              
            int capacity = 48;  
            // 传入缓冲区大小建立缓冲区  
            ByteBuffer buffer = ByteBuffer.allocate(capacity);  
              
            // 读取字节输入  
            int byteRead = socketChannel.read(buffer);  
              
        } finally {  
            socketChannel.close();  
        }  
    }  
}  
```

注意NIO下的连接、读取和写入操作均为非阻塞操作，可能并没有达到我们预料中的结果就返回了，因此切记在循环中使用并进行相应的判断。
服务器Java NIO代码：
```java
public class Main {  
    public static void main(String[] args) throws Exception {  
        int port = 0;  
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();  
        // 绑定监听端口  
        serverSocketChannel.bind(new InetSocketAddress(port));  
        // 设置为非阻塞模式  
        serverSocketChannel.configureBlocking(false);  
        try {  
            while (true) {  
                SocketChannel socketChannel = serverSocketChannel.accept();  
                // 非阻塞模式下可能没监听到任何通道就返回了  
                if (socketChannel != null) {  
                    // 获取socket通道后进行对应操作  
                }  
            }  
        } finally {  
            serverSocketChannel.close();  
        }  
    }  
}  
```

由于非阻塞的原因，accept方法不一定成功获取socketChannel，因此我们需要进行判断是否返回了null

# IP地址分类

IP地址分为IPv4地址（32位）和IPv6地址（128位），在此我们讨论IPv4地址。
IP地址由两部分（网络部分和主机部分）组成，可以分为有类网和无类网两类。

## 有类网

有类网分为以下5种：

| 种类 | 定义 | 网络地址范围 |
| - | - | - |
| A类网 | 第一位为0，后7位为网络号，剩余24位为主机号 | 1.0.0.0 到 126.0.0.0 有效（0.0.0.0 与 127.0.0.0保留） |
| B类网 | 前两位为10，后14位为网络号，剩余16位为主机号 | 128.1.0.0 到 191.254.0.0 有效（128.0.0.0 与 191.255.0.0保留） |
| C类网 | 前三位为110，后21位为网络号，剩余8位为主机号 | 192.0.1.0 到 223.255.254.0 有效（192.0.0.0 与 223.255.255.0保留） |
| D类网（不可用） | 前四位为1110，后28位为多播地址 | 224.0.0.0 到 239.255.255.255 用于多点广播 |
| E类网（不可用） | 前四位为1111，被保留 | 240.0.0.0 到 255.255.255.254 保留（255.255.255.255用于广播） |

除了D类网与E类网不能使用外，A、B和C类网IP均可用来表示一台主机。我们一般根据自己网络中主机的多少来选择A、B还是C类网，但一般而言网路中的主机数目都不会刚好等于有类网提供的主机数，于是经常会造成有多余的IP地址浪费，因此我们有了无类网

## 无类网

无类网加入了子网掩码的概念。子网掩码是一个32位地址，用于将某个IP地址划分成网络地址和主机地址两部分。在子网掩码中我们以1表示为网络号，例：255.255.255.0表示前24位为网络号

# 路由器与交换机区别

路由器工作于网络模型的网络层，其主要的功能是路由选择与存储转发，路由器上还能开启ACL访问控制列表、NAT地址转换等功能，扩展网络应用
交换机工作于网络模型的数据链路层，其主要的功能是泛洪、存储转发、过滤和自学习，交换机还能够隔离冲突域，并划分VLAN