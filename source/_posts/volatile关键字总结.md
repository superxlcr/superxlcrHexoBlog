---
title: volatile关键字总结
tags: [java,基础知识]
categories: [java]
date: 2020-03-14 22:00:04
description: volatile关键字特性、volatile原理、volatile应用
---

# volatile关键字特性

volatile关键字有如下三个特性：
- 可见性
- 原子性
- 有序性

## 可见性

是指线程之间的可见性，一个线程修改的状态对另一个线程是可见的
也就是一个线程修改的结果，另一个线程马上就能看到
在Java中，我们可以通过volatile、synchronized和final实现可见性

## 原子性

原子是世界上的最小单位，具有不可分割的特性
volatile能保证对变量单次读/写的原子性
比如说，因为long和double两种数据类型的操作可分为高32位和低32位两部分，因此普通的long或double类型读/写可能不是原子的
所以，我们可以将共享的long和double变量设置为volatile类型，这样能保证任何情况下对long和double的单次读/写操作都具有原子性

## 有序性

Java 语言提供了volatile和synchronized两个关键字来保证线程之间操作的有序性
volatile是因为其本身包含“禁止指令重排序”的语义
synchronized 是由“一个变量在同一个时刻只允许一条线程对其进行 lock 操作”这条规则获得的，此规则决定了持有同一个对象锁的两个同步块只能串行执行

# volatile原理

Java语言提供了一种稍弱的同步机制，即volatile变量，用来确保将变量的更新操作通知到其他线程
当把变量声明为volatile类型后，编译器与运行时都会注意到这个变量是共享的，因此不会将该变量上的操作与其他内存操作一起重排序（**有序性**）
volatile变量不会被缓存在寄存器或者对其他处理器不可见的地方，因此在读取volatile类型的变量时总会返回最新写入的值（**可见性**）

在访问volatile变量时不会执行加锁操作，因此也就不会使执行线程阻塞，因此volatile变量是一种比sychronized关键字更轻量级的同步机制

![java变量读取示意图](1.png)

当对非 volatile 变量进行读写的时候，每个线程先从内存拷贝变量到CPU缓存中
如果计算机有多个CPU，每个线程可能在不同的CPU上被处理，这意味着每个线程可以拷贝到不同的 CPU cache 中
而声明变量是 volatile 的，JVM 保证了每次读变量都从内存中读，跳过 CPU cache 这一步

ps: volatile 的读性能消耗与普通变量几乎相同，但是写操作稍慢，因为它需要在本地代码中插入许多内存屏障指令来保证处理器不发生乱序执行，不过性能还是比加锁好的

# volatile应用

volatile比较经典的一个应用场景是在java实现单例模式时，双重校验锁的场景：
```java
class Singleton {

    private volatile static Singleton singleton;

    private Singleton(){}       
    public static Singleton getInstance(){       
        if(singleton == null){                  
            synchronized(Singleton.class){      
                if(singleton == null){          
                    singleton = new Singleton(); // 可能出现指令重排序的情况
                }
            }
        } 
        return singleton;           
    }
}
```
在上面代码注释的那一行，可能会出现指令重排序的问题
这里我们先要理解new Singleton()做了什么

new一个对象有如下几个步骤：
1. 看class对象是否加载，如果没有就先加载class对象
2. 分配内存空间，初始化实例
3. 调用构造函数
4. 返回地址给引用

而cpu为了优化程序，可能会进行指令重排序，打乱这3，4这几个步骤，导致实例内存还没分配，就被使用了

因此如果此处的单例变量我们不加上volatile关键字，可能会有如下的场景：
用线程A和线程B举例，假设线程A执行到new Singleton()，开始初始化实例对象，由于存在指令重排序，这次new操作，先把引用赋值了，还没有执行构造函数
这时时间片结束了，切换到线程B执行，线程B调用getInstance()方法，发现引用不等于null，就直接返回引用地址了，然后线程B执行了一些操作，就可能导致线程B使用了还没有被初始化的变量