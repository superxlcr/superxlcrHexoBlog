---
title: 关于Java泛型的一些思考
tags: [java,基础知识]
categories: [java]
date: 2018-10-29 21:00:56
description: 为什么引入泛型、泛型的使用、泛型的擦除
---

# 为什么引入泛型

泛型，即“参数化类型”。是Java 1.5引入的一种新特性。
为什么Java要引入泛型呢？我们看一下这个例子：
```java
List arrayList = new ArrayList();
arrayList.add("aaaa"); // it's ok
arrayList.add(100); // it's ok, too

for (int i = 0; i < arrayList.size(); i++) {
	String str = (String)arrayList.get(i);
	System.out.print(str);
}
```

毫无疑问，程序的运行结果会以崩溃结束：
java.lang.ClassCastException: java.lang.Integer cannot be cast to java.lang.String

这个例子中，我们构造了一个ArrayList列表，并往其中放入了一个String类型与一个Integer类型。而在使用时，我们都是以String类型的方式来强转取出，因此程序崩溃了。
为了让这种类似的容器类能够带上其内容的类型信息，解决类似的类型转换问题，泛型应运而生。
有了泛型之后，我们的例子可以优化成这个样子：
```java
List<String> arrayList = new ArrayList<String>();
arrayList.add("aaaa"); // it's ok
arrayList.add(100); // it's not ok !!!!

for (int i = 0; i < arrayList.size(); i++) {
	String str = (String)arrayList.get(i);
	System.out.print(str);
}
```

泛型能让编译器在编译阶段避免不少类似的类型转换问题。

# 泛型的使用

泛型的使用主要有三种：
- 泛型类：通常定义于各种容器类中，如List&lt;T&gt;
- 泛型方法：提供类型参数推断的功能，很方便
- 泛型构造方法：博主感觉与泛型方法类似，只是为了区分普通方法与构造方法，因此有了泛型方法与泛型构造方法

从Java的Type体系中我们可以看出三种泛型的定义：

Type是Java中所有类型的公共高级接口，其子类如下：
- ParameterizedType：参数化类型，即泛型；例如：List&lt;T&gt;、Map&lt;K,V&gt;等带有参数化的类
- TypeVariable：类型变量，即泛型中的变量；例如：T、K、V等变量，可以表示任何类；在这需要强调的是，TypeVariable代表着泛型中的变量，而ParameterizedType则代表整个泛型
- GenericArrayType：泛型数组类型，用来描述ParameterizedType、TypeVariable类型的数组；即List&lt;T&gt;[] 、T[]等
- Class：上三者不同，Class是Type的一个实现类，属于原始类型，是Java反射的基础，对Java类的抽象
- WildcardType：泛型表达式（或者通配符表达式），即？ extend Number、？ super Integer这样的表达式

其中，TypeVariable的接口定义如下：
```java
public interface TypeVariable<D extends GenericDeclaration> extends Type {
```

可以看到，它也是一个泛型类，它的泛型表示的是：它所表示的类型变量的具体种类，是泛型类、泛型方法还是泛型构造方法
GenericDeclaration的子类如下图所示：
![GenericDeclaration的子类](1.png)

从图中可以看出：Method、Constructor、Class分别表示类、方法以及构造方法三种泛型

更多泛型的使用与特性可以参考博主以前的文章：
[泛型常用特点，List&lt;String&gt;能否转为List&lt;Object&gt;](/2016/03/22/关于一些基础的Java问题的解答（七）/#泛型常用特点，List-lt-String-gt-能否转为List-lt-Object-gt)

# 泛型的擦除

泛型的擦除可谓是Java泛型的一大特点，那么什么是泛型擦除呢？
博主认为：泛型擦除即我们无法获得某个泛型实例对象的精确泛型参数

我们可以从两个现象来看这个特点：
- 泛型类中的编译器静态检查
- 泛型类反射

## 泛型类中的编译器静态检查

我们来看下下面的例子：
```java
class A {  
    public void fa() {};  
}  
class C <T> { // 擦除到Object  
    T t;  
    public void f(Object a) {  
        if (a instanceof T) {} // error，不知道具体的类型信息  
        T var = new T(); // error，不知道该类型是否有默认构造函数  
        T[] array = new T[1]; // error    
        t.fa(); // error  
    }  
}
class D <T extends A> { // 擦除到A  
    T t;  
    public void f(Object a) {  
        t.fa(); // this works  
    }  
}
```

当我们的编译器执行静态检查时，是没有运行时的信息的
因此，对于例子中的泛型类C，编译器只能把它的泛型参数T当做其上界来处理（即把类型信息擦除到边界），由于Object类为所有类的父类，因此编译器检查时T都是被当成Object类来处理的
当然这种情况我们可以使用extends关键字来改善，对于例子中的泛型类D，其泛型参数的上界就变为了类A

## 泛型类反射

要解释这个，我们需要先知道Java提供的反射功能是什么
Java的反射功能，是Java提供的一种允许我们在运行时，获取某个类（class）全部信息的能力，包括类的成员变量，方法，构造函数等等，同时还提供了一系列设置与调用的手段

我们来看下下面的例子：
```java
class A {}  
class B extends A {}  
class C extends B {};  
class D <T> {  
    T t;  
    D(T t) {  
        this.t = t;  
    }  
    public void f() {  
        System.out.println(Arrays.toString(this.getClass().getTypeParameters()));  
    }  
};  
  
  
public class Main {  
    public static void main(String[] args) {  
        D<A> a = new D<A>(new A());  
        D<B> b = new D<B>(new B());  
        D<C> c = new D<C>(new C());  
        a.f();  
        b.f();  
        c.f();  
    }  
}
```

D中的f方法通过获取D的Class类来获取其类型信息，其打印的结果如下：
![例子打印结果](2.jpg)

可以看到打印出来的结果并不是我们传人的泛型参数A、B、C三个类，这是为什么呢？
博主认为，这是因为这里用的是Java的反射机制获取D类的类型信息，而泛型参数实际上应该属于生成实例时传入的参数，因此我们只能获取**类**相关的信息，即TypeVariable（类型变量）以及它的名称T，而不能获取**实例**相关的泛型实际参数A、B、C

受这个特点影响比较大的要数如Gson等支持反序列化泛型对象的工具了：
```java
class Foo<T> {
  T value;
}
Gson gson = new Gson();
Foo<Bar> foo = new Foo<Bar>();
gson.toJson(foo); // May not serialize foo.value correctly

gson.fromJson(json, foo.getClass()); // Fails to deserialize foo.value as Bar
```

上面是一段Gson的官方例子，可以想象到从foo实例中通过反射获取Foo类中的泛型参数是失败的
由于泛型参数的缘故，我们只能拿到TypeVariable（类型变量）以及它的名称T，并不能获得实际的Bar类
官方介绍链接：
https://github.com/google/gson/blob/master/UserGuide.md#serializing-and-deserializing-generic-types

那么有没有什么方法可以缓解这种问题呢？
当然是有的，反射可以帮助我们获取**类型**相关的信息，因此这种情况我们只需要继承实现一个子类就可以了：
```java
Class Foo<T> {
	T value;
}
Class SubFoo extends Foo<String> {
	
}

Class clazz = SubFoo.class;
ParameterizedType type = (ParameterizedType)clazz.getGenericSuperclass();
type.getActualTypeArguments()[0]; // String Class
```

在继承实现子类的时候，由于我们显式地定义了SubFoo类的基类Foo的泛型参数，因此通过Java的反射，我们也能轻松的获取相关的泛型信息
可以看到Gson官方提供的解决方案也是类似的：
```java
Type fooType = new TypeToken<Foo<Bar>>() {}.getType();
gson.toJson(foo, fooType);

gson.fromJson(json, fooType);
```

通过继承创建匿名内部类TypeToken后，再使用Java的反射机制，获取相关的泛型信息(TypeToken内部的源码在此就不进行分析了)