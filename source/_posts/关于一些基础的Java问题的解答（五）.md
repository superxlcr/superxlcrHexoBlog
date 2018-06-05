---
title: 关于一些基础的Java问题的解答（五）
tags: [java,基础知识]
categories: [java]
date: 2016-03-19 10:15:48
description: 实现多线程的两种方法：Thread与Runable、线程同步的方法：sychronized、lock、reentrantLock等、锁的等级：对象锁、类锁、写出生产者消费者模式、ThreadLocal的设计理念与作用
---
上一篇文章的传送门：[关于一些基础的Java问题的解答（四）](/2016/03/18/关于一些基础的Java问题的解答（四）/)

# 实现多线程的两种方法：Thread与Runable

在Java中实现多线程编程有以下几个方法：

## 继承Thread类，重写run方法

```java
public class Test {  
      
    public static void main(String[] args) {  
        new MyThread().start();  
    }  
      
    private static class MyThread extends Thread {  
        @Override  
        public void run() {  
            System.out.println("run!");  
        }  
    }  
      
}  
```

## 实现Runnable接口，作为参数传入Thread构造函数

```java
public class Test {  
      
    public static void main(String[] args) {  
        new Thread(new Runnable() {  
              
            @Override  
            public void run() {  
                System.out.println("run!");  
                  
            }  
        }).start();  
    }  
      
}  
```

## 使用ExecutorService类

```java
import java.util.concurrent.ExecutorService;  
import java.util.concurrent.Executors;  
  
public class Test {  
      
    public static void main(String[] args) {  
        ExecutorService service = Executors.newCachedThreadPool();  
        service.execute(new Runnable() {  
              
            @Override  
            public void run() {  
                System.out.println("Run!");  
            }  
        });  
    }  
      
}  
```

补充：根据《阿里巴巴Java开发手册》，并不推荐使用Executors类中提供的线程池来开启线程，虽然这会比较方便，但是因为线程池参数不太合理的缘故，容易造成系统OOM

# 线程同步的方法：sychronized、lock、reentrantLock等

多线程编程时同步一直是一个非常重要的问题，很多时候我们由于同步问题程序失败的概率非常低，导致往往存在我们的代码缺陷，但他们看起来是正确的：
```java
public class Test {  
      
    private static int value = 0;  
      
    public static void main(String[] args) {  
        Test test = new Test();  
        // 创建两个线程  
        MyThread thread1 = test.new MyThread();  
        MyThread thread2 = test.new MyThread();  
        thread1.start();  
        thread2.start();  
    }  
      
    /** 
     * 为静态变量value加2 
     * @return 
     */  
    public int next() {  
        value++;  
        Thread.yield(); // 加速问题的产生  
        value++;  
        return value;  
    }  
      
    /** 
     * 判断是否偶数 
     * @param num 
     * @return boolean 是否偶数 
     */  
    public boolean isEven(int num) {  
        return num % 2 == 0;  
    }  
      
    class MyThread extends Thread {  
        @Override  
        public void run() {  
            System.out.println(Thread.currentThread() + " start!");  
            while(isEven(next()));  
            System.out.println(Thread.currentThread() + " down!");  
        }  
    }  
      
}  
```
上面的代码创建了两个线程操作Test类中的静态变量value，调用next方法每次会为value的值加2，理论上来说isEven方法的返回值应该总是true，两个线程的工作会不停止的执行下去。但事实是：
![同步问题例子](1.jpg)
因此在我们进行多线程并发编程时，使用同步技术是非常重要的。

## synchronized

Java以提供关键字synchronized的形式，为防止资源冲突提供了内置支持。当某个线程处于一个对于标记为synchronized的方法的调用中，那么在这个线程从方法返回前，其他所有要调用类中任何标记为synchronized方法的线程都会被阻塞。对刚才的代码稍作修改，如下：
```java
	/** 
     * 为静态变量value加2 
     * @return 
     */  
    public synchronized int next() {  
        value++;  
        Thread.yield(); // 加速问题的产生  
        value++;  
        return value;  
    } 
```
除了锁定方法，synchronized关键字还能锁定固定代码块：
```java
	/** 
     * 为静态变量value加2 
     *  
     * @return 
     */  
    public int next() {  
        synchronized (this) {  
            value++;  
            Thread.yield(); // 加速问题的产生  
            value++;  
            return value;  
        }  
    } 
```
在synchronized关键字后的小括号内加入要加锁的对象即可。通过这种方法分离出来的代码段被称为临界区，也叫作同步控制块。
加入了synchronized后，在一个线程访问next方法的时候，另一个线程就无法访问next方法了，使得两个线程的工作互不干扰，循环也变得根本停不下来。

## ReentrantLock

除了synchronized关键字外，我们还可以使用Lock对象为我们的代码加锁，Lock对象必须被显示地创建、锁定和释放：
```java
	private static Lock lock = new ReentrantLock();  
	/** 
     * 为静态变量value加2 
     * @return 
     */  
    public int next() {  
        lock.lock();  
        try {  
            value++;  
            Thread.yield(); // 加速问题的产生  
            value++;  
            return value;  
        } finally {  
            lock.unlock();  
        }  
    }  
```
一般而言，当我们使用synchronized时，需要写的代码量更少，因此通常只有我们在解决某些特殊问题时，才需要使用到Lock对象，比如尝试去获得锁：
```java
	/** 
     * 为静态变量value加2 
     * @return 
     */  
    public int next() {  
        boolean getLock = lock.tryLock();  
        if (getLock) {  
            try {  
                value++;  
                Thread.yield(); // 加速问题的产生  
                value++;  
                return value;  
            } finally {  
                lock.unlock();  
            }  
        } else {  
            // do something else  
            System.out.println(Thread.currentThread() + "say : I don't get the lock, QAQ");  
            return 0;  
        }  
    }  
```
除了ReentrantLock外，Lock类还有众多子类锁，在此不做深入讨论。值得注意的是，很明显，使用Lock通常会比使用synchronized高效许多，但我们并发编程时都应该从synchronized关键字入手，只有在性能调优时才替换为Lock对象这种做法。

# 锁的等级：对象锁、类锁

这是关于synchronized关键字的概念，synchronized关键字可以用来锁定对象的非静态方法或其中的代码块，此时关键字是为对象的实例加锁了，所以称为对象锁：
```java
	public synchronized void f() {};  
    public void g() {  
        synchronized (this) {  
              
        }  
    }  
```
另外，synchronized也可以用来锁定类的静态方法和其中的代码块，此时关键字就是为类（类的Class对象）加锁了，因此被称为类锁：
```java
public class Test {  
    public static synchronized void f() {};  
    public static void g() {  
        synchronized (Test.class) {  
              
        }  
    }  
}  
```

# 写出生产者消费者模式

生产者消费者模式一般而言有四种实现方法：
1. wait和notify方法
2. await和signal方法
3. BlockingQueue阻塞队列方法
4. PipedInputStream和PipedOutputStream管道流方法

第一种方法（wait和notify）的实现：
```java
import java.util.LinkedList;  
import java.util.Queue;  
  
class MyQueue {  
    Queue<Integer> q;  
    int size; // 队列持有产品数  
    final int MAX_SIZE = 5; // 队列最大容量  
  
    public MyQueue() {  
        q = new LinkedList<>();  
        size = 0;  
    }  
  
    /** 
     * 生产产品 
     *  
     * @param num 
     *            产品号码 
     */  
    public synchronized void produce(int num) {  
        // 容量不足时，等待消费者消费  
        try {  
            while (size > MAX_SIZE)  
                wait();  
        } catch (InterruptedException e) {  
        }  
        ;  
        System.out.println("produce " + num);  
        q.add(num);  
        size++;  
        // 提醒消费者消费  
        notifyAll();  
    }  
      
    /** 
     * 消费产品 
     */  
    public synchronized void comsume() {  
        // 没有产品时，等待生产  
        try {  
            while (size < 1)  
                wait();  
        } catch (InterruptedException e) {  
        }  
        ;  
        System.out.println("comsume " + q.poll());  
        size--;  
        // 提醒生产者生产  
        notifyAll();  
    }  
}  
  
class Producer extends Thread {  
    private MyQueue q;  
  
    public Producer(MyQueue q) {  
        this.q = q;  
    }  
  
    @Override  
    public void run() {  
        for (int i = 0; i < 10; i++)  
            q.produce(i);  
    }  
}  
  
class Consumer extends Thread {  
    private MyQueue q;  
  
    public Consumer(MyQueue q) {  
        this.q = q;  
    }  
  
    @Override  
    public void run() {  
        for (int i = 0; i < 10; i++)  
            q.comsume();  
    }  
}  
  
public class Test {  
    public static void main(String[] args) {  
        MyQueue q = new MyQueue();  
        Producer producer = new Producer(q);  
        Consumer consumer = new Consumer(q);  
        producer.start();  
        consumer.start();  
    }  
}  
```

第二种方法（await和signal）实现：
```java
import java.util.LinkedList;  
import java.util.Queue;  
import java.util.concurrent.locks.Condition;  
import java.util.concurrent.locks.Lock;  
import java.util.concurrent.locks.ReentrantLock;  
  
class MyQueue {  
    Queue<Integer> q;  
    int size; // 队列持有产品数  
    final int MAX_SIZE = 5; // 队列最大容量  
    private Lock lock; // 锁  
    private Condition condition; // 条件变量  
  
    public MyQueue() {  
        q = new LinkedList<>();  
        size = 0;  
        lock = new ReentrantLock();  
        condition = lock.newCondition();  
    }  
  
    /** 
     * 生产产品 
     *  
     * @param num 
     *            产品号码 
     */  
    public void produce(int num) {  
        // 进入临界区上锁  
        lock.lock();  
        // 容量不足时，等待消费者消费  
        try {  
            while (size > MAX_SIZE)  
                condition.await();  
        } catch (InterruptedException e) {  
            e.printStackTrace();  
        };  
        System.out.println("produce " + num);  
        q.add(num);  
        size++;  
        // 提醒消费者消费  
        condition.signalAll();  
        // 退出临界区解锁  
        lock.unlock();  
    }  
  
    /** 
     * 消费产品 
     */  
    public void comsume() {  
        // 上锁进入临界区  
        lock.lock();  
        // 没有产品时，等待生产  
        try {  
            while (size < 1)  
                condition.await();  
        } catch (InterruptedException e) {  
            e.printStackTrace();  
        };  
        System.out.println("comsume " + q.poll());  
        size--;  
        // 提醒生产者生产  
        condition.signalAll();  
        // 退出临界区解锁  
        lock.unlock();  
    }  
}  
  
class Producer extends Thread {  
    private MyQueue q;  
  
    public Producer(MyQueue q) {  
        this.q = q;  
    }  
  
    @Override  
    public void run() {  
        for (int i = 0; i < 10; i++)  
            q.produce(i);  
    }  
}  
  
class Consumer extends Thread {  
    private MyQueue q;  
  
    public Consumer(MyQueue q) {  
        this.q = q;  
    }  
  
    @Override  
    public void run() {  
        for (int i = 0; i < 10; i++)  
            q.comsume();  
    }  
}  
  
public class Main {  
    public static void main(String[] args) {  
        MyQueue q = new MyQueue();  
        Producer producer = new Producer(q);  
        Consumer consumer = new Consumer(q);  
        producer.start();  
        consumer.start();  
    }  
}  
```

第三种方法（BlockingQueue阻塞队列）实现：
```java
import java.util.concurrent.BlockingQueue;  
import java.util.concurrent.LinkedBlockingQueue;  
  
class MyQueue {  
    BlockingQueue<Integer> q; // 阻塞队列  
    int size; // 队列持有产品数（此例无用）  
    final int MAX_SIZE = 5; // 队列最大容量  
  
    public MyQueue() {  
        q = new LinkedBlockingQueue<>(MAX_SIZE);  
    }  
  
    /** 
     * 生产产品 
     *  
     * @param num 
     *            产品号码 
     */  
    public void produce(int num) {  
        // 阻塞队列会自动阻塞，不需要处理  
        try {  
            q.put(num);  
            System.out.println("produce " + num);  
        } catch (InterruptedException e) {  
            e.printStackTrace();  
        }  
    }  
  
    /** 
     * 消费产品 
     */  
    public void comsume() {  
        // 阻塞队列会自动阻塞，不需要处理  
        try {  
            System.out.println("comsume " + q.take());  
        } catch (InterruptedException e) {  
            e.printStackTrace();  
        }  
    }  
}  
  
class Producer extends Thread {  
    private MyQueue q;  
  
    public Producer(MyQueue q) {  
        this.q = q;  
    }  
  
    @Override  
    public void run() {  
        for (int i = 0; i < 10; i++)  
            q.produce(i);  
    }  
}  
  
class Consumer extends Thread {  
    private MyQueue q;  
  
    public Consumer(MyQueue q) {  
        this.q = q;  
    }  
  
    @Override  
    public void run() {  
        for (int i = 0; i < 10; i++)  
            q.comsume();  
    }  
}  
  
public class Main {  
    public static void main(String[] args) {  
        MyQueue q = new MyQueue();  
        Producer producer = new Producer(q);  
        Consumer consumer = new Consumer(q);  
        producer.start();  
        consumer.start();  
    }  
}  
```

第四种方法（PipedInputStream和PipedOutputStream）：
```java
import java.io.PipedInputStream;  
import java.io.PipedOutputStream;  
  
class MyQueue {  
    int size; // 队列持有产品数（此例无用）  
    final int MAX_SIZE = 5; // 队列最大容量  
    PipedInputStream pis;  
    PipedOutputStream pos;  
  
    public MyQueue() {  
        // 初始化流  
        pis = new PipedInputStream(MAX_SIZE);  
        pos = new PipedOutputStream();  
        // 管道流建立连接  
        try {  
            pos.connect(pis);  
        } catch (Exception e) {  
            e.printStackTrace();  
        }  
    }  
  
    /** 
     * 生产产品 
     *  
     * @param num 
     *            产品号码 
     */  
    public void produce(int num) {  
        // 管道流会自动阻塞，不需要处理  
        try {  
            // 输出写在前面，否则会有奇怪的事情发生～  
            System.out.println("produce " + num);  
            pos.write(num);  
            pos.flush();  
        } catch (Exception e) {  
            e.printStackTrace();  
        }  
    }  
  
    /** 
     * 消费产品 
     */  
    public void comsume() {  
        // 管道流会自动阻塞，不需要处理  
        try {  
            System.out.println("comsume " + pis.read());  
        } catch (Exception e) {  
            e.printStackTrace();  
        }  
    }  
  
    @Override  
    protected void finalize() throws Throwable {  
        pis.close();  
        pos.close();  
        super.finalize();  
    }  
}  
  
class Producer extends Thread {  
    private MyQueue q;  
  
    public Producer(MyQueue q) {  
        this.q = q;  
    }  
  
    @Override  
    public void run() {  
        for (int i = 0; i < 10; i++)  
            q.produce(i);  
    }  
}  
  
class Consumer extends Thread {  
    private MyQueue q;  
  
    public Consumer(MyQueue q) {  
        this.q = q;  
    }  
  
    @Override  
    public void run() {  
        for (int i = 0; i < 10; i++)  
            q.comsume();  
    }  
}  
  
public class Main {  
    public static void main(String[] args) {  
        MyQueue q = new MyQueue();  
        Producer producer = new Producer(q);  
        Consumer consumer = new Consumer(q);  
        producer.start();  
        consumer.start();  
    }  
}  
```

输出结果：
![生产者消费者模式例子](2.jpg)

# ThreadLocal的设计理念与作用

ThreadLocal即线程本地存储。防止线程在共享资源上产生冲突的一种方式是根除对变量的共享。ThreadLocal是一种自动化机制，可以为使用相同变量的每个不同的线程都创建不同的存储，ThreadLocal对象通常当做静态域存储，通过get和set方法来访问对象的内容：
```java
import java.util.Random;  
import java.util.concurrent.ExecutorService;  
import java.util.concurrent.Executors;  
import java.util.concurrent.TimeUnit;  
  
class Accessor implements Runnable {  
    private final int id; // 线程id  
    public Accessor(int id) {  
        this.id = id;  
    }  
    @Override  
    public void run() {  
        while(!Thread.currentThread().isInterrupted()) {  
            ThreadLocalVariableHolder.increment();  
            System.out.println(this);  
            Thread.yield();  
        }  
    }  
    @Override  
    public String toString() {  
        return "#" + id + " : " + ThreadLocalVariableHolder.get();  
    }  
}  
public class ThreadLocalVariableHolder {  
    private static ThreadLocal<Integer> value = new ThreadLocal<Integer>() {  
        // 返回随机数作为初始值  
        protected Integer initialValue() {  
            return new Random().nextInt(10000);  
        }  
    };  
      
    /** 
     * 为当前线程的value值加一 
     */  
    public static void increment() {  
        value.set(value.get() + 1);  
    }  
      
    /** 
     * 返回当前线程存储的value值 
     * @return 
     */  
    public static int get() {  
        return value.get();  
    }  
      
    public static void main(String[] args) throws InterruptedException {  
        ExecutorService service = Executors.newCachedThreadPool();  
        // 开启5个线程  
        for (int i = 0; i < 5; i++)  
            service.execute(new Accessor(i));  
        // 所有线程运行3秒  
        TimeUnit.SECONDS.sleep(1);  
        // 关闭所有线程  
        service.shutdownNow();  
    }  
}  
```

运行部分结果如下：
![ThreadLocal例子](3.jpg)

在上面的例子中虽然多个线程都去调用了ThreadLocalVariableHolder的increment和get方法，但这两个方法都没有进行同步处理，这是因为ThreadLocal保证我们使用的时候不会出现竞争条件。从结果来看，每个线程都在单独操作自己的变量，每个单独的线程都被分配了自己的存储（即便只有一个ThreadLocalVariableHolder对象），线程之间并没有互相造成影响。对于多线程资源共享的问题，同步机制采用了“以时间换空间”的方式，而ThreadLocal采用了“以空间换时间”的方式。在ThreadLocal类中有一个Map，用于存储每一个线程的变量的副本。