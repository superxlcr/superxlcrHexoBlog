---
title: Android String 中的 %d %n$d
tags: [android,资源文件]
categories: [android]
date: 2017-07-09 15:49:57
description: Android String 中的 %d %n$d
---
在Java中，我们可以使用String的format函数来实现格式化字符串的效果。
在Android中，由于我们的字符串String往往是定义在xml文件中的。因此，以整型为例，除了原本的%d以外，我们还可以使用%n$d的形式。其中，n代表format函数的具体替换参数中的第几个参数。
举例：
Java代码如下，获取并格式化字符串后，通过Log打印出来：

```java
        String str = String.format(getResources().getString(R.string.strTest), 1, 2, 3);
        Log.d(TAG , str);
```

xml文件如下：
使用%n$d的情况：

```html
<string name="strTest">String format test : %1$d %2$d %3$d %2$d</string>
```

打印Log如下，依次打印第一、第二、第三、第二个整型数据：

```
MyLog: String format test : 1 2 3 2
```


使用%d的情况如下，记得把formatted属性设置为false：

```html
<string name="strTest" formatted="false">String format test : %d %d %d</string>
```
打印Log如下，依次打印整型数据：

```
MyLog: String format test : 1 2 3
```




