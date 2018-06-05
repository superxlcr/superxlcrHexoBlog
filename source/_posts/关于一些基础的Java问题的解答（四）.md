---
title: 关于一些基础的Java问题的解答（四）
tags: [java,基础知识]
categories: [java]
date: 2016-03-18 09:19:17
description: Java面向对象的三个特征与含义、Override和Overload的含义和区别、Interface与abstract类的区别、Static class 与non static class的区别、java多态的实现原理
---
上一篇文章的传送门：[关于一些基础的Java问题的解答（三）](/2016/03/17/关于一些基础的Java问题的解答（三）/)

# Java面向对象的三个特征与含义

java中的面向对象的三大基本特征分别是：封装、继承、多态：
1. 封装：把过程和数据包围起来，对数据的访问只能通过已定义的界面，主要是方便类的修改与拓展
2. 继承：对象的一个新类可以从现有的类中派生，这个过程称为类继承。通常继承把共性放到父类，特性放到子类。继承性很好的解决了软件的可重用性问题
3. 多态：指允许不同类的对象对同一消息作出响应，多态性语言具有灵活、抽象、行为共享、代码共享的优势，很好的解决了应用程序函数同名问题

# Override和Overload的含义和区别

Override即重写，也叫覆盖，即在子类中拥有与父类非private方法一模一样的方法（返回值，参数，方法名均相同，在Java SE5中引入了协变类型，协变类型允许覆盖的方法的返回值为原返回值的子类型），且子类方法的访问修饰权限不能少于父类，则对子类调用该方法时会调用子类方法代替父类方法。
Overload即重载，是指在同一类中拥有多个方法名相同，参数不同（参数类型或个数），返回值可相同可不同的方法。调用方法时会根据传入参数选择合适方法。
两者区别：
1. Override描述子类与父类方法的关系，Overload描述同一个类中方法的关系
2. Override的方法必须完全相同，权限不少于父类，Overload方法名必须相同，参数必须不同，返回值可同可不同

# Interface与abstract类的区别

Interface即接口，接口可以有变量，但接口的变量均为public static final的编译期常量，接口中的方法均为public abstract的公开抽象方法，一个类可以实现多个接口。
abstract类即抽象类，抽象类有关键字abstract修饰，抽象类不能实例化只能被继承。抽象类中一般含有抽象方法（为public或protected，private无法继承），其他方面与一般类区别不大。
两者区别：
1. 接口的变量为public static final的编译期常量，抽象类的变量没有限制
2. 接口中的方法为public abstract的公开抽象方法，抽象类的抽象方法不为private即可，且允许有一般方法
3. 一个类可以实现多个接口，一个类最多只能继承一个抽象类

# Static class 与non static class的区别

首先由于顶级类（top level class）是不能为静态的，因此此处讨论的是内部类（nested class）。
静态内部类和非静态内部类的区别主要有以下几点：
1. 静态内部类不需要持有指向外部类的引用，但非静态内部类需要持有对外部类的引用
2. 静态内部类不能访问外部类的非静态成员，只能访问外部类的静态成员，而非静态内部类可以访问外部类所有成员
3. 静态内部类与非静态内部类创建方法不同，一个非静态内部类不能脱离外部类实体被创建

创建静态内部类和非静态内部类的代码：
```java
public class Test {  
    public static void main(String[] args) {  
        // 创建静态内部类  
        Test.staticClass class1 = new Test.staticClass();  
        // 创建非静态内部类  
        // 不能通过编译！  
//      Test.noStaticClass class2 = new Test.noStaticClass();  
        // 正确创建非静态内部类方法  
        Test.noStaticClass class2 = new Test().new noStaticClass();  
    }  
      
    static class staticClass {};  
    class noStaticClass {};  
}  
```

# java多态的实现原理

多态，也叫作动态绑定、后期绑定或运行时绑定，指允许不同类的对象对同一消息作出响应。下面介绍一些和绑定相关的概念：
1. 绑定：指的是将一个方法调用和一个方法主体关联起来的过程
2. 前期绑定：在程序执行前进行绑定（一般由编译器和连接程序实现，C++默认为前期绑定）
3. 后期绑定：也叫作动态绑定或运行时绑定，在程序运行时通过识别对象的类型，从而调用恰当的方法（Java除static和final方法外，其他均为后期绑定。C++通过virtual关键字实现虚函数，从而实现后期绑定）

来看一段Java代码：
```java
public class Test {  
    static class A {  
        public void sayHi() {  
            System.out.println("Hi, I am A");  
        }  
    }  
      
    static class B extends A {  
        @Override  
        public void sayHi() {  
            System.out.println("Hi, I am B");  
        }  
    }  
      
    static class C extends A {  
        @Override  
        public void sayHi() {  
            System.out.println("Hi, I am C");  
        }  
    }  
      
    public static void saySomething(A a) {  
        a.sayHi();  
    }  
      
    public static void main(String[] args) {  
        saySomething(new A());  
        saySomething(new B());  
        saySomething(new C());  
    }  
      
}  
```
看到第23行的方法，从编译器给出的信息我们只能知道a是一个A类的引用，但无法确定a是A类的实例，还是其子类（B或C）的实例（导出类可以通过向上转型，把对自身的引用视为对其基类的引用），因此如果通过前期绑定，我们并不能实现多态。但是如果我们使用的是后期绑定，我们就可以在程序运行时先确定传入实例的具体类型，再根据其具体类型来调用对应的方法，就可以实现多态。