---
title: Java读取文件byte转化String问题
tags: [java,应用]
categories: [java]
date: 2017-03-16 20:25:46
description: Java读取文件byte转化String问题
---
最近接触了关于读取文件获得byte数组转化为String后的问题，具体代码如下：


```java
// 打开图片作为临时文件
File tempFile = new File("C:/Users/Administrator/Desktop/test.jpg");
try {
	FileInputStream fis = new FileInputStream(tempFile);
	byte[] bytes = new byte[1024];
	// 读取文件byte数组
	fis.read(bytes);
	System.out.println("bytes.length " + bytes.length);
	// 直接转化为String
	String bytesStr = new String(bytes);
	System.out.println("bytesStr.length() " + bytesStr.length());
	System.out.println("bytesStr.getBytes().length " + bytesStr.getBytes().length);
	// 使用ISO-8859-1编码转化为String			
	String bytesStr2 = new String(bytes, "ISO-8859-1");
	System.out.println("bytesStr2.length() " + bytesStr2.length());
	System.out.println("bytesStr2.getBytes(\"ISO-8859-1\").length " + bytesStr2.getBytes("ISO-8859-1").length);
} catch (IOException e) {
	e.printStackTrace();
}
```


运行出来效果如下：

![Byte to String_pic1](1.png)


可以发现在使用ISO-8859-1编码后，byte和String的转化问题得到了解决，长度达到了一致。
