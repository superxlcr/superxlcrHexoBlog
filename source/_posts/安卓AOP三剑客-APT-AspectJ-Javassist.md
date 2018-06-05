---
title: '安卓AOP三剑客:APT,AspectJ,Javassist'
tags: [android,AOP]
categories: [android]
date: 2018-05-01 16:32:35
description: （转载）APT、AspectJ、Javassist、AOP
---
本文转载自：https://www.jianshu.com/p/dca3e2c8608a?from=timeline

AOP:面向切面编程(Aspect-Oriented Programming)。如果说，OOP如果是把问题划分到单个模块的话，那么AOP就是把涉及到众多模块的某一类问题进行统一管理。

Android AOP就是通过预编译方式和运行期动态代理实现程序功能的统一维护的一种技术。利用AOP可以对业务逻辑的各个部分进行隔离，从而使得业务逻辑各部分之间的耦合度降低，提高程序的可重用性，提高开发效率。本文仅做知识介绍，相关详细内容不做过多描述，全部代码在项目[T-MVP](https://link.jianshu.com/?t=https://github.com/north2016/T-MVP)。

话不多说，先上图：
![APT,AspectJ,Javassist对应的编译时期](1.jpg)

AOP在Java后台，已经被各路大神研发出各种框架风生水起，例如SSH、SpringMVC等等殿堂级框架。在Android端，近年来也是异军突起。

# APT

代表框架：DataBinding,Dagger2, ButterKnife, EventBus3 、DBFlow、AndroidAnnotation

注解处理器 Java5 中叫APT(Annotation Processing Tool)，在Java6开始，规范化为 Pluggable Annotation Processing。Apt应该是这其中我们最常见到的了，难度也最低。定义编译期的注解，再通过继承Proccesor实现代码生成逻辑，实现了编译期生成代码的逻辑。

使用姿势 ：
1、建立一个java的Module,写一个继承AbstractProcessor的类
![AbstractProcessor](2.jpg)

2、在工具类里处理我们自定义的注解、生成代码：
![Processor](3.jpg)

3、在Gradle中添加 dependencies annotationProcessor project(':apt')
低版本需要使用第三方插件 apply plugin: 'com.neenbedankt.android-apt'
然后apt project(':apt')

生成的源代码在build/generated/source/apt下可以看到
![apt生成代码的路径](4.jpg)

难点：
就apt本身来说没有任何难点可言，难点一在于设计模式和解耦思想的灵活应用，二在与代码生成的繁琐，你可以手动字符串拼接，当然有更高级的玩法用squareup的javapoet，用建造者的模式构建出任何你想要的源代码。
想详细了解可以看官网或这篇博客：
[Android 利用 APT 技术在编译期生成代码](http://brucezz.itscoder.com/use-apt-in-android)

优点：
它的强大之处无需多言，看代表框架的源码，你可以学到很多新姿势。总的一句话：它可以做任何你不想做的繁杂的工作，它可以帮你写任何你不想重复代码。懒人福利，老司机必备神技，可以提高车速，让你以任何姿势漂移。它可以生成任何源代码供你在任何地方使用，就像剑客的剑，快疾如风，无所不及。

# AspectJ

代表框架： Hugo(Jake Wharton)

AspectJ支持编译期和加载时代码注入，在开始之前，我们先看看需要了解的词汇：
**Advice（通知）**: 典型的 Advice 类型有 before、after 和 around，分别表示在目标方法执行之前、执行后和完全替代目标方法执行的代码。

**Joint point（连接点）**: 程序中可能作为代码注入目标的特定的点和入口。

**Pointcut（切入点）**: 告诉代码注入工具，在何处注入一段特定代码的表达式。

**Aspect（切面）**: Pointcut 和 Advice 的组合看做切面。例如，在本例中通过定义一个 pointcut 和给定恰当的advice，添加一个了内存缓存的切面。

**Weaving（织入）**: 注入代码（advices）到目标位置（joint points）的过程。

下面这张图简要总结了一下上述这些概念。
![AOP概念图](5.png)

使用姿势：
1、建立一个android lib Module，定义一个切片，处理自定义注解，和添加切片逻辑
![AspectJ](6.jpg)

2、自定义一个gradle插件，使用 AspectJ 的编译器（ajc，一个java编译器的扩展),对所有受 aspect 影响的类进行织入，在 gradle 的编译 task 中增加额外配置，使之能正确编译运行。
![AspectjPlugin](7.jpg)

3、在grade中apply plugin:com.app.plugin.AspectjPlugin

生成的class文件在build/intermediates/classes下可以看到
![Aspectj编织后的文件路径](8.jpg)

难点：
AspectJ语法比较多，但是掌握几个简单常用的，就能实现绝大多数切片，完全兼容Java（纯Java语言开发，然后使用AspectJ注解，简称@AspectJ。）想详细了解可以看官网或这篇博客：
[深入理解AndroidAOP](https://link.jianshu.com/?t=http://blog.csdn.net/innost/article/details/49387395)

优点：
AspectJ除了hook之外，AspectJ还可以为目标类添加变量,接口。另外，AspectJ也有抽象，继承等各种更高级的玩法。它能够在编译期间直接修改源代码生成class，强大的团战切入功能，指哪打哪，鞭辟入里。有了此神器，编程亦如庖丁解牛，游刃而有余。

# Javassist

代表框架：热修复框架HotFix 、Savior（InstantRun）等

Javassist作用是在编译器间修改class文件，与之相似的ASM（热修复框架女娲）也有这个功能，可以让我们直接修改编译后的class二进制代码，首先我们得知道什么时候编译完成，并且我们要赶在class文件被转化为dex文件之前去修改。在Transfrom这个api出来之前，想要在项目被打包成dex之前对class进行操作，必须自定义一个Task，然后插入到predex或者dex之前，在自定义的Task中可以使用javassist或者asm对class进行操作。而Transform则更为方便，Transfrom会有他自己的执行时机，不需要我们插入到某个Task前面。Tranfrom一经注册便会自动添加到Task执行序列中，并且正好是项目被打包成dex之前。

使用姿势
1、定义一个buildSrc module添加自定义Plugin
![自定义Plugin](9.jpg)

2、自定义Transform
![自定义Transform](10.jpg)

3、在Transform里处理Task，通过inputs拿到一些东西，处理完毕之后就输出outputs，而下一个Task的inputs则是上一个Task的outputs。
![处理Task](11.jpg)

4、使用Javassist操作字节码，添加新的逻辑或者修改原有逻辑
![Javassist操作字节码](12.jpg)

5、在grade中apply plugin:com.app.plugin.MyPlugin

修改后的class文件在build/intermediates/transforms/MyTrans下可以看到
![Javassist修改后的文件路径](13.jpg)

难点：
相比ASM，Javassist对java极度友好的api更容易快速上手，难点在思想的应用，小到切片逻辑的控制，如本例中的性能log打印日志，大到宏观的热修复，插件化中对preDex的操作修改，剑客精神到了这一层级，已经是上帝视角，无所不能。

优点：
由于Javassist可以直接操作修改编译后的字节码，直接绕过了java编译器，所以可以做很多突破限制的事情，例如，跨dex引用，解决热修复中CLASS_ISPREVERIFIED的问题。

想详细了解可以看官网或这篇博客：
[Android热补丁动态修复技术](https://blog.csdn.net/u010386612/article/details/51131642)

[基于Instant Run思想的HotFix方案实现](https://halfstackdeveloper.github.io/2016/09/23/%E5%9F%BA%E4%BA%8EInstant-Run%E6%80%9D%E6%83%B3%E7%9A%84HotFix%E6%96%B9%E6%A1%88%E5%AE%9E%E7%8E%B0/)

# AOP

AOP技术常用在以下方面：
1. 日志记录：业务埋点
2. 持久化
3. 性能监控：性能日志
4. 数据校验：方法的参数校验
5. 缓存：内存缓存和持久缓存
6. 权限检查：业务权限（如登陆，或用户等级）、系统权限（如拍照定位）
7. 异常处理

利用AOP技术将这些功能代码从业务逻辑代码中划分出来，通过对这些行为的分离，可以将它们独立到非业务逻辑。无论是日后新增，或是修改，都手到擒来易如反掌。

例如新的tmvp的demo中 apt用于生成实例化工厂，替换掉(对于小项目来说)繁杂冗余的Dagger2,实现了初始化功能的aop ；aspectj 的切片主要用在缓存和日志，用注解实现方法级别的内存缓存和方法耗时日志；Javassist 这里只是做了个示例，也是通过注解实现方法耗时日志的自动打印功能，当然这些都只是AOP的九牛一毛，AOP还可以做很多事，弥补OOP的不足，把所有跨对象的横切面关注点的功能都可以提取出来用AOP去实现 ，好处显而易见，将来要改的地方永远只有一处，而不是像OOP那样牵扯很多模块很多代码很多类。

当然还有更多未知的可能，需要等各位大侠来研究开发，让Aop在Android上的应用更加广泛。