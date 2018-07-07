---
title: BufferedReader使用readLine问题
tags: [java,应用]
categories: [java]
date: 2017-03-16 20:53:09
description: BufferedReader使用readLine问题
---
有时我们在使用BufferedReader时候会发现使用readLine函数迟迟没有任何返回，这是因为BufferedReader和BufferedWriter是基于行进行操作的，因此我们使用BufferedWriter的时候使用newLine函数即可，具体代码如下：

```java
		BufferedWriter writer = new BufferedWriter(new OutputStreamWriter(out));
		writer.write(str);
		writer.newLine();
		writer.flush();
		
		BufferedReader reader = new BufferedReader(new InputStreamReader(in));
		str = reader.readLine();
```


 