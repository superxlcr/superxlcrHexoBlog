---
title: md5介绍
tags: [android,java,基础知识]
categories: [java]
date: 2019-11-03 11:00:11
description: md5算法介绍、md5算法Android实现、32位、16位md5输出
---

最近博主在处理一些异常上报相关的需求，由于异常上报的日志字数是有限的，因此在参考回顾一些相关的摘要算法
在此，整理一下需求中此次用到的md5算法

# md5算法介绍

md5算法全称：MD5信息摘要算法（英语：MD5 Message-Digest Algorithm）
是一种被广泛使用的密码散列函数，可以产生出一个128位（16字节）的散列值（hash value），用于确保信息传输完整一致

md5算法主要有以下特点：
- 通常情况下，不同明文md5计算后的密文不相同
- 计算速度快，加密速度快，不需要秘钥
- 不可逆

md5算法一般有以下用途：
- 密码管理，用md5密文代替密码保存至数据库，防止数据库破解后密码泄漏
- 数字签名，验证身份
- 校验信息，防止内容被篡改

虽然现在已经有人证明了md5是并不安全的，可以被破解的，但对于安全性要求不是特别高的一般场景而言，我们还是可以选用md5算法

以上内容搬运自百度百科
详情参考：
https://baike.baidu.com/item/MD5/212708?fr=aladdin

# md5算法Android实现

主要通过系统预置的java.security.MessageDigest接口来实现：

```java
public synchronized static String getMd5(String src) {
	try {
		MessageDigest md = MessageDigest.getInstance("MD5");
		md.update(src.getBytes());
		return bytesToHex(md.digest());
	} catch (Exception e) {
		e.printStackTrace();
	}
	return null;
}

private static String bytesToHex(byte[] bytes) {
	StringBuilder sb = new StringBuilder();
	for (int i = 0; i < bytes.length; i++) {
		String hex = Integer.toHexString(0xFF & bytes[i]);
		if (hex.length() == 1) {
			sb.append('0');
		}
		sb.append(hex);
	}
	return sb.toString();
}
```

md5密文计算后的输出都是512bit的，即长度为32的16进制字符串

# 32位、16位md5输出

md5密文的输出除了区分大小写以外，还有32位与16位的分别
上面说到md5密文的输出都是32位的
其实16位的md5密文就是32位密文的第9位到第24位的部分
代码表示如下：

```java
String md5_16 = md5_32.substring(8, 24);
```