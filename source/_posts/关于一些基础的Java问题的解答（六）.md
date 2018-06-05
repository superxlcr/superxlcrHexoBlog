---
title: 关于一些基础的Java问题的解答（六）
tags: [java,基础知识]
categories: [java]
date: 2016-03-20 09:53:11
description: ThreadPool用法与优势、Concurrent包里的工具：ArrayBlockingQueue、CountDownLatch等等、wait()和sleep()的区别、foreach与正常for循环效率对比、Java IO与NIO
---
上一篇文章的传送门：[关于一些基础的Java问题的解答（五）](/2016/03/19/关于一些基础的Java问题的解答（五）/)

# ThreadPool用法与优势

ThreadPool即线程池，它是JDK1.5引入的Concurrent包中用于处理并发编程的工具。使用线程池有如下好处：
1. 降低资源消耗：通过重复利用已创建的线程降低线程创建和销毁造成的消耗
2. 提高响应速度：当任务到达时，任务可以不需要等到线程创建，复用缓存线程就能立即执行
3. 提高线程的可管理性：线程是稀缺资源，如果无限制的创建，不仅会消耗系统资源，还会降低系统的稳定性，使用线程池可以进行统一的分配，调优和监控

线程池的创建方法如下：
通过ThreadPoolExecutor创建：
```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue, threadFactory, handler);  
```
该构造函数具有重载，最后两个参数是可选的，各参数的含义如下：
1. corePoolSize（基本线程池大小）：当提交一个任务到线程池时，线程池会创建一个线程来执行任务，即使其他空闲的基本线程能够执行新任务也会创建线程，等到需要执行的任务数大于基本线程池大小时就不再创建。线程池有一个prestartAllCoreThreads方法用来提前创建并启动所有基本线程
2. maximumPoolSize（最大线程池大小）：线程池允许创建的最大线程数，必须大于基本线程池大小。线程池中含有一个任务阻塞队列，如果队列满了，并且已创建的线程数小于最大线程数，则线程池会再创建新的线程执行任务
3. keepAliveTime（线程活动保持时间）：线程池的工作线程空闲后，保持存活的时间。所以如果任务很多，并且每个任务执行的时间比较短，可以调大这个时间，提高线程的利用率
4. unit（线程活动保持时间的单位）：前一个参数的时间单位
5. workQueue（工作队列）：用于保存等待执行的任务的阻塞队列，如果当任务提交时基本线程池中已没有空闲线程，则任务会被放入工作队列等待
6. threadFactory（线程工厂，可选参数）：创建一个新线程时会调用其newThread方法，可以初始化一些信息
7. handler（饱和处理器）：当队列和线程池都满了，说明线程池处于饱和状态，那么就会使用处理器处理提交的新任务。ThreadPoolExecutor内部提供了一些handler的静态实现类：
- AbortPolicy：直接抛出异常RejectedExecutionException
- CallerRunsPolicy：使用调用者所在线程来运行任务
- DiscardOldestPolicy：丢弃队列头的任务，并执行当前任务
- DiscardPolicy：什么也不做，放弃掉该任务

除了自定义线程池，我们还可以使用Executors类中提供的实现好的线程池：
```java
	// 为每个任务创建一个新线程或回收利用旧线程的线程池  
        ExecutorService service1 = Executors.newCachedThreadPool();  
        // 创建指定数目线程的线程池  
        ExecutorService service2 = Executors.newFixedThreadPool(nThreads);  
        // 线程数量固定为1的线程池，拥有无界的工作队列  
        ExecutorService service3 = Executors.newSingleThreadExecutor(); 
```

以上三种线程池为比较常用的线程池，其余的不再探讨。
线程池的用法如下：
```java
// 传入一个Runnable接口处理任务  
executor.execute(runnable);  
// 传入一个Callable接口处理任务，返回Future对象代表任务返回值  
Future<Object> result = executor.submit(callable);  
// 使用get方法获取具体内容  
result.get();  
// 关闭线程池，中断空闲线程  
executor.shutdown();  
// 关闭线程池，中断所有线程  
executor.shutdownNow();  
```

综上，线程池的工作流程为：
![线程池工作流程图](1.png)

# Concurrent包里的工具：ArrayBlockingQueue、CountDownLatch等等

JDK1.2引入的容器类库为了效率问题所以是不同步的，要同步容器类我们只能自己实现或指望Collections类提供的各种static同步方法。但在JDK1.5中引入的Concurrent包中，Java为我们提供了许多线程安全的免锁容器，主要使用的有以下几种：
1. CopyOnWriteArrayList：写时拷贝列表，对容器的写入操作将导致创建整个底层数组的副本，而原数组保存在原地，使得当数组在被修改时，读取可以安全的执行。修改完成时，一个原子性的操作会把新数组换入，使得新的读取操作可以看到修改
2. CopyOnWriteArraySet：与CopyOnWriteArrayList类似的集合
3. ConcurrentHashMap：线程安全的HashMap
4. ConcurrentLinkedQueue：一个线程安全的队列
5. DelayQueue：延迟队列，是一个无界阻塞队列，对于放入的元素实现延迟接口，设定延迟时间，元素只有过了延迟时间后才能被取走
6. LinkedBlockingQueue：阻塞队列，LinkedBlockingQueue 可以指定容量，也可以不指定，不指定的话，默认最大是Integer.MAX_VALUE，其中主要用到put和take方法，put方法在队列满的时候会阻塞直到有队列成员被消费，take方法在队列空的时候会阻塞，直到有队列成员被放进来
7. PriorityBlockingQueue：优先级阻塞队列，与优先级队列相似，具有阻塞的特点

除了容器外，Concurrent还为我们提供了各种用于同步的辅助类，常见的有以下几种：
1. Atomic类，原子类，各种包装类型的同步类，可以使用compareAndSet形式的方法来更新变量
2. CountDownLatch：一个同步辅助类，在完成一组正在其他线程中执行的操作之前，它允许一个或多个线程一直等待。其构造方法可以指定计数次数，当其countDown方法被调用时次数减一，而调用await方法（可以设置超时）会使当前线程一直被阻塞，直到计时器的值为0
3. CyclicBarrier：回环栅栏，通过它可以实现让一组线程等待至某个状态之后再全部同时执行。叫做回环是因为当所有等待线程都被释放以后，CyclicBarrier可以被重用。其构造方法可以指定线程数，还可以传入Runnable表示所有线程到达状态后要执行的内容。线程调用其await方法时表示线程已到达指定状态并被阻塞，当有所有线程都到达状态时，线程才可以继续往下执行。调用其reset方法可以重复使用。
4. Exchanger：用于两个线程交换数据的辅助类，调用exchange方法后线程会被阻塞直到另一个线程调用exchange方法（可以设置超时）
5. ReadWriteLock：读写锁允许我们拥有多个读者，对向数据结构相对不频繁的写入但频繁读取做了优化。我们分别可以调用readLock和writeLock方法获取读锁和写锁，使用lock和unlock方法来加锁和解锁。
6. Semaphore：信号量，可以控同时访问的线程个数（通过构造函数设置），通过 acquire方法 获取一个许可，如果没有就等待，而 release方法 释放一个许可。

# wait()和sleep()的区别

wait是Object类的方法，只有当线程拥有调用对象的锁的时候才可以调用该方法，否则会抛出IllegalMonitorStateException。wait方法的作用是阻塞当前线程并让出调用对象的锁，直到别的线程使用notify或notifyAll唤醒，或经过特定时间，或被别的线程中断才继续工作。
sleep是Thread类的静态方法，他可以让当前线程休眠一段时间，直到经过特定休眠时间，或被别的线程中的才继续工作。

# foreach与正常for循环效率对比

for的写法都比较熟悉就不提了。
foreach语句是JDK1.5的新特征之一，在遍历数组、集合方面，foreach为开发人员提供了极大的方便。foreach语句是for语句的特殊简化版本，但是foreach语句并不能完全取代for语句，然而，任何的foreach语句都可以改写为for语句版本。foreach并不是一个关键字，习惯上将这种特殊的for语句格式称之为“foreach”语句。可以使用foreach的对象必须实现Iterator 接口。
一般而言，只是遍历的话我们可以使用foreach，如果要涉及对数组或容器的操作就只能使用for循环了。

# Java IO与NIO

Java NIO（Java new IO），是jdk1.4 里提供的新api ，为所有的原始类型提供缓存支持。
两者的不同主要体现在以下几点：Java IO是面向流的而Java NIO是面向缓冲的，Java IO提供阻塞IO服务而Java NIO提供非阻塞IO服务，Java IO需要使用多个线程来处理多个IO流而Java NIO引入了Selector（选择器）来处理多个Channel（通道）。

## 面向流与面向缓冲

Java NIO和IO之间第一个最大的区别是，IO是面向流的，NIO是面向缓冲区的。 Java IO面向流意味着每次从流中读一个或多个字节，直至读取所有字节，它们没有被缓存在任何地方。此外，它不能前后移动流中的数据。如果需要前后移动从流中读取的数据，需要先将它缓存到一个缓冲区。 Java NIO的缓冲导向方法略有不同。数据读取到一个它稍后处理的缓冲区，需要时可在缓冲区中前后移动。这就增加了处理过程中的灵活性。但是，还需要检查是否该缓冲区中包含所有您需要处理的数据。而且，需确保当更多的数据读入缓冲区时，不要覆盖缓冲区里尚未处理的数据。

## 阻塞IO与非阻塞IO

Java IO的各种流是阻塞的。这意味着，当一个线程调用read() 或 write()时，该线程被阻塞，直到有一些数据被读取，或数据完全写入。该线程在此期间不能再干任何事情了。 Java NIO的非阻塞模式，使一个线程从某通道发送请求读取数据，但是它仅能得到目前可用的数据，如果目前没有数据可用时，就什么都不会获取。而不是保持线程阻塞，所以直至数据变的可以读取之前，该线程可以继续做其他的事情。 非阻塞写也是如此。一个线程请求写入一些数据到某通道，但不需要等待它完全写入，这个线程同时可以去做别的事情。 线程通常将非阻塞IO的空闲时间用于在其它通道上执行IO操作，所以一个单独的线程现在可以管理多个输入和输出通道（channel）

## 选择器（Selector）

Java NIO的选择器允许一个单独的线程来监视多个输入通道，你可以注册多个通道使用一个选择器，然后使用一个单独的线程来“选择”通道：这些通道里已经有可以处理的输入，或者选择已准备写入的通道。这种选择机制，使得一个单独的线程很容易来管理多个通道。

Java IO 优点：简单容易编写。 缺点：阻塞IO使得线程在等待输入时什么也不能做，多个流需要多个线程来处理。
Java NIO 优点：非阻塞IO非常灵活，多个通道可以通过搭配选择器使用少数线程来处理。 缺点：编写复杂。若传输内容以行为单位且具有一定逻辑性，则传输过程逻辑性可能会丢失（每次接收的数据不一定为完整一行，没有readLine方法）。