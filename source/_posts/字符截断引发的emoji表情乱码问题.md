---
title: 字符截断引发的emoji表情乱码问题
tags: [android,编码]
categories: [android]
date: 2018-06-19 19:18:00
description: 背景、Unicode编码、Character#isHighSurrogate
---
最近做需求的时候遇到了一个字符串截断导致emoji表情乱码的问题，在此记录一下

# 背景

最近遇到了一个这样的需求：给定一串字符串，截取其中固定长度的字符串下来
本以为是一个使用CharSequence#subSequence方法轻松解决的问题，谁知在QA童鞋测试之后就报来了bug：原来字符串中还包含了emoji表情，由于一个emoji表情占用两个char的大小，如果emoji表情刚好在字符串的最后一位，截断之后的表情只剩下一个char的大小，就变成了乱码的形式
![裁剪前后示意图](1.png)

# Unicode编码

在百度了一些相关资料后，我发现该问题与字符编码相关

## Unicode

Unicode是计算机领域的一项行业标准，它对世界上绝大部分的文字的进行整理和统一编码，Unicode的编码空间可以划分为17个平面（plane），每个平面包含2的16次方（65536）个码位。17个平面的码位可表示为从U+0000到U+10FFFF，共计1114112个码位，第一个平面称为基本多语言平面（Basic Multilingual Plane, BMP），或称第零平面（Plane 0）。其他平面称为辅助平面（Supplementary Planes）。基本多语言平面内，从U+D800到U+DFFF之间的码位区段是永久保留不映射到Unicode字符，所以有效码位为1112064个。

对于被Unicode收录的字符其编码是唯一且确定的。但是Unicode的实现方式(出于传输、存储、处理或向后兼容的考虑)却有不同的几种，其中最流行的是UTF-8、UTF-16、UCS2、UCS4/UTF-32等，细分的话还有大小端的区别。

对于我们Java而言，可以从char占用2字节来推断出使用的是UTF-16编码

## UTF-8

我们先来讲讲UTF-8编码，UTF-8是一种变长编码，对于一个Unicode的字符被编码成1至4个字节。Unicode编码与UTF-8的编码的对应关系如下表：

| Unicode编码 |	UTF-8编码(二进制) |
| - | - |
| U+0000 – U+007F | 0xxxxxxx |
| U+0080 – U+07FF | 110xxxxx 10xxxxxx |
| U+0800 – U+FFFF | 1110xxxx 10xxxxxx 10xxxxxx |
| U+10000 – U+10FFFF | 11110xxx 10xxxxxx 10xxxxxx 10xxxxxx |

其中绝大部分的中文用三个字节编码，部分中文用四个字节编码，举例如下:

| Unicode | 字符 | UTF-8编码 |
| - | - | - |
| U+0041 | A | 0x41 |
| U+7834 | 破 | 0xE7 0xA0 0xB4 |
| U+6653 | 晓 | 0xE6 0x99 0x93 |
| U+2A6A5 | 𪚥(四个龍) | 0xF0 0xAA 0x9A 0xA5 |

## UTF-16

UTF-16是一种变长编码，对于一个Unicode字符被编码成1至2个码元，每个码元为16位

在**基本多语言平面（码位范围U+0000-U+FFFF）**内的码位UTF-16编码使用1个码元且其值与Unicode是相等的（不需要转换）。举例如下：

| Unicode | 字符 | UTF-16（码元） | UTF-16 LE（字节） | UTF-16 BE（字节） |
| - | - | - | - | - |
| U+0041 | A | 0x0041 | 0x41 0x00 | 0x00 0x41 |
| U+7834 | 破 | 0x7834 | 0x34 0x78 | 0x78 0x34 |
| U+6653 | 晓 | 0x6653 | 0x53 0x66 | 0x66 0x53 |

在**辅助平面（码位范围U+10000-U+10FFFF）**内的码位在UTF-16中被编码为一对16bit的码元（即32bit,4字节），称作代理对(surrogate pair)。组成代理对的两个码元前一个称为前导代理(lead surrogates)范围为0xD800-0xDBFF，后一个称为后尾代理(trail surrogates)范围为0xDC00-0xDFFF。UTF-16辅助平面代理对与Unicode的对应关系如下表：

| Lead \ Trail | 0xDC00 | 0xDC01 | … | 0xDFFF |
| - | - | - | - | - |
| 0xD800 | U+10000 | U+10001 | … | U+103FF |
| 0xD801 | U+10400 | U+10401 | … | U+107FF |
| ⋮ | ⋮ | ⋮ | ⋱ | ⋮ |
| 0xDBFF | U+10FC00 | U+10FC01 | … | U+10FFFF |

举例如下：

| Unicode | 字符 | UTF-16（码元） | UTF-16 LE（字节） | UTF-16 BE（字节） |
| - | - | - | - | - |
| U+2A6A5 | 𪚥 | 0xD869 0xDEA5 | 0x69 0xD8 0xA5 0xDE | 0xD8 0x69 0xDE 0xA5 |

# Character#isHighSurrogate

由上面的UTF-16编码知识可以推断出，我们的emoji表情删除一个char后出现乱码的原因，是因为它是属于UTF-16编码辅助平面内的代理对
对于这种情况，我们可以通过Character类的静态方法isHighSurrogate来判断，对于辅助平面内的代理对，做到整个移除或保留即可，isHighSurrogate方法的源码如下：
```java
    public static final char MIN_HIGH_SURROGATE = '\uD800';
	public static final char MAX_HIGH_SURROGATE = '\uDBFF';
	
    public static boolean isHighSurrogate(char ch) {
        return (MIN_HIGH_SURROGATE <= ch && MAX_HIGH_SURROGATE >= ch);
    }
```

本文部分参考自：https://blog.poxiao.me/p/unicode-character-encoding-conversion-in-cpp11/#UTF-16(16-bit_Unicode_Transformation_Format)