---
title: 关于一些基础的Java问题的解答（三）
tags: [java,基础知识]
categories: [java]
date: 2016-03-17 10:38:44
description: HashMap和ConcurrentHashMap的区别、TreeMap、HashMap、LinkedHashMap的区别、Collection包结构，与Collections的区别、try catch finally，try里有return，finally还执行么？、Excption与Error包结构，OOM遇到的情况
---
上一篇文章的传送门：[关于一些基础的Java问题的解答（二）](/2016/03/16/关于一些基础的Java问题的解答（二）/)

# HashMap和ConcurrentHashMap的区别

从JDK1.2起，就有了HashMap，正如上一个问题所提到的，HashMap与HashTable不同，不是线程安全的，因此多线程操作时需要格外小心。
在JDK1.5中，伟大的Doug Lea给我们带来了concurrent包，从此我们有线程安全的ConcurrentHashMap用了。
那么ConcurrentHashMap是怎么实现线程安全的呢？肯定不可能是每个方法都加上synchronized关键字，否则就和HashTable一样了，我们来看看ConcurrentHashMap的put方法：
```java
	public V put(K key, V value) {  
        return putVal(key, value, false);  
    }  
```
调用了putVal方法，我们继续看看putVal方法的源码。
```java
	final V putVal(K key, V value, boolean onlyIfAbsent) {  
        if (key == null || value == null) throw new NullPointerException();  
        int hash = spread(key.hashCode());  
        int binCount = 0;  
        for (Node<K,V>[] tab = table;;) {  
            Node<K,V> f; int n, i, fh;  
            if (tab == null || (n = tab.length) == 0)  
                tab = initTable();  
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {  
                if (casTabAt(tab, i, null,  
                             new Node<K,V>(hash, key, value, null)))  
                    break;                   // no lock when adding to empty bin  
            }  
            else if ((fh = f.hash) == MOVED)  
                tab = helpTransfer(tab, f);  
            else {  
                V oldVal = null;  
                synchronized (f) {  
                    if (tabAt(tab, i) == f) {  
                        if (fh >= 0) {  
                            binCount = 1;  
                            for (Node<K,V> e = f;; ++binCount) {  
                                K ek;  
                                if (e.hash == hash &&  
                                    ((ek = e.key) == key ||  
                                     (ek != null && key.equals(ek)))) {  
                                    oldVal = e.val;  
                                    if (!onlyIfAbsent)  
                                        e.val = value;  
                                    break;  
                                }  
                                Node<K,V> pred = e;  
                                if ((e = e.next) == null) {  
                                    pred.next = new Node<K,V>(hash, key,  
                                                              value, null);  
                                    break;  
                                }  
                            }  
                        }  
                        else if (f instanceof TreeBin) {  
                            Node<K,V> p;  
                            binCount = 2;  
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,  
                                                           value)) != null) {  
                                oldVal = p.val;  
                                if (!onlyIfAbsent)  
                                    p.val = value;  
                            }  
                        }  
                    }  
                }  
                if (binCount != 0) {  
                    if (binCount >= TREEIFY_THRESHOLD)  
                        treeifyBin(tab, i);  
                    if (oldVal != null)  
                        return oldVal;  
                    break;  
                }  
            }  
        }  
        addCount(1L, binCount);  
        return null;  
    }  
```
来看一些关键的代码：在第2行，方法检测了key和value是否为空，如果为空则抛出NullPointerException（看来线程安全的key和value都必须非空，和HashTable一样）。然后在第9行构造了一个Node对象，这个对象代表的是键值对节点（内部有next指针指向下一个节点）的实体。在ConcurrentHashMap内部维护了一个Node对象的数组，它大小是2的指数，且是volatile的具有原子可见性，数组索引是key经过哈希函数得出的哈希值：
```java
	/** 
     * The array of bins. Lazily initialized upon first insertion. 
     * Size is always a power of two. Accessed directly by iterators. 
     */  
    transient volatile Node<K,V>[] table;  
```
看回putVal方法，从第12行的注释我们可以看出，如果是往一个空的索引位置放入一个新的Node节点，则不需要加锁。再看到方法第18行，我们发现这里有个临界区，此时处理的是往一个已有节点的索引位置加入新的节点情况，那么在链表完成之前很明显我们不应该让其他新节点干扰我们的工作，因此此处为索引头的Node对象加了锁，但此时别的索引位置是不加锁的。
看完了put方法，我们再来看看get方法：
```java
	public V get(Object key) {  
        Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;  
        int h = spread(key.hashCode());  
        if ((tab = table) != null && (n = tab.length) > 0 &&  
            (e = tabAt(tab, (n - 1) & h)) != null) {  
            if ((eh = e.hash) == h) {  
                if ((ek = e.key) == key || (ek != null && key.equals(ek)))  
                    return e.val;  
            }  
            else if (eh < 0)  
                return (p = e.find(h, key)) != null ? p.val : null;  
            while ((e = e.next) != null) {  
                if (e.hash == h &&  
                    ((ek = e.key) == key || (ek != null && key.equals(ek))))  
                    return e.val;  
            }  
        }  
        return null;  
    }  
```
可以看到由于volatile关键字保证了原子可见性，get方法是完全没有加锁的。
对于remove方法和put方法类似，都是为要操作的索引头Node对象加锁构造临界区，此处不再贴出代码赘述。

综上所述，HashMap和ConcurrentHashMap区别如下：
1. HashMap允许key和value为空值，ConcurrentHashMap不允许
2. HashMap的put和remove不加锁，不是线程安全的，而ConcurrentHashMap加锁，是线程安全的
3. 两者的get方法都没有加锁，但HashMap的Node数组不具有volatile关键字

补充：本人的JDK版本为1.7，网上找的大部分资料都介绍ConcurrentHashMap使用了分段锁，但从源码来看这种方法在1.7貌似已经弃用了（put方法不一样了）：
```java
	/** 
     * Stripped-down version of helper class used in previous version, 
     * declared for the sake of serialization compatibility 
     */  
    static class Segment<K,V> extends ReentrantLock implements Serializable {  
        private static final long serialVersionUID = 2249069246763182397L;  
        final float loadFactor;  
        Segment(float lf) { this.loadFactor = lf; }  
    }  
```

# TreeMap、HashMap、LinkedHashMap的区别

HashMap就不说了，一个最常用的Map，前面几个问题都提及了。

## LinkedHashMap

LinkedHashMap是HashMap的子类，其类的声明如下：
```java
	public class LinkedHashMap<K,V>  
    extends HashMap<K,V>  
    implements Map<K,V>  
```
其与HashMap最大的不同在于内部维护了一个用于遍历的双向链表，遍历的时候能够保持元素插入的顺序：
```java
	/** 
     * The head (eldest) of the doubly linked list. 
     */  
    transient LinkedHashMap.Entry<K,V> head;  
  
    /** 
     * The tail (youngest) of the doubly linked list. 
     */  
    transient LinkedHashMap.Entry<K,V> tail;  
```

## TreeMap

TreeMap内部使用红黑树实现，因此插入TreeMap内部的元素遍历时是有序的：
```java
	/** 
     * The comparator used to maintain order in this tree map, or 
     * null if it uses the natural ordering of its keys. 
     * 
     * @serial 
     */  
    private final Comparator<? super K> comparator;  
  
    private transient Entry<K,V> root;  
```

# Collection包结构，与Collections的区别

Collection即Java中的容器类，其包结构如下所示：
![Java容器包结构图](1.png)

## Collection

可以看到Java容器中有Collection和Map两种类型，Collection表示元素的集合，Map表示键值对映射的集合，Map中也包含了Collection（key的集合，value的集合以及键值对的集合），而Collection又可以返回Iterator（迭代器）用于容器的遍历和访问。Collection下又细分为多种特点不同的容器，主要有：
1. 按元素插入顺序存储的List
2. 不允许元素重复的Set
3. 先进先出的Queue
4. 后进先出的Stack

每种容器又有对应的更细节的实现。一般我们创造新容器时不需要继承Collection类，继承Collection子类下的容器抽象类即可。

## Collections

除了Collection类以外，可以看到图的右下角还有两个工具类，分别是Collections和Arrays。两个类的构造方法均为private，即不允许新建该类的实例，同时两个类包含了大量的静态方法用于处理我们的存储结构（排序，二分查找，填充等操作）。其中，Collections用于处理Collection容器类的存储结构，而Arrays则用于处理基本类型组成的数组。

# try catch finally，try里有return，finally还执行么？

执行，无论发生啥情况，try后面的finally中的代码块必定会执行，示例：
```java
	public class Test {  
      
    public static void main(String[] args) {  
        try {  
            System.out.println("try");  
            return;  
        } catch (Exception e) {  
              
        } finally {  
            System.out.println("finally");  
        }  
    }  
      
}  
```
结果输出为：
```
try
finally
```

# Excption与Error包结构，OOM遇到的情况

Java中有关于异常类的结构图如下：
![Java异常类结构图](2.jpg)

在Java中，异常的根类是java.lang.Throwable类，而根类又分为两大类：Error和Exception：
- Error是无法处理的异常，比如OutOfMemoryError，一般发生这种异常，JVM会选择终止程序。因此我们编写程序时不需要关心这类异常。
- Exception，也就是我们经常见到的一些异常情况，比如NullPointerException、IndexOutOfBoundsException，这些异常是我们可以处理的异常。

其中，Exception类又细分为checked exception和unchecked exception（也称RuntimeException运行时异常）
对于unchecked exception（非检查异常），也称运行时异常（RuntimeException），比如常见的NullPointerException、IndexOutOfBoundsException，java编译器不要求必须进行异常捕获处理或者抛出声明，由程序员自行决定。（可以处理，也可以不处理）
对于checked exception（检查异常），也称非运行时异常（运行时异常以外的异常就是非运行时异常），java编译器强制程序员必须进行捕获处理，比如常见的IOExeption和SQLException。对于非运行时异常如果不进行捕获或者抛出声明处理，编译都不会通过。

## 关于OOM

OOM即out of memory，内存溢出，与JVM的运行时内存有关，当JVM内存不够时就会发生OOM，主要分一下几种情况：
1. 堆溢出：堆是JVM存放对象实例的地方，如果我们产生的对象过多，JVM又没有及时的GC，就会突破最大堆容量限制从而发生OOM
2. 操作栈或本地方法栈溢出：如果线程在拓展栈时无法申请到足够的内存，也会发生OOM，一般而言是递归出现了死循环
3. 方法区溢出：方法区存储了JVM中的常量，静态变量和类信息等信息，一个类如果要被垃圾收集器回收，判定条件是很苛刻的，因此在经常动态生成加载大量Class也可能发生OOM