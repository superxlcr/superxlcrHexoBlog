---
title: 不可思议的OOM
tags: [android,内存,源码]
categories: [android]
date: 2018-07-22 18:06:04
description: （转载）引子、问题描述、问题分析及解决、结论及监控、Demo
---
本文转载自：https://www.jianshu.com/p/e574f0ffdb42

摘要:
本文发现了一类OOM（OutOfMemoryError），**这类OOM的特点是崩溃时java堆内存和设备物理内存都充足**，探索并解释了这类OOM抛出的原因。
关键字:

OutOfMemoryError ，OOM，pthread_create failed , Could not allocate JNI Env

# 引子

对于每一个移动开发者，内存是都需要小心使用的资源，而线上出现的OOM（OutOfMemoryError）都会让开发者抓狂，因为我们通常仰仗的直观的堆栈信息对于定位这种问题通常帮助不大。

网上有很多资料教我们如何“紧衣缩食“的利用宝贵的堆内存（比如，使用小图片，bitmap复用等），可是:


- **线上的OOM真的全是由于堆内存紧张导致的吗？**

- **有没有App堆内存宽裕，设备物理内存也宽裕的情况下发生OOM的可能？**




**内存充裕的时候出现OOM崩溃？**看似不可思议，然而，最近笔者在调查一个问题的时候，通过自研的APM平台发现公司的一个产品的大部分OOM确实有这样的特征，即：

- **OOM崩溃时，java堆内存远远低于Android虚拟机设定的上限，并且物理内存充足，SD卡空间充足**



既然内存充足，这时候为什么会有OOM崩溃呢？

# 问题描述

在详细描述问题之前，先弄清楚一个问题：
**什么导致了OOM的产生？**
下面是几个关于Android官方声明内存限制阈值的API：

```java
ActivityManager.getMemoryClass()：     虚拟机java堆大小的上限，分配对象时突破这个大小就会OOM
ActivityManager.getLargeMemoryClass()：manifest中设置largeheap=true时虚拟机java堆的上限
Runtime.getRuntime().maxMemory() ：    当前虚拟机实例的内存使用上限，为上述两者之一
Runtime.getRuntime().totalMemory() ：  当前已经申请的内存，包括已经使用的和还没有使用的
Runtime.getRuntime().freeMemory() ：   上一条中已经申请但是尚未使用的那部分。那么已经申请并且正在使用的部分used=totalMemory() - freeMemory()
ActivityManager.MemoryInfo.totalMem:   设备总内存
ActivityManager.MemoryInfo.availMem:   设备当前可用内存
/proc/meminfo                                           记录设备的内存信息

```

图2-1 Android内存指标
通常认为OOM发生是由于java堆内存不够用了，即

```java
Runtime.getRuntime().maxMemory()这个指标满足不了申请堆内存大小时

```

图2-2 Java堆OOM产生原因

这种OOM可以非常方便的验证（比如: 通过new byte[]的方式尝试申请超过阈值maxMemory()的堆内存），通常这种OOM的错误信息通常如下：

```java
java.lang.OutOfMemoryError: Failed to allocate a XXX byte allocation with XXX free bytes and XXXKB until OOM

```

图2-3 堆内存不够导致的OOM的错误信息

而前面已经提到了，**本文中发现的OOM案例中堆内存充裕（Runtime.getRuntime().maxMemory()大小的堆内存还剩余很大一部分），设备当前内存也很充裕（ActivityManager.MemoryInfo.availMem还有很多）**。这些OOM的错误信息大致有下面两种：

1. 这种OOM在Android6.0，Android7.0上各个机型均有发生，文中简称为**OOM一**，错误信息如下：
	```java
	java.lang.OutOfMemoryError: Could not allocate JNI Env
	```
	图2-4 OOM一的错误信息
2. 集中发生在Android7.0及以上的华为手机（EmotionUI_5.0及以上）的OOM，简称为**OOM二**，对应错误信息如下：
	```java
	java.lang.OutOfMemoryError: pthread_create (1040KB stack) failed: Out of memory
	```
	图2-5 OOM二的错误信息

# 问题分析及解决


## 代码分析

Android系统中，OutOfMemoryError这个错误是怎么被系统抛出的？下面基于Android6.0的代码进行简单分析：

1. Android虚拟机最终抛出OutOfMemoryError的代码位于 **/art/runtime/thread.cc**
	```java
	void Thread::ThrowOutOfMemoryError(const char* msg)
	参数msg携带了OOM时的错误信息
	```
	图3-1 ART Runtime抛出的位置
2. 搜索代码可以发现以下几个地方调用了上述方法抛出OutOfMemoryError错误

- 第一个地方是堆操作时
```java
系统源码文件：
    /art/runtime/gc/heap.cc
函数：
    void Heap::ThrowOutOfMemoryError(Thread* self, size_t byte_count, AllocatorType allocator_type)
抛出时的错误信息：
    oss << "Failed to allocate a " << byte_count << " byte allocation with " << total_bytes_free  << " free bytes and " << PrettySize(GetFreeMemoryUntilOOME()) << " until OOM";

```

图3-2 Java堆OOM

这种抛出的其实就是堆内存不够用的时候，即前面提到的申请堆内存大小超过了Runtime.getRuntime().maxMemory()

- 第二个地方是创建线程时




```java
系统源码文件：
    /art/runtime/thread.cc
函数：
    void Thread::CreateNativeThread(JNIEnv* env, jobject java_peer, size_t stack_size, bool is_daemon)
抛出时的错误信息：
    "Could not allocate JNI Env"
  或者
    StringPrintf("pthread_create (%s stack) failed: %s", PrettySize(stack_size).c_str(), strerror(pthread_create_result)));

```

图3-3 线程创建时OOM

对比错误信息，可以知道我们遇到的OOM崩溃就是这个时机，即创建线程的时候（Thread::CreateNativeThread）产生的。

- 还有其他的一些错误信息如”[XXXClassName] of length XXX would overflow“是系统限制String/Array的长度所致，不在本文讨论之列。



那么，我们关心的就是Thread::CreateNativeThread时抛出的OOM错误，**创建线程为什么会导致OOM呢？**

## 推断

既然抛出来OOM，一定是线程创建过程中触发了某些我们不知道的限制，既然不是Art虚拟机为我们设置的堆上限，那么可能是更底层的限制。

Android系统基于linux，所以linux的限制对于Android同样适用，这些限制有：

### /proc/pid/limits 描述着linux系统对对应进程的限制
下面是一个样例：




```java
Limit                     Soft Limit           Hard Limit           Units     
Max cpu time              unlimited            unlimited            seconds   
Max file size             unlimited            unlimited            bytes     
Max data size             unlimited            unlimited            bytes     
Max stack size            8388608              unlimited            bytes     
Max core file size        0                    unlimited            bytes     
Max resident set          unlimited            unlimited            bytes     
Max processes             13419                13419                processes 
Max open files            1024                 4096                 files     
Max locked memory         67108864             67108864             bytes     
Max address space         unlimited            unlimited            bytes     
Max file locks            unlimited            unlimited            locks     
Max pending signals       13419                13419                signals   
Max msgqueue size         819200               819200               bytes     
Max nice priority         40                   40                   
Max realtime priority     0                    0                    
Max realtime timeout      unlimited            unlimited            us 

```

图3-4 Linux进程限制示例

用排除法筛选上面样例中的limits:

- Max stack size，Max processes的限制是整个系统的，不是针对某个进程的，排除

- Max locked memory ，排除，后面会分析，线程创建过程中分配线程私有stack使用的mmap调用没有设置MAP_LOCKED，所以这个限制与线程创建过程无关

- Max pending signals，c层信号个数阈值，无关，排除

- Max msgqueue size，Android IPC机制不支持消息队列，排除



剩下的limits项中，**Max open files**这一项限制最可疑

Max open files表示**每个进程最大打开文件的数目**，进程**每打开一个文件就会产生一个文件描述符fd（记录在/proc/pid/fd下面）**，这个限制表明**fd的数目不能超过Max open files规定的数目**。

后面分析线程创建过程中会发现过程中涉有及到文件描述符。

### /proc/sys/kernel中描述的限制



这些限制中与线程相关的是/proc/sys/kernel/threads-max，规定了每个进程创建线程数目的上限，所以线程创建导致OOM的原因也有可能与这个限制相关。
**Ps：其实这里的说法并不完全正确，根据博主自己测试以及查询相关资料后得知，该文件规定的应该是整个系统总的线程数目限制**
**Pss：目前并没有找到相关的方法查询到明确的线程限制，我们只能通过 ulimit -a 指令查看一些别的限制条件，通过测试得知，部分华为手机单进程限制线程数为500左右**

## 验证

下面对上述的推断进行验证，分两步：本地验证和线上验收。

- 本地验证：在本地验证推断，**试图复现与图[2-4]OOM一与图[2-5]OOM二所示错误消息一致的OOM**

- 线上验收：**下发插件，验收线上用户OOM时确实是由于上面的推断的原因导致的**。




### 本地验证

**实验一：**

触发大量网络连接（每个连接处于独立的线程中）并保持，每打开一个socket都会增加一个fd（/proc/pid/fd下多一项）

注：不只有这一种增加fd数的方式，也可以用其他方法，比如打开文件，创建handlerthread等等

- 实验预期：当进程fd数（可以通过 ls /proc/pid/fd | wc -l 获得）突破 /proc/pid/limits中规定的Max open files时，产生OOM

- 实验结果：当fd数目到达 /proc/pid/limits中规定的Max open files时，继续开线程确实会导致OOM的产生。错误信息及堆栈如下：




```java
E/art: ashmem_create_region failed for 'indirect ref table': Too many open files
E/AndroidRuntime: FATAL EXCEPTION: main
                  Process: com.netease.demo.oom, PID: 2435
                  java.lang.OutOfMemoryError: Could not allocate JNI Env
                      at java.lang.Thread.nativeCreate(Native Method)
                      at java.lang.Thread.start(Thread.java:730)
                      ......

```

图3-5 FD数超限导致OOM的详细信息

可以看出，**此OOM发生时的错误信息确与线上发现的OOM一的“Could not allocate JNI Env”**吻合，因此线上上报的OOM一**可能**就是由FD数超限导致的，不过最终确定需要到线上进行验证(下一小节).

此外从ART虚拟机的Log中看出，还有一个关键的信息**“ art: ashmem_create_region failed for 'indirect ref table': Too many open files”**，后面会用于问题定位及解释。
**实验二：**

创建大量的空线程（不做任何事情，直接sleep）

- 实验预期：当线程数（可以在/proc/pid/status中的threads项实时查看）超过/proc/sys/kernel/threads-max中规定的上限时产生OOM崩溃

- 实验结果：




1. 在Android7.0及以上的华为手机（EmotionUI_5.0及以上）的手机产生OOM，这些手机的线程数限制都很小(应该是华为rom特意修改的limits)，每个进程只允许最大同时开500个线程，因此很容易复现了。OOM时错误信息如下：




```java
W libc    : pthread_create failed: clone failed: Out of memory
W art     : Throwing OutOfMemoryError "pthread_create (1040KB stack) failed: Out of memory"
E AndroidRuntime: FATAL EXCEPTION: main
                  Process: com.netease.demo.oom, PID: 4973
                  java.lang.OutOfMemoryError: pthread_create (1040KB stack) failed: Out of memory
                      at java.lang.Thread.nativeCreate(Native Method)
                      at java.lang.Thread.start(Thread.java:745)
                      ......

```

图3-6 线程数超限导致的OOM详细信息

可以看出**错误信息与我们线上遇到的OOM二吻合："pthread_create (1040KB stack) failed: Out of memory"**

另外ART虚拟机还有一个关键Log：**“pthread_create failed: clone failed: Out of memory”**，后面会用于问题定位及解释。

1. 其他Rom的手机线程数的上限都比较大，不容易复现上述问题。但是，**对于32位的系统，当进程的逻辑地址空间不够的时候也会产生OOM**,每个线程通常需要mapp 1MB左右的stack空间（stack大小可以自行设置），32为系统进程逻辑地址4GB，用户空间少于3GB。逻辑地址空间不够（**已用逻辑空间地址可以查看/proc/pid/status中的VmPeak/VmSize记录**），此时创建线程产生的OOM具有如下信息：




```java
W/libc: pthread_create failed: couldn't allocate 1069056-bytes mapped space: Out of memory
W/art: Throwing OutOfMemoryError "pthread_create (1040KB stack) failed: Try again"
E/AndroidRuntime: FATAL EXCEPTION: main
                  Process: com.netease.demo.oom, PID: 8638
                  java.lang.OutOfMemoryError: pthread_create (1040KB stack) failed: Try again
                       at java.lang.Thread.nativeCreate(Native Method)
                       at java.lang.Thread.start(Thread.java:1063)
                       ......

```

图3-7 逻辑地址空间占满导致的OOM

### 线上验收及问题解决

**本地尝试复现的OOM错误信息中图[3-5]与线上OOM一情况比较吻合，图[3-6]与线上OOM二的情况比较吻合**，但线上的OOM一真的时FD数目超限，OOM二真的是由于华为手机线程数超限的原因导致的吗？最终确定还需要取线上设备的数据进行验证．
**验证方法：**

下发插件到线上用户，当Thread.UncaughtExceptionHandler捕获到OutOfMemoryError时记录/proc/pid目录下的如下信息：

1. /proc/pid/fd目录下文件数(fd数)

2. /proc/pid/status中threads项（当前线程数目）

3. OOM的日志信息（出了堆栈信息还包含其他的一些warning信息



**线上OOM一验证**

发生OOM一的线上设备中采集到的信息：

1. **/proc/pid/fd目录下文件数与/proc/pid/limits中的Max open files 数目持平**，证明FD数目已经满了

2. 崩溃时日志信息与图[3-5]基本一致



由此，证明**线上的OOM一确实是由于FD数目过多导致的OOM，推断验证成功**．
**OOM一的定位与解决：**

最终原因是App中使用的长连接库再某些时候会有瞬时发出大量http请求的bug(导致FD数激增)，已修复
**线上OOM二验证**

集中在华为系统的OOM二崩溃时收集到的信息样例如下，（收集的样例中包含的devicemodel有VKY-AL00，TRT-AL00A，BLN-AL20，BLN-AL10，DLI-AL10，TRT-TL10，WAS-AL00等）：

1. /proc/pid/status中threads记录全部到达上限：Threads:    500

2. 崩溃时日志信息与图[3-6]基本一致



推断验证成功，即**线程数受限导致创建线程时clone failed导致了线上的OOM二**。
**OOM二的定位与解决：**

关于App业务代码中的问题还在定位修复中

## 解释

下面从代码分析本文描述的OOM是怎么发生的，首先线程创建的简易版流程图如下所示：



![pic1](1.png)

图3-8 线程创建流程

上图中，线程创建大概有两个关键的步骤：

- 第一列中的**创建线程私有的结构体JNIENV**(JNI执行环境，用于C层调用Java层代码)

- 第二列中的**调用posix C库的函数pthread_create进行线程创建工作**



下面对流程图中关键节点（图中有标号的）进行说明：

1. 图中节点①， /art/runtime/thread.cc中的函数Thread:CreateNativeThread部分节选代码如下：




```java
    std::string msg(child_jni_env_ext.get() == nullptr ?
        "Could not allocate JNI Env" :
        StringPrintf("pthread_create (%s stack) failed: %s", PrettySize(stack_size).c_str(), strerror(pthread_create_result)));
    ScopedObjectAccess soa(env);
    soa.Self()->ThrowOutOfMemoryError(msg.c_str());

```

图3-9 Thread:CreateNativeThread节选

可知：

- JNIENV创建不成功时产生OOM的错误信息为**"Could not allocate JNI Env"，与文中OOM一一致**

- pthread_create失败时抛出OOM的错误信息为"pthread_create (%s stack) failed: %s"．其中详细的错误信息由pthread_create的返回值（错误码）给出．错误码与错误描述的对应关系可以参见**bionic/libc/include/sys/_errdefs.h**中的定义．文中OOM二的具体错误信息为"Out of memory"，就说明pthread_create的返回值为12.




```java
...
__BIONIC_ERRDEF( EAGAIN         ,  11, "Try again" )
__BIONIC_ERRDEF( ENOMEM         ,  12, "Out of memory" )
...
__BIONIC_ERRDEF( EMFILE         ,  24, "Too many open files" )
...

```

图3-10 系统错误定义_errdefs.h

1. 图中节点②和③是创建JNIENV过程的关键节点，节点②/art/runtime/mem_map.cc中**函数MemMap:MapAnonymous的作用是为JNIENV结构体中Indirect_Reference_table（C层用于存储JNI局部/全局变量）申请内存**，申请内存的方法是节点③所示的函数**ashmem_create_region（创建一块ashmen匿名共享内存,并返回一个文件描述符）**．节点②代码节选如下：




```java
  if (fd.get() == -1) {
      *error_msg = StringPrintf("ashmem_create_region failed for '%s': %s", name, strerror(errno));
      return nullptr;
  }

```

图3-11　MemMap:MapAnonymous节选

我们**线上的OOM一的错误信息＂ashmem_create_region failed for 'indirect ref table': Too many open files＂，与此处打印的信息吻合**．＂Too many open files＂的错误描述说明此处的errno（系统全局错误标识）为24(见图[3-10]系统错误定义_errdefs.h)．

由此看出我们线上的**OOM一是由于文件描述符数目已满，ashmem_create_region无法返回新的FD而导致的**．

1. 图中节点④和⑤是调用C库创建线程时的环节，创建线程首先**调用__allocate_thread函数申请线程私有的栈内存(stack)等**，然后**调用clone方法进行线程创建**．申请stack采用的时mmap的方式，节点⑤代码节选如下：




```java
  if (space == MAP_FAILED) {
    __libc_format_log(ANDROID_LOG_WARN,
                      "libc",
                      "pthread_create failed: couldn't allocate %zu-bytes mapped space: %s",
                      mmap_size, strerror(errno));
    return NULL;
  }

```

图3-12  __create_thread_mapped_space节选

**打印的错误信息与图[3-7]中进程逻辑地址占满导致的OOM错误信息吻合**，图[3-7]中错误信息＂ Try again＂说明系统全局错误标识errno为11(见图[3-10]系统错误定义_errdefs.h).

pthread_create过程中，节点４相关代码如下：

```java
 int rc = clone(__pthread_start, child_stack, flags, thread, &(thread->tid), tls, &(thread->tid));
  if (rc == -1) {
    int clone_errno = errno;
    // We don't have to unlock the mutex at all because clone(2) failed so there's no child waiting to
    // be unblocked, but we're about to unmap the memory the mutex is stored in, so this serves as a
    // reminder that you can't rewrite this function to use a ScopedPthreadMutexLocker.
    pthread_mutex_unlock(&thread->startup_handshake_mutex);
    if (thread->mmap_size != 0) {
      munmap(thread->attr.stack_base, thread->mmap_size);
    }
    __libc_format_log(ANDROID_LOG_WARN, "libc", "pthread_create failed: clone failed: %s", strerror(errno));
    return clone_errno;
  }

```

图3-13 pthread_create节选

**此处输出的错误日志"pthread_create failed: clone failed: %s"与我们线上发现的OOM二吻合**，图[3-6]中的错误描述＂ Out of memory＂说明系统全局错误标识errno为12(见图[3-10]系统错误定义_errdefs.h).

由此线上的**OOM二就是由于线程数的限制而在节点5 clone失败导致OOM**.

# 结论及监控


## 导致OOM发生的原因

综上，可以导致OOM的原因有以下几种：

1. **文件描述符(fd)数目超限**，即proc/pid/fd下文件数目突破/proc/pid/limits中的限制。可能的发生场景有：短时间内大量请求导致socket的fd数激增，大量（重复）打开文件等

2. **线程数超限**，即proc/pid/status中记录的线程数（threads项）突破/proc/sys/kernel/threads-max中规定的最大线程数。可能的发生场景有：app内多线程使用不合理，如多个不共享线程池的OKhttpclient等等

3. 传统的**java堆内存超限**，即申请堆内存大小超过了 Runtime.getRuntime().maxMemory()

4. （低概率）32为系统进程逻辑空间被占满导致OOM.

5. 其他




## 监控措施

可以利用linux的inotify机制进行监控:

- watch /proc/pid/fd来监控app打开文件的情况,

- watch /proc/pid/task来监控线程使用情况．




# Demo

POC(Proof of concept) 代码参见：[https://github.com/piece-the-world/OOMDemo](https://link.jianshu.com?t=https://github.com/piece-the-world/OOMDemo)
