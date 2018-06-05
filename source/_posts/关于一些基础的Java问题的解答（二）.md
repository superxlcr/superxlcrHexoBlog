---
title: 关于一些基础的Java问题的解答（二）
tags: [java,基础知识]
categories: [java]
date: 2016-03-16 15:37:45
description:  Hashcode的作用、ArrayList、LinkedList、Vector的区别、String、StringBuffer与StringBuilder的区别、Map、Set、List、Queue、Stack的特点与用法、HashMap和HashTable的区别
---
上一篇文章的传送门：[关于一些基础的Java问题的解答（一）](/2016/03/15/关于一些基础的Java问题的解答（一）/)

# Hashcode的作用

官方对于hashCode的解释如下：
- Whenever it is invoked on the same object more than once during an execution of a Java application, the hashCode method must consistently return the same integer, provided no information used in equals comparisons on the object is modified. This integer need not remain consistent from one execution of an application to another execution of the same application.
- If two objects are equal according to the equals(Object) method, then calling the hashCode method on each of the two objects must produce the same integer result.
- It is not required that if two objects are unequal according to the equals(java.lang.Object) method, then calling the hashCode method on each of the two objects must produce distinct integer results. However, the programmer should be aware that producing distinct integer results for unequal objects may improve the performance of hash tables.

以上这段官方文档的定义，我们可以抽出成以下几个关键点：
1. hashCode的存在主要是用于查找的快捷性，如Hashtable，HashMap等，hashCode是用来在散列存储结构中确定对象的存储地址的
2. 两个equals（相同）的对象返回的hashCode是相同的
3. 如果对象的equals方法被重写，那么对象的hashCode也尽量重写，避免违反第2点
4. 两个不同的对象hashCode不一定不同，但如果他们的hashCode不同哈希表的性能会更好（减少冲突）

# ArrayList、LinkedList、Vector的区别

先说说相同点：三者都实现了List接口

再来说说不同的地方：
1. LinkedList额外实现了Queue接口
2. ArrayList和Vector底层是由数组实现，LinkedList底层是由双向链表实现
3. 根据第2点，ArrayList和Vector具有更好的随机索引读取性能，LinkedList具有更好的插入删除元素性能
4. ArrayList和LinkedList不是线程安全的，Vector是线程安全的（关键方法都加了synchronized关键字），因此Vector在性能上不如前两者

# String、StringBuffer与StringBuilder的区别

## String

String即字符串，底层由char数组实现，代码如下：
```java
	/** The value is used for character storage. */  
    private final char value[];  
```
官方对String解释如下：
Strings are constant; their values cannot be changed after they are created. 
综上我们可以得知String是不变的常量，String类中每一个看起来会修改String的方法，实际上都创建了一个全新的String对象。当我们为一个String变量赋值的时候，其实是新创建了一个String对象把地址赋给他。
除此以外，值得注意的是String是一个final类，这意味着它不能被继承。

## StringBuffer和StringBuilder

由于String是常量，当我们处理长度变化的String时效率低下，因此官方提供了StringBuffer和StringBuilder两个类。
两个类有共同的父类：AbstractStringBuilder，方法也差不多，底层也是由char数组存储数据的，但是StringBuffer是线程安全的（关键方法都加了synchronized关键字），而StringBuilder不是线程安全的，因此StringBuilder处理字符串的效率比较高。

# Map、Set、List、Queue、Stack的特点与用法

先上Java容器分类图（虚线框表示抽象类）：

![Java容器分类图](1.png)

## Map

Map即映射表，里面保存的是一组成对的"键值对"对象，一个映射不能包含重复的键，每个键最多只能映射到一个值，我们可以通过"键"找到该键对应的"值"。

常用的Map的方法有：
1. clear：移除Map中的所有元素
2. containsKey：判断Map中是否含有指定的“键”
3. containsValue：判断Map中是否含有指定的“值”
4. entrySet：返回Map中键值对的集合
5. get：通过“键”获取指定的“值”
6. isEmpty：是否为空Map
7. keySet：返回Map中“键”的集合
8. put：加入新的键值对，返回该键的旧值
9. remove：移除指定“键”的键值对，返回该键的值
10. size：Map含有的键值对数目
11. values：返回值的容器视图

常用的Map的实现类有：
1. HashMap：哈希映射表，最常用的Map，使用了hashCode（散列码）优化的Map结构，不保证遍历顺序（例：先存入a后存入b，但遍历时可能先出现b再出现a）
2. LinkedHashMap：HashMap的子类，但内部维护了一个双向链表，保证遍历顺序是插入顺序或基于LRU（最近最少使用）算法
3. WeakHashMap：使用弱引用实现的HashMap，当其中的某些键值对不再被使用时会被自动GC掉
4. HashTable：与HashMap大致上类似，但是是线程安全的
5. IdentityHashMap：使用==代替equals对“键”进行比较的HashMap
6. TreeMap：基于红黑树实现的HashMap，查看“键”或“键值对”时，它们会被排序，另外可以使用subMap方法返回子树

## Set

Set即集合，里面保存的是一堆不可重复的元素，使用equals方法来确保对象的唯一性。

常用的Set的方法有：
1. add：为Set加入新的元素，返回是否加入成功（存在则失败，反之则成功）
2. clear：清除所有元素
3. contains：判断是否含有某元素
4. isEmpty：判断是否为空Set
5. iterator：返回迭代器
6. remove：移除某元素
7. size：Set含有元素的数目

常用Set的实现类有：
1. HashSet：为快速查找而设计的Set，存入的元素必须定义hashCode
2. LinkedHashSet：具有HashSet的查询速度，且内部有链表维护元素顺序的Set
3. TreeSet：保持排序的Set，底层为树结构

## List

List即列表，与数组类似，按照元素插入的顺序保存元素。

常用的List的方法有：
1. add：在最后或特定位置加入新的元素
2. clear：清除所有元素
3. contains：是否含有指定元素
4. get：获取特定索引元素
5. indexOf：返回特定元素第一次出现的索引，没有则返回-1
6. isEmpty：判断是否为空
7. iterator：返回迭代器
8. lastIndexOf：返回特定元素最后一次出现的索引，没有则返回-1
9. remove：移除特定元素
10. size：返回List大小
11. subList：根据传入的两个索引返回子列表

常用的List实现类有：
1. ArrayList：擅长随机访问的列表
2. LinkedList：擅长插入和删除操作的列表
3. Vector：与ArrayList类似，线程安全的列表

## Queue

Queue即队列，是一个FIFO（先进先出）的容器。

常用的Queue方法有：
1. add/offer：往队列中加入新元素
2. peek：返回队首元素
3. poll：返回并移除队首元素
4. isEmpty：队列是否为空

常用的Queue的实现类有：
1. LinkedList：即普通的队列
2. PriorityQueue：一个由优先级堆实现的队列，队列中的元素是有序的

## Stack

Stack即栈，是一个LIFO（后进先出）的容器。

常用的Stack方法有：
1. push：往栈中添加新元素
2. peek：返回栈顶元素
3. pop：返回并移除栈顶元素
4. isEmpty：栈是否为空

Stack的实现类即Stack。

# HashMap和HashTable的区别

HashMap和HashTable都实现了Map接口，他们功能也相当类似，两者的主要区别如下：
1. HashMap继承自AbstractMap类，而HashTable继承自Dictionary类
2. HashMap不是线程安全的，HashTable是线程安全的（关键方法添加了synchronized关键字），因此HashTable效率相对较低
3. HashMap允许key和value为null，HashTable不允许key和value为null

HashMap和HashTable类声明：
```java
public class HashMap<K,V> extends AbstractMap<K,V>  
    implements Map<K,V>, Cloneable, Serializable  
```
```java
public class Hashtable<K,V>  
    extends Dictionary<K,V>  
    implements Map<K,V>, Cloneable, java.io.Serializable  
```

HashMap的put方法：

```java
	public V put(K key, V value) {  
        return putVal(hash(key), key, value, false, true);  
    }  
	final V putVal(int hash, K key, V value, boolean onlyIfAbsent,  
                   boolean evict) {  
        Node<K,V>[] tab; Node<K,V> p; int n, i;  
        if ((tab = table) == null || (n = tab.length) == 0)  
            n = (tab = resize()).length;  
        if ((p = tab[i = (n - 1) & hash]) == null)  
            tab[i] = newNode(hash, key, value, null);  
        else {  
            Node<K,V> e; K k;  
            if (p.hash == hash &&  
                ((k = p.key) == key || (key != null && key.equals(k))))  
                e = p;  
            else if (p instanceof TreeNode)  
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);  
            else {  
                for (int binCount = 0; ; ++binCount) {  
                    if ((e = p.next) == null) {  
                        p.next = newNode(hash, key, value, null);  
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st  
                            treeifyBin(tab, hash);  
                        break;  
                    }  
                    if (e.hash == hash &&  
                        ((k = e.key) == key || (key != null && key.equals(k))))  
                        break;  
                    p = e;  
                }  
            }  
            if (e != null) { // existing mapping for key  
                V oldValue = e.value;  
                if (!onlyIfAbsent || oldValue == null)  
                    e.value = value;  
                afterNodeAccess(e);  
                return oldValue;  
            }  
        }  
        ++modCount;  
        if (++size > threshold)  
            resize();  
        afterNodeInsertion(evict);  
        return null;  
    }  
```

从上面代码我们可以看出对key和value并没有不能为null的限制。
HashTable的put方法：

```java
	public synchronized V put(K key, V value) {  
        // Make sure the value is not null  
        if (value == null) {  
            throw new NullPointerException();  
        }  
  
        // Makes sure the key is not already in the hashtable.  
        Entry<?,?> tab[] = table;  
        int hash = key.hashCode();  
        int index = (hash & 0x7FFFFFFF) % tab.length;  
        @SuppressWarnings("unchecked")  
        Entry<K,V> entry = (Entry<K,V>)tab[index];  
        for(; entry != null ; entry = entry.next) {  
            if ((entry.hash == hash) && entry.key.equals(key)) {  
                V old = entry.value;  
                entry.value = value;  
                return old;  
            }  
        }  
  
        addEntry(hash, key, value, index);  
        return null;  
    }  
```

这是一个同步的方法，我们看到第3行会判断value是否为null，如果为null则抛出NullPointer的异常。在第9行调用了key的hashCode方法，如果key对象为null，则会抛出NullPointer的异常。因此，插入HashTable的key和value均不能为null。