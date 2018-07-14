---
title: 关于一些基础的Java问题的解答（九）
tags: [java,基础知识]
categories: [java]
date: 2017-08-10 17:07:49
description: Collections工具类的shuffle方法、Java 找不到或无法加载主类、Java中单件模式的实现方法、如何判断两个float是否相等、Java对象序列化
---
上一篇文章的传送门：[关于一些基础的Java问题的解答（八）](/2017/07/09/关于一些基础的Java问题的解答（八）/)

# Collections工具类的shuffle方法
Java中Collections类的shuffle()方法的作用是将List中的内容随机打乱顺序
其源码如下：
```java
public static void shuffle(List<?> list) {  
    if (r == null) {  
        r = new Random();  
    }  
    shuffle(list, r);  
}  
private static Random r;  
 
 
public static void shuffle(List<?> list, Random rnd) {  
    int size = list.size();  
    if (size < SHUFFLE_THRESHOLD || list instanceof RandomAccess) {  
        for (int i=size; i>1; i--)  
            swap(list, i-1, rnd.nextInt(i));  
    } else {  
        Object arr[] = list.toArray();  
 
        // Shuffle array  
        for (int i=size; i>1; i--)  
            swap(arr, i-1, rnd.nextInt(i));  
  
        // Dump array back into list  
        ListIterator it = list.listIterator();  
        for (int i=0; i<arr.length; i++) {  
            it.next();  
            it.set(arr[i]);  
        }  
    }  
}
```

shuffle方法使用了Random类，通过把list其中的元素随机交换size次，打乱list元素的顺序
# Java 找不到或无法加载主类
如果在你没有打错类名的前提下，可能是Java解释器搜索的目录**没有包含当前目录**，一般而言Java解释器会搜索环境变量下的ClassPath目录
因此我们可以为我们的ClassPath环境变量添加：.;也可以调用java解释器的时候带上参数：-cp .

# Java中单件模式的实现方法
在Java中，单件模式的实现方法有以下五种：懒汉、饿汉、静态内部类、枚举以及双重校验锁
懒汉的实现方法如下（只有在使用的时候才进行初始化）：
```java
public class SingleInstance {
	
	public static SingleInstance instance = null;
	
	public static synchronized SingleInstance getInstance() {
		if (instance == null) {
			instance = new SingleInstance();
		}
		return instance;
	}
	
	private SingleInstance() {}
	
}
```

饿汉的实现方法如下（提前进行初始化）：
```java
public class SingleInstance {
	
	public static SingleInstance instance = new SingleInstance();
	
	public static SingleInstance getInstance() {
		return instance;
	}
	
	private SingleInstance() {}
	
}
```

静态内部类的实现方法如下（只有在初次使用时才会加载内部静态类，实例化单件实例）：
```java
public class SingleInstance {
	
	public static SingleInstance getInstance() {
		return InnerStaticClass.instance;
	}
	
	private SingleInstance() {}
	
	private static class InnerStaticClass {
		private static SingleInstance instance = new SingleInstance();
	}
}
```

枚举的实现方法如下：
```java
public enum SingleInstance {
	
	INSTANCE;

	private SingleInstance() {}
	
}
```

双重校验锁实现方法：
```java
public class SingleInstance {
	
	public static SingleInstance instance;
	
	public static SingleInstance getInstance() {
		if (instance == null) {
			synchronized (SingleInstance.class) {
				if (instance == null) {
					instance = new SingleInstance();
				}
			}
		}
		return instance;
	}
	
	private SingleInstance() {}
	
}
```


# 如何判断两个float是否相等
首先两个float是不能直接使用==来判断是否相等的，一般而言，我们有两种写法：较严格的：
```java
Math.abs(a-b) <= 0
```
较宽松的：
```java
Math.abs(a-b) <= 0.00000001
```


# Java对象序列化
在Java中，序列化对象是一种把对象转换为字节码的方法，我们可以通过ObjectOutputStream以及ObjectInputStream来实现对象的序列化以及反序列化
当我们需要把某个对象进行序列化时，我们需要为对象implement Serializable接口来“标记”可以序列化该对象
序列化以及反序列化能为我们保存对象的成员变量，当我们不希望某个成员变量被序列化时，我们可以使用transient关键字来标记它
反序列化时，系统通过 static final long serialVersionUID 来判断字节码与class是否一致，如果不一致则会抛出异常
一般而言，serialVersionUID如果没有显示声明，系统会根据class的成员变量为其设置一个值（这将会导致class成员变量修改时我们的反序列化出现异常，不过我们可以通过自行显示设置 serialVersionUID来解决这个问题）
一般而言，系统会自行处理对象序列化以及反序列化的过程，不过我们也可以通过实现以下方法来对该过程进行控制（系统序列化之前会通过反射判断是否有对应方法，如果有则把流程交给其处理）：
```java
 private void writeObject(java.io.ObjectOutputStream out) throws IOException
 private void readObject(java.io.ObjectInputStream in) throws IOException, ClassNotFoundException;
 private void readObjectNoData() throws ObjectStreamException;
```

 