---
title: Java构造并初始化Map
tags: [java,基础知识]
categories: [java]
date: 2018-06-30 10:16:45
description: static块初始化、双括号初始化（匿名内部类）、Guava工具包
---
在Java中，优雅的构造并初始化Map的方式有三种，在此记录一下

# static块初始化

```java
private static final Map<String, String> MAP;
static {
	MAP = new HashMap<String, String>();
	MAP.put("xx", "xx");
	MAP.put("yy", "yy");
}
```

# 双括号初始化（匿名内部类）

```java
HashMap<String, String > h = new HashMap<String, String>(){{  
	put("xx","xx");      
}}; 
```

这种写法本质上是创建了一个匿名内部类，因此会持有外部类实例的引用，如果拥有比外部类实例更长的生命周期，有内存泄漏的风险，需要慎重使用

# Guava工具包

使用Google提供的Guava工具包
jcenter库依赖：com.google.guava:guava:21.0
```java
Map<String, Integer> test1 = ImmutableMap.of("xx", 1, "yy", 2, "zz", 3);  
// 或者  
Map<String, String> test2 = ImmutableMap.<String, String>builder()  
    .put("xx", "xx")  
    .put("yy", "yy")  
    ...  
    .build();  
```