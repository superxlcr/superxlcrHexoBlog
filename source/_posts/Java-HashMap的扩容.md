---
title: Java HashMap的扩容
tags: [java,基础知识]
categories: [java]
date: 2017-05-30 18:29:18
description: HashMap中的变量、HashMap的构造函数、何时进行扩容、resize扩容
---
最近博主参加面试，发现自己对于Java的HashMap的扩容过程理解不足，故最近在此进行总结。


首先说明博主德Java为1.8版本


# HashMap中的变量
首先要了解HashMap的扩容过程，我们就得了解一些HashMap中的变量：

- Node&lt;K,V&gt;：链表节点，包含了key、value、hash、next指针四个元素
- table：Node&lt;K,V&gt;类型的数组，里面的元素是链表，用于存放HashMap元素的实体
- size：记录了放入HashMap的元素个数
- loadFactor：负载因子
- threshold：阈值，决定了HashMap何时扩容，以及扩容后的大小，一般等于table大小乘以loadFactor



# HashMap的构造函数
HashMap的构造函数主要有四个，代码如下：

```java
    public HashMap(int initialCapacity, float loadFactor) {
        ...
        this.loadFactor = loadFactor;
        this.threshold = tableSizeFor(initialCapacity);
    }

    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }

    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }

    public HashMap(Map<? extends K, ? extends V> m) {
        this.loadFactor = DEFAULT_LOAD_FACTOR;
        putMapEntries(m, false);
    }
```


其中主要有两种形式：

- 直接拷贝别的HashMap的形式，在此不作讨论
- 定义初始容量大小（table数组的大小，缺省值为16），定义负载因子（缺省值为0.75）的形式

值得注意的是，当我们自定义HashMap初始容量大小时，构造函数并非直接把我们定义的数值当做HashMap容量大小，而是把该数值当做参数调用方法tableSizeFor，然后把返回值作为HashMap的初始容量大小：


```java
    /**
     * Returns a power of two size for the given target capacity.
     */
    static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
```


该方法会返回一个大于等于当前参数的2的倍数，因此HashMap中的table数组的容量大小总是2的倍数。



# 何时进行扩容？
HashMap使用的是懒加载，构造完HashMap对象后，只要不进行put 方法插入元素之前，HashMap并不会去初始化或者扩容table：

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
            ...
        }
        ++modCount;
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```


在putVal方法第8、9行我们可以看到，当首次调用put方法时，HashMap会发现table为空然后调用resize方法进行初始化
在putVal方法第16、17行我们可以看到，当添加完元素后，如果HashMap发现size（元素总数）大于threshold（阈值），则会调用resize方法进行扩容


在这里值得注意的是，在putVal方法第10行我们可以看到，插入元素的hash值是一个32位的int值，而实际当前元素插入table的索引的值为 ：

```java
（table.size - 1）& hash
```


又由于table的大小一直是2的倍数，2的N次方，因此当前元素插入table的索引的值为其hash值的后N位组成的值


# resize扩容

```java
    final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
        else {               // zero initial threshold signifies using defaults
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }
```


从第15 ～ 20行可以看到，若threshold（阈值）不为空，table的首次初始化大小为阈值，否则初始化为缺省值大小16


当table需要扩容时，从第11 ~ 13行可以看到，扩容后的table大小变为原来的两倍，接下来就是进行扩容后table的调整：
假设扩容前的table大小为2的N次方，有上述put方法解析可知，元素的table索引为其hash值的后N位确定
那么扩容后的table大小即为2的N+1次方，则其中元素的table索引为其hash值的后N+1位确定，比原来多了一位
因此，table中的元素只有两种情况：

1. 元素hash值第N+1位为0：不需要进行位置调整
2. 元素hash值第N+1位为1：调整至原索引的两倍位置

在resize方法中，第45行的判断即用于确定元素hashi值第N+1位是否为0：


- 若为0，则使用loHead与loTail，将元素移至新table的原索引处
- 若不为0，则使用hiHead与hiHead，将元素移至新table的两倍索引处

扩容或初始化完成后，resize方法返回新的table

