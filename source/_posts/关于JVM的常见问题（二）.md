---
title: 关于JVM的常见问题（二）
tags: [jvm,java,基础知识]
categories: [jvm]
date: 2016-04-01 10:56:14
description: 常见的GC收集器以及特点、Minor GC与Full GC执行时机、类加载的五个过程、类加载双亲委派模型、静态分派与动态分派
---
上一篇文章的传送门：[关于JVM的常见问题（一）](/2016/03/28/关于JVM的常见问题（一）/)

# 常见的GC收集器以及特点

常见的GC收集器如下图所示，连线代表可搭配使用：
![常见的GC收集器示意图](1.png)

## Serial收集器（串行收集器）

用于新生代的单线程收集器，收集时需要暂停所有工作线程（Stop the world）。优点在于：简单高效，单个CPU时没有线程交互的开销，堆较小时停顿时间不长。常与Serial Old 收集器一起使用，示意图如下所示：
![Serial收集器示意图](2.jpg)

## ParNew收集器（parallel new 收集器，新生代并行收集器）

Serial收集器多线程版本，除了使用多线程外和Serial收集器一模一样。常与Serial Old 收集器一起使用，示意图如下：
![ParNew收集器示意图](3.jpg)

## Parallel Scavenge收集器
与ParNew收集器一样是一款多线程收集器，其特点在于关注点与别的GC收集器不同：一般的GC收集器关注于缩短工作线程暂停的时间，而该收集器关注于吞吐量，因此也被称为吞吐量优先收集器。（吞吐量 = 用户运行代码时间 /  (用户运行代码时间 + 垃圾回收时间)）高吞吐量与停顿时间短相比主要强调任务快完成，因此常和Parallel Old 收集器一起使用（没有Parallel Old之前与Serial Old一起使用），示意图如下：
![Parallel Scavenge收集器示意图](4.jpg)

## Serial Old收集器
Serial收集器的年老代版本，不再赘述。

## Parallel Old收集器
年老代的并行收集器，在JDK1.6开始使用。

## CMS收集器（Concurrent Mark Sweep，并发标记清除收集器）
CMS收集器是一个年老代的收集器，是以最短回收停顿时间为目标的收集器，其示意图如下所示：
![CMS收集器](5.jpg)

CMS收集器基于标记清除算法实现，主要分为4个步骤：
- 初始标记，需要stop the world，标记GC Root能关联到的对象，速度快
- 并发标记，对GC Root执行可达性算法
- 重新标记，需要stop the world，修复并发标记时因用户线程运行而产生的标记变化，所需时间比初始标记长，但远比并发标记短
- 并发清理

CMS收集器的缺点在于：
- 其对于CPU资源很敏感。在并发阶段，虽然CMS收集器不会暂停用户线程，但是会因为占用了一部分CPU资源而导致应用程序变慢，总吞吐量降低。其默认启动的回收线程数是（cpu数量+3）/4，当cpu数较少的时候，会分掉大部分的cpu去执行收集器线程
- 无法处理浮动垃圾，浮动垃圾即在并发清除阶段因为是并发执行，还会产生垃圾，这一部分垃圾即为浮动垃圾，要等下次收集
- CMS收集器使用的是标记清除算法，GC后会产生碎片

## G1收集器（Garbage First收集器）

相比CMS收集器，G1收集器主要有两处改进：
- 使用标记整理算法，确保GC后不会产生内存碎片
- 可以精确控制停顿，允许指定消耗在垃圾回收上的时间

G1收集器可以实现在基本不牺牲吞吐量的前提下完成低停顿的内存回收，这是由于它能够极力地避免全区域的垃圾收集，之前的收集器进行收集的范围都是整个新生代或老年代，而G1将整个Java堆（包括新生代、老年代）划分为多个大小固定的独立区域（Region），并且跟踪这些区域里面的垃圾堆积程度，在后台维护一个优先列表，每次根据允许的收集时间，优先回收垃圾最多的区域（这就是Garbage First名称的来由）。区域划分及有优先级的区域回收，保证了G1收集器在有限的时间内可以获得最高的收集效率。

# Minor GC与Full GC执行时机

Minor GC也叫Young GC，当年轻代内存满的时候会触发，会对年轻代进行GC
Full GC也叫Major GC，当年老代满的时候会触发，当我们调用System.gc时也可能会触发，会对年轻代和年老代进行GC

# 类加载的五个过程

JVM把class文件加载的内存，并对数据进行校验、转换解析和初始化，最终形成JVM可以直接使用的Java类型的过程就是加载机制。
类从被加载到虚拟机内存中开始，到卸载出内存为止，它的生命周期包括了：加载(Loading)、验证(Verification)、准备(Preparation)、解析(Resolution)、初始化(Initialization)、使用(Using)、卸载(Unloading)七个阶段，其中验证、准备、解析三个部分统称链接。

## 加载

在加载阶段，虚拟机需要完成以下事情：
1. 通过一个类的权限定名来获取定义此类的二进制字节流
2. 将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构
3. 在java堆中生成一个代表这个类的java.lang.Class对象，作为方法去这些数据的访问入口

## 验证

在验证阶段，虚拟机主要完成：
1. 文件格式验证：验证class文件格式规范
2. 元数据验证：这个阶段是对字节码描述的信息进行语义分析，以保证起描述的信息符合java语言规范要求
3. 字节码验证：进行数据流和控制流分析，这个阶段对类的方法体进行校验分析，这个阶段的任务是保证被校验类的方法在运行时不会做出危害虚拟机安全的行为
4. 符号引用验证：符号引用中通过字符串描述的全限定名是否能找到对应的类、符号引用类中的类，字段和方法的访问性(private、protected、public、default)是否可被当前类访问

## 准备

准备阶段是正式为类变量（被static修饰的变量）分配内存并设置变量初始值（0值）的阶段，这些内存都将在方法区中进行分配

## 解析

解析阶段是虚拟机将常量池内的符号引用替换为直接引用的过程
常见的解析有四种：
1. 类或接口的解析
2. 字段解析
3. 类方法解析
4. 接口方法解析

## 初始化

初始化阶段才真正开始执行类中定义的java程序代码，初始化阶段是执行类构造器<clinit>()方法的过程

# 类加载双亲委派模型

JVM运行的是Java字节码，而Java字节码需要通过类加载器找到对应的.class文件，并加载在内存中生成Class类供我们使用。类加载器的主要工作流程如下：

loadClass（Java方法，确认由哪个加载器进行加载）->findClass（Java方法，寻找.class文件）->defineClass（native方法，通过.class文件在内存中生成Class类供使用）

JVM的类加载有多个， 不同的类加载器用于搜索不同位置的.class文件，Java中的类加载器体系结构如下：
![Java类加载器体系示意图](6.jpg)

- 启动类加载器（Bootstrap ClassLoader）：是用本地代码实现的类装入器，它负责将 <Java_Runtime_Home>/lib下面的类库加载到内存中（比如rt.jar）。由于引导类加载器涉及到虚拟机本地实现细节，开发者无法直接获取到启动类加载器的引用，所以不允许直接通过引用进行操作.**该类加载器代码在JVM内核中，由native代码实现。**
- 标准扩展类加载器：是由 Sun 的 ExtClassLoader（sun.misc.Launcher$ExtClassLoader）实现的。它负责将< Java_Runtime_Home >/lib/ext或者由系统变量 java.ext.dir指定位置中的类库加载到内存中。开发者可以直接使用标准扩展类加载器。**该类加载器由Java代码实现。**
- 应用程序类加载器：由sun.misc.Launcher$AppClassLoader实现，负责加载用户类路径classpath上所指定的类库，是类加载器ClassLoader中的getSystemClassLoader()方法的返回值，开发者可以直接使用应用程序类加载器，如果程序中没有自定义过类加载器，该加载器就是程序中默认的类加载器。**该类加载器由Java代码实现。**

值得注意的是：上述三个JDK提供的类加载器虽然是父子类加载器关系，但是没有使用继承，而是使用了组合关系。

从JDK1.2开始，JVM规范推荐开发者使用双亲委派模式(ParentsDelegation Model)进行类加载，其加载过程如下：
1. 如果一个类加载器收到了类加载请求，它首先不会自己去尝试加载这个类，而是把类加载请求委派给父类加载器去完成
2. 每一层的类加载器都把类加载请求委派给父类加载器，直到所有的类加载请求都传递给顶层的启动类加载器
3. 如果顶层的启动类加载器无法完成加载请求，则子类加载器会尝试去加载，如果连最初发起类加载请求的类加载器也无法完成加载请求时，将会抛出ClassNotFoundException，而不再调用其子类加载器去进行类加载

采用双亲委派模型的好处在于：使得java类随着它的类加载器一起具备了一种带有优先级的层次关系，越是基础的类，越是被上层的类加载器进行加载，保证了java程序的稳定运行。

# 静态分派与动态分派

要理解分派，我们先来理解Java中的两种类型：
- 静态类型：变量声明时的类型
- 实际类型：变量实例化时采用的类型

例子如下：
```java
class Car {}  
class Bus extends Car {}  
public class Main {  
    public static void main(String[] args) throws Exception {  
        // Car 为静态类型，Bus 为实际类型  
        Car car = new Bus();  
    }  
}  
```

## 静态分派

所有依赖静态类型来定位方法执行版本的分派动作称为静态分派，其典型应用是方法重载（重载是通过参数的静态类型而不是实际类型来选择重载的版本的）。
举例Java代码如下：
```java
class Car {}  
class Bus extends Car {}  
class Jeep extends Car {}  
public class Main {  
    public static void main(String[] args) throws Exception {  
        // Car 为静态类型，Car 为实际类型  
        Car car1 = new Car();  
        // Car 为静态类型，Bus 为实际类型  
        Car car2 = new Bus();  
        // Car 为静态类型，Jeep 为实际类型  
        Car car3 = new Jeep();  
          
        showCar(car1);  
        showCar(car2);  
        showCar(car3);  
    }  
    private static void showCar(Car car) {  
        System.out.println("I have a Car !");  
    }  
    private static void showCar(Bus bus) {  
        System.out.println("I have a Bus !");  
    }  
    private static void showCar(Jeep jeep) {  
        System.out.println("I have a Jeep !");  
    }  
}  
```

代码输出如下：
![静态分派代码输出](7.jpg)

从上面的例子我们可以看出重载调用的具体方法版本是由静态类型来决定的。

## 动态分派

与静态分派类似，动态分派指在在运行期根据实际类型确定方法执行版本，其典型应用是方法重写（即多态）。
举例Java代码如下：
```java
class Car {  
    public void showCar() {  
        System.out.println("I have a Car !");  
    }  
}  
class Bus extends Car {  
    public void showCar() {  
        System.out.println("I have a Bus !");  
    }  
}  
class Jeep extends Car {  
    public void showCar() {  
        System.out.println("I have a Jeep !");  
    }  
}  
public class Main {  
    public static void main(String[] args) throws Exception {  
        // Car 为静态类型，Car 为实际类型  
        Car car1 = new Car();  
        // Car 为静态类型，Bus 为实际类型  
        Car car2 = new Bus();  
        // Car 为静态类型，Jeep 为实际类型  
        Car car3 = new Jeep();  
          
        car1.showCar();  
        car2.showCar();  
        car3.showCar();  
    }  
}  
```

运行结果如下：
![动态分派代码输出](8.jpg)

可以看出来重写是一个根据实际类型决定方法版本的动态分派过程。