---
title: CentOS 配置 apache+php+mysql
tags: [操作系统,linux]
categories: [操作系统]
date: 2016-05-23 11:32:17
description: Apache、Mysql、php
---
最近博主配置了自己的CentOS云服务器的php环境，期间遇到了不少困难，查询了许多资料才得以解决，在此写下博客以记录。

# Apache

apache默认在CentOS中已经安装了，输入以下指令可以查看版本：
```
apachectl -v
```

如果提示没安装apache的话使用yum安装即可

# Mysql

首先我们尝试使用yum安装：
```
yum install mysql
yum install mysql-server
yum install mysql-devel
```

然而如果系统是CentOS 7的话，会发现如下错误：
mysql与mysql-devel均安装成功，mysql-server安装失败。
```
No package mysql-server available.
Error: Nothing to do
```

根据网上查询的资料得知：CentOS 7 版本将MySQL数据库软件从默认的程序列表中移除，用mariadb代替了。
有两种解决办法：
1. 安装mariadb
2. 官网下载安装mysql-server

## 安装mariadb

MariaDB数据库管理系统是MySQL的一个分支，主要由开源社区在维护，采用GPL授权许可。开发这个分支的原因之一是：甲骨文公司收购了MySQL后，有将MySQL闭源的潜在风险，因此社区采用分支的方式来避开这个风险。MariaDB的目的是完全兼容MySQL，包括API和命令行，使之能轻松成为MySQL的代替品。

使用yum安装mariadb：
```
yum install mariadb-server mariadb
```

安装完成后，开启mariadb服务：
```
systemctl start mariadb
```

就可以正常的使用mysql了：
```
mysql -u root -p
```

## 官网下载安装mysql-server

在合适的工作目录下下载文件并安装：
```
wget http://dev.mysql.com/get/mysql-community-release-el7-5.noarch.rpm
rpm -ivh mysql-community-release-el7-5.noarch.rpm
yum install mysql-community-server
```

安装完成后启动mysql服务：
```
service mysqld restart
```

然后就可以正常的使用mysql了（初次登陆没有密码）：
```
mysql -u root
```

使用SQL设置密码：
```
set password for 'root'@'localhost' =password('password');
```

# php

美国时间2014年11月13日，PHP开发团队，在「PHP 5.6.3 is available｜PHP: Hypertext Preprocessor」上公布了PHP5.6系的最新版本「PHP 5.6.3」。
在最新的版本5.6.3不仅修改了多个Bug，并且修改了fileinfo模块里存在的安全漏洞。
PHP团队推荐使用PHP5.6系列的用户，升级到最新版本5.6.3。

追加CentOS 6.5的epel及remi源：
```
rpm -Uvh http://ftp.iij.ad.jp/pub/linux/fedora/epel/6/x86_64/epel-release-6-8.noarch.rpm
rpm -Uvh http://rpms.famillecollet.com/enterprise/remi-release-6.rpm
```

CentOS 7的源：
```
yum install epel-release
rpm -ivh http://rpms.famillecollet.com/enterprise/remi-release-7.rpm
```

查看可安装的包：
```
yum list --enablerepo=remi --enablerepo=remi-php56 | grep php
```

yum源配置好了，下一步就安装PHP5.6：
```
yum install --enablerepo=remi --enablerepo=remi-php56 php php-opcache php-devel php-mbstring php-mcrypt php-mysqlnd php-phpunit-PHPUnit php-pecl-xdebug php-pecl-xhprof
```

安装完后查看php版本：
```
[root@VM_120_39_centos ~]# php --version
PHP 5.6.21 (cli) (built: Apr 28 2016 07:39:37)
Copyright (c) 1997-2016 The PHP Group
Zend Engine v2.6.0, Copyright (c) 1998-2016 Zend Technologies
    with Zend OPcache v7.0.6-dev, Copyright (c) 1999-2016, by Zend Technologies
    with Xdebug v2.4.0, Copyright (c) 2002-2016, by Derick Rethans
```

安装完php后，我们需要配置apache的配置文件httpd.conf，以使apache支持php：
```
vim /etc/httpd/conf/httpd.conf
```

首先找到AddType，添加如下二行：
```
AddType application/x-httpd-php  .php
AddType application/x-httpd-php-source  .phps
```

然后定位至DirectoryIndex index.html，修改为：
```
DirectoryIndex  index.php  index.html
```

重启apache服务：
```
service httpd restart
```

CentOS的apache网页目录在/var/www/html中，转到该目录下，新建文件index.php测试吧：
```php
<?php
$conn=mysql_connect('localhost','root','');
  if ($conn)
    echo "Connect Success...";
  else
    echo "Connect Failure...";
 phpinfo();
?>
```

参考网页资料：
Mysql
http://www.cnblogs.com/starof/p/4680083.html
php安装
http://my.oschina.net/u/573270/blog/423238
php配置
http://jingyan.baidu.com/article/d5c4b52bec7d6bda560dc5fb.html