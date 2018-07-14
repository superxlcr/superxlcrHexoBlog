---
title: Groovy中对xml的操作补充
tags: [gradle,java]
categories: [java]
date: 2017-09-07 23:37:22
description: 一些Groovy中的xml工具类找不到、关于namespace的问题、关于动态修改xml中元素的属性
---
Android中Gradle编译器使用的是Groovy语言，Groovy为我们提供了一系列的工具类用于处理xml文件。
关于Groovy中如何对xml文档进行操作，这里有一处文档：http://www.groovy-lang.org/processing-xml.html
在此，补充一些文档中遗漏的点：

# 一些Groovy中的xml工具类找不到

可以尝试

```java
import groovy.xml.*
```
类似于Namespace、QName以及XmlUtil工具类，均在groovy.xml包中

# 关于namespace的问题

xml中namespace（命名空间）为的是提供避免元素命名冲突的方法，但却让我们访问xml文档变得十分不方便
在Groovy中，我们常用的xml解析器有XmlSlurper以及XmlPraser，他们的具体用法可以参考上面链接中的介绍，下面分别来讲讲两种解析器如何解析带命名空间的xml文件



## XmlSlurper

XmlSlurper比较简单，在解析xml文件的同时声明命名空间即可：

```java
    def testManifest = new XmlSlurper().parse("${WORKSPACE}${SRC_DIR}/AndroidManifest.xml")
    testManifest.declareNamespace('android':'http://schemas.android.com/apk/res/android')
    println testManifest.application[0].@"android:name"
```

上面代码是访问AndroidManifest文件中Application元素下的android:name属性的示例



## XmlParser

XmlParser则比较麻烦，我们需要先声明一个Namespace对象，然后再使用attribute方法获取元素属性（目前找不到别的写法……）

```java
    // 声明命名空间
    def android = new Namespace('http://schemas.android.com/apk/res/android', 'android')

    // 获取apk application name
    def parser = new XmlParser()
    def srcManifest = parser.parse("${WORKSPACE}${SRC_DIR}/AndroidManifest.xml")
    def srcApp = srcManifest.application[0].attribute(android.name)
```


# 关于动态修改xml中元素的属性


在上面链接中，我们学会了通过xmlParser修改xml的元素属性，在此我们再补充一种修改元素属性的方法
由于xml中元素属性载入内存后其实是存在Map中的，因此我们可以通过attributes方法获取Map，并使用put方法修改对应属性：

```java
srcManifest.application[0].attributes().put(android.name, value)
```
上面代码是把AndroidManifest文件中Application元素的android:name属性改为value值




最后，对于Groovy中的类有任何不懂的问题，我们都可以通过查看其文档解决：http://docs.groovy-lang.org/latest/html/api/
