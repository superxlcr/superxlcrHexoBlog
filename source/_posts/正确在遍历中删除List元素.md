---
title: 正确在遍历中删除List元素
tags: [java,基础知识]
categories: [java]
date: 2016-05-30 00:45:35
description: 使用普通for循环遍历、使用增强型for循环遍历、使用iterator遍历
---
最近在写代码的时候遇到了遍历时删除List元素的问题，在此写一篇博客记录一下。
一般而言，遍历List元素有以下三种方式：

- 使用普通for循环遍历
- 使用增强型for循环遍历
- 使用iterator遍历

# 使用普通for循环遍历
代码如下：

```java
public class Main {
	public static void main(String[] args) throws Exception {
		List<Integer> list = new ArrayList<>();
		for (int i = 0; i < 5; i++)
			list.add(i);
		// list {0, 1, 2, 3, 4}
		for (int i = 0; i < list.size(); i++) {
			// index and number
			System.out.print(i + " " + list.get(i));
			if (list.get(i) % 2 == 0) {
				list.remove(list.get(i));
				System.out.print(" delete");
				i--; // 索引改变!
			}
			System.out.println();
		}
	}
}

```


结果如下：
![普通for循环遍历_pic1](1.png)


可以看到遍历删除偶数的结果是成功的，但是这种方法由于删除的时候会改变list的index索引和size大小，可能会在遍历时导致一些访问越界的问题，因此不是特别推荐。


# 使用增强型for循环遍历
```java
public class Main {
	public static void main(String[] args) throws Exception {
		List<Integer> list = new ArrayList<>();
		for (int i = 0; i < 5; i++)
			list.add(i);
		// list {0, 1, 2, 3, 4}
		for (Integer num : list) {
			// index and number
			System.out.print(num);
			if (num % 2 == 0) {
				list.remove(num);
				System.out.print(" delete");
			}
			System.out.println();
		}
	}
}

```

结果如下：
![增强for循环遍历_pic2](2.png)



可以看到删除第一个元素时是没有问题的，但删除后继续执行遍历过程的话就会抛出ConcurrentModificationException的异常。


# 使用iterator遍历
```java
public class Main {
	public static void main(String[] args) throws Exception {
		List<Integer> list = new ArrayList<>();
		for (int i = 0; i < 5; i++)
			list.add(i);
		// list {0, 1, 2, 3, 4}
		Iterator<Integer> it = list.iterator();
		while (it.hasNext()) {
			// index and number
			int num = it.next();
			System.out.print(num);
			if (num % 2 == 0) {
				it.remove();
				System.out.print(" delete");
			}
			System.out.println();
		}
	}
}

```

结果如下：
![iterator循环遍历_pic3](3.png)



可以看到顺利的执行了遍历并删除的操作，因此最推荐的做法是使用iterator执行遍历删除操作。


以上是关于非线程安全的ArrayList，如果是线程安全的CopyOnWriteArrayList呢？



# 使用普通for循环遍历



```java
public class Main {
	public static void main(String[] args) throws Exception {
		List<Integer> list = new CopyOnWriteArrayList<>();
		for (int i = 0; i < 5; i++)
			list.add(i);
		// list {0, 1, 2, 3, 4}
		for (int i = 0; i < list.size(); i++) {
			// index and number
			System.out.print(i + " " + list.get(i));
			if (list.get(i) % 2 == 0) {
				list.remove(list.get(i));
				System.out.print(" delete");
				i--; // 索引改变!
			}
			System.out.println();
		}
	}
}

```

结果如下：
![CopyOnWriteArrayList遍历删除_pic4](4.png)

可以看到遍历删除是成功的，但是这种方法由于删除的时候会改变list的index索引和size大小，可能会在遍历时导致一些访问越界的问题，因此不是特别推荐。




# 使用增强型for循环遍历
```java
public class Main {
	public static void main(String[] args) throws Exception {
		List<Integer> list = new CopyOnWriteArrayList<>();
		for (int i = 0; i < 5; i++)
			list.add(i);
		// list {0, 1, 2, 3, 4}
		for (Integer num : list) {
			// index and number
			System.out.print(num);
			if (num % 2 == 0) {
				list.remove(num);
				System.out.print(" delete");
			}
			System.out.println();
		}
	}
}

```

结果如下：
![CopyOnWriteArrayList增强for遍历删除_pic5](5.png)



可以看见与ArrayList遍历删除时情况不同，CopyOnWriteArrayList是允许使用增强型for进行循环遍历删除的。



# 使用iterator遍历
```java
public class Main {
	public static void main(String[] args) throws Exception {
		List<Integer> list = new CopyOnWriteArrayList<>();
		for (int i = 0; i < 5; i++)
			list.add(i);
		// list {0, 1, 2, 3, 4}
		Iterator<Integer> it = list.iterator();
		while (it.hasNext()) {
			// index and number
			int num = it.next();
			System.out.print(num);
			if (num % 2 == 0) {
				it.remove();
				System.out.print(" delete");
			}
			System.out.println();
		}
	}
}

```

结果如下：
![CopyOnWriteArrayList的iterator遍历删除_pic6](6.png)



与ArrayList不同，由于CopyOnWriteArrayList的iterator是对其List的一个“快照”，因此是不可改变的，所以无法使用iterator遍历删除。


综上所述，当使用ArrayList时，我们可以使用iterator实现遍历删除；而当我们使用CopyOnWriteArrayList时，我们直接使用增强型for循环遍历删除即可，此时使用iterator遍历删除反而会出现问题。



