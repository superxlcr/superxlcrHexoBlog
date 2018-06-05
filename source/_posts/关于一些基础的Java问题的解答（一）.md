---
title: 关于一些基础的Java问题的解答（一）
tags: [java,基础知识]
categories: [java]
date: 2016-03-15 16:13:53
description: 九种基本数据类型的包装类及大小、switch的参数、equals与==的区别、Object有哪些公用方法、Java的四种引用，强弱软虚，用到的场景
---
学习一门语言基础是非常重要的，因此本文总结了一些常见的Java基础问题的解答，希望可以帮到大家。

# 九种基本数据类型的包装类及大小。

9种基本数据类型：

| 基本类型 | 包装类型 | 大小 |
| - | - | - |
| boolean | Boolean | - |
| byte | Byte | 8bit |
| short | Short | 16bit |
| int | Integer | 32bit |
| long | Long | 64bit |
| float | Float | 32bit |
| double | Double | 64bit |
| char | Character | 16bit |
| void | Void | - |

上面有两个比较值得注意的地方：
一个是boolean的大小并非是确定的，根据网上找来的JVM规范：
```
Instead, expressions in the Java programming language that operate on boolean values are compiled to use values of the Java virtual machine int data type.  
Where Java programming language boolean values are mapped by compilers to values of Java virtual machine type int, the compilers must use the same encoding. Java virtual machine type int, whose values are 32-bit signed two's-complement integers。
Arrays of type boolean are accessed and modified using the byte array instructions  
In Sun's JDK releases 1.0 and 1.1, and the Java 2 SDK, Standard Edition, v1.2, boolean arrays in the Java programming language are encoded as Java virtual machine byte arrays, using 8 bits per boolean element.
```
从上文我们可以得知，boolean变量单独声明的时候被当做int变量处理，其大小为32bit，当boolean变量声明数组的时候，每个boolean当做byte变量处理，大小为8bit。

另一个是void也是Java的基本类型之一，其对应的包装类是不可实例化的，如下所示：
```java
package java.lang;  
  
/** 
 * The {@code Void} class is an uninstantiable placeholder class to hold a 
 * reference to the {@code Class} object representing the Java keyword 
 * void. 
 * 
 * @author  unascribed 
 * @since   JDK1.1 
 */  
public final  
class Void {  
  
    /** 
     * The {@code Class} object representing the pseudo-type corresponding to 
     * the keyword {@code void}. 
     */  
    @SuppressWarnings("unchecked")  
    public static final Class<Void> TYPE = (Class<Void>) Class.getPrimitiveClass("void");  
  
    /* 
     * The Void class cannot be instantiated. 
     */  
    private Void() {}  
}  
```

# switch的参数

在Java中，switch后面的括号里放的参数类型只能是int类型，虽然说放入char，byte，short类型也不会报错，但其实是因为char，byte和short类型可以自己转化（宽化）为int类型，实际上最后操作的还是int类型。

原理：在Java的9种基本类型中，boolean和void类型不能进行转换。大小较小的类型向大小较大的类型转换叫宽化（如char转int），宽化会自动在变量前面补零且是安全的，因此可以自动转换。大小较大的类型向大小较小的类型转化叫窄化（如double转int），窄化不能自动转换，必须使用强制类型转换，如：（int）double。

补充：除了int类型外还支持枚举类型。另外在jdk1.7中有了新的特性，支持String类型。

# equals与==的区别

很经典的一个问题了。首先==在Java中，对于基本类型比较的是他们的值是否相等，对于引用类型则比较两个对象是否地址相同，是否为同一引用。equals在Object中的默认实现和==没有区别：
```java
	public boolean equals(Object obj) {  
        return (this == obj);  
    }  
```
值的注意的是用的较多的String类重写了equals方法：
```java
	public boolean equals(Object anObject) {  
        if (this == anObject) {  
            return true;  
        }  
        if (anObject instanceof String) {  
            String anotherString = (String)anObject;  
            int n = value.length;  
            if (n == anotherString.value.length) {  
                char v1[] = value;  
                char v2[] = anotherString.value;  
                int i = 0;  
                while (n-- != 0) {  
                    if (v1[i] != v2[i])  
                        return false;  
                    i++;  
                }  
                return true;  
            }  
        }  
        return false;  
    }  
```
从上面方法我们可以看出，首先String比较传入的Object对象是否和当前String为同一引用，是则返回true。如果不是，则判断传入的Object对象是否String对象及其子类实例，如果不是则返回false。再然后如果传入的参数是String的实例，则比较两个String的内容是否完全相同。

# Object有哪些公用方法

![Object公用方法截图](1.jpg)

Object中的public方法如图所示，简单介绍下每个方法：
1. Object()，默认构造函数，略
2. getClass()，native和final方法，返回此对象的运行时类
3. hashCode()，native方法，返回此对象的哈希码数值
4. equals(Object)，详见上一个问题
5. toString()，返回一个代表该对象的字符串，此处返回：“类名@16进制哈希码”
6. notify()，native和final方法，唤醒在此对象监视器上等待的单个线程
7. notifyAll()，native和final方法，唤醒在此对象监视器等待上的所有线程
8. wait()，该方法有3个重载，作用是让当前线程处于等待状态，直到其他线程调用此对象的 notify() 方法或 notifyAll() 方法，或超过指定时间，或其他线程中断当前线程。

# Java的四种引用，强弱软虚，用到的场景

在Java中，对象的引用和JVM的GC有着密切的联系，对于对象我们有4中不同的引用：
1. 强引用：我们平时普通创建对象的方法就是强引用，普通系统99%以上都是强引用。如果一个对象是强引用创建的，则需要手动置null，JVM的GC才能回收其内存，否则即使是报内存不足，也不会清理具有强引用的对象。
2. 软引用：通过SoftReference<T>方法创建为软引用，一般JVM的GC不会清理软引用，但会在发生OOM（out of memory）时清理软引用。一般软引用可以用来实现内存敏感的高速缓存。
3. 弱引用：通过WeakReference<T>方法创建即弱引用，弱引用的生命周期比软引用短很多，如果一个对象只具有弱引用，那么他会在JVM GC时被清理掉。JDK有使用弱引用实现的WeakHashMap，他会在GC时回收掉不怎么使用的键值对。
4. 虚引用：通过PhantomReference<T>方法创建即为虚引用，虚引用形同虚设，如果一个对象只具有虚引用，那么它就和没有任何引用一样，随时会被JVM当作垃圾进行回收。虚引用主要用来跟踪对象被垃圾回收的活动。
