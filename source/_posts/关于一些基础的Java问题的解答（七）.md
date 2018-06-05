---
title: 关于一些基础的Java问题的解答（七）
tags: [java,基础知识]
categories: [java]
date: 2016-03-22 15:04:54
description: 反射的作用与原理、泛型常用特点，List&ltString&gt能否转为List&ltObject&gt、解析XML的几种方式的原理与特点：DOM、SAX、PULL、Java与C++对比、Java1.5、1.7与1.8新特性
---
上一篇文章的传送门：[关于一些基础的Java问题的解答（六）](/2016/03/20/关于一些基础的Java问题的解答（六）/)

# 反射的作用与原理

简单的来说，反射机制其实就是指程序在运行的时候能够获取自身的信息。如果知道一个类的名称或者它的一个实例对象， 就能把这个类的所有方法和变量的信息(方法名，变量名，方法，修饰符，类型，方法参数等等所有信息)找出来。如果明确知道这个类里的某个方法名+参数个数 类型，还能通过传递参数来运行那个类里的那个方法，这就是反射。
在Java中，Class类与java.lang.reflect类库一起对反射的概念提供了支持，该类库包含了Field、Method以及Constructor类（每个类都实现了Member接口）。我们知道对RTTI（运行时类型识别）来说，编译器在编译时打开和检查.class文件。而对于反射机制来说，.class文件在编译时是不可获取的，所以是在运行时打开和检查.class文件的。
说了这么多，反射究竟有什么用呢？我们来看一下以下的例子：
```java
class A {  
    private int varA;  
    public void myPublicA() {  
        System.out.println("I am public in A !");  
    };  
    private void myPrivateA() {  
        System.out.println("I am private in A !");  
    };  
}  
  
class B extends A {  
    public int varB;  
    public void myPublicB(){};  
}  
  
public class Main {  
          
    public static void main(String[] args) throws Exception {  
        B b = new B();  
        // 子类方法  
        Method methods[] = b.getClass().getMethods();  
        for (Method method : methods)  
            System.out.println(method);  
        System.out.println("");  
        // 子类变量  
        Field fields[] = b.getClass().getFields();  
        for (Field field : fields)  
            System.out.println(field);  
        // 基类  
        System.out.println("\n" + b.getClass().getSuperclass() + "\n");  
        // 基类private方法也不能避免  
        Class superClass = b.getClass().getSuperclass();  
        methods = superClass.getDeclaredMethods();  
        for (Method method : methods) {  
            System.out.println(method);  
            method.setAccessible(true);  
            // 实例化A来调用private方法！  
            method.invoke(superClass.newInstance(), null);  
        }  
    }  
}  
```

在上面的例子中，我们用一个子类B，通过反射找到了他的数据域、与方法，还找到了他的基类A。更甚者，我们实例化了基类A，还调用了A里面的所有方法，甚至是private方法。从以上的例子相信大家都感受到了反射的威力了，运用使用class对象和反射提供的方法我们可以轻易的获取一个类的所有信息，包括被封装隐藏起来的信息，不仅如此我们还可以调用获取的信息来构造实例和调用方法。以上的展示只是反射的冰山一角，反射在动态代理和调用隐藏API等黑科技方面还发挥着重要的作用，博主在此不做深入探讨。

# 泛型常用特点，List<String>能否转为List<Object>

泛型是Java SE5引入的一种新特性，泛型实现了**参数化类型**概念，使得我们的代码可以应用于更多类型，更多场景。在以往的J2SE中，没有泛型的情况下，通常是使用Object类型来进行多种类型数据的操作。这个时候操作最多的就是针对该Object进行数据的强制转换，而这种转换是基于开发者对该数据类型明确的情况下进行的（例如将Object型转换为String型）。如果类型不一致，编译器在编译过程中不会报错，但在运行时会出错。相比之下，使用泛型的好处在于，它在编译的时候进行类型安全检查，并且在运行时所有的转换都是强制的，隐式的，大大提高了代码的重用率。
先来回答List<String>能否转为List<Object>的问题，答案是不行的，因为String的list不是Object的list，String的list持有String类和其子类型，Object的list持有任何类型的Object，String的list在类型上不等价于Object的list。但List<String>可以转为List<? extends Object>，Java代码如下：
```java
List<String> listString = new ArrayList<>();  
// error : Type mismatch: cannot convert from List<String> to List<Object>  
List<Object> listObject = listString;  
// it's ok !  
List<? extends Object> listExtendsObject = listString;  
// error : The method add(capture#1-of ? extends Object) in the type List<capture#1-of ? extends Object> is not applicable for the   
// arguments (String)  
listExtendsObject.add("string");  
// it's ok !  
listExtendsObject.add(null);  
// it's ok !  
Object object = listExtendsObject.get(0);  
```

接下来讲讲泛型常用的特点，利用泛型我们可以实现以下内容：

## 带参数类型的类，泛型类

类的参数类型写在类名的后面，多个参数类型用逗号分隔，定义完类型参数后，我们可以在定义位置之后的类的几乎任意地方（静态块，静态属性，静态方法除外）使用类型参数：
```java
class A<T,S> {  
  
}  
```

## 带参数类型的方法，泛型方法

除了可以定义泛型类，我们还可以把泛型应用于方法之上，要定义泛型方法只需把泛型参数列表置于返回值之前：
```java
public <T> void f(T x) {  
      
}  
```

使用泛型方法时通常不必指明参数类型，编译器会为我们找出具体类型，这称为类型参数推断。

## 关键字

泛型还有两个重要的关键字extends和super，这两个关键字用于限制泛型的范围。extends把类型参数限制为某个类的子类：
```java
class A {}  
class B extends A {}  
class C {};  
//代表了T为A或A的子类  
class D <T extends A> {};  
  
  
public class Main {  
    public static void main(String[] args) {  
        D<A> a;  
        D<B> b;  
        // no work!  
        D<C> c;  
    }  
}  
```

super与extends相反，把类型参数限制为某个类的父类。（此处博主研究不够深入，故对super关键字不够了解，在此不深入讨论）

## 擦除

Java的泛型不是完美的泛型，Java的泛型为了考虑兼容性的问题，使用了擦除来实现。看一下下面的例子：
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
![运行时泛型获取结果](1.jpg)

并不是我们传入的参数A、B、C，真是太失望了。这就是Java泛型擦除的特点，残酷的现实告诉我们在泛型代码的内部，我们无法获得任何有关泛型的参数类型信息，擦除会把类的类型信息给擦除到它的边界（如果有多个边界会擦除到第一个）。也就是说，对于上面例子中的T，我们只能把其当做Object类来处理（Object类为所有类的父类）。擦除使得所有与类型信息相关的操作都无法在泛型代码中进行，extends会稍微改善一点这种情况：
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
};  
class D <T extends A> { // 擦除到A  
    T t;  
    public void f(Object a) {  
        t.fa(); // this works  
    }  
};  
```

# 解析XML的几种方式的原理与特点：DOM、SAX、PULL

## DOM

DOM解析方法首先把xml文件读取到内存中，保存为节点树的形式，然后我们使用其API来读取树上的节点的信息。由于DOM解析xml文件时需要将其载入到内存，故xml文件较大时或内存较小的设备不适用该方法。
使用DOM解析xml主要步骤如下：
1. 使用DocumentBuilderFactory.newInstance方法获取DOM工厂实例
2. 使用工厂的newDocumentBuilder方法获取builder
3. 使用builder的parse方法解析xml获取生成的Document对象
4. 使用Document的getDocumentElement方法获取根节点
5. 调用根节点的getChildNodes方法遍历子节点

## SAX

与DOM不同，SAX全称为Simple API for XML ，是基于事件驱动的解析手段。对于SAX而言分析xml能够立即开始，而不用等待所有的数据被处理。而且，由于SAX只是在读取数据时检查数据，因此不需要将数据存储在内存中。一般来说SAX解析比DOM解析快许多，但由于SAX解析xml文件是一次性处理，因此相对DOM而言没有那么灵活方便。
使用SAX解析主要步骤如下：
1. 调用SAXParserFactory.newInstance获取SAX工厂实例
2. 调用工厂的newSAXParser方法获取解析器
3. 调用解析器的getXMLReader获取事件源reader
4. 调用setContentHandler方法为事件源reader设置处理器
5. 调用parse方法开始解析数据

## PULL

与SAX类似，Pull也是一种基于事件驱动的xml解析器。与SAX不同在于Pull让我们手动控制解析进度，通过返回eventType来让我们自行处理xml的节点，而不是调用回调函数，eventType有如下几种：
- 读取到xml的声明返回      START_DOCUMENT
- 读取到xml的结束返回       END_DOCUMENT 
- 读取到xml的开始标签返回 START_TAG 
- 读取到xml的结束标签返回 END_TAG
- 读取到xml的文本返回       TEXT

使用Pull解析的主要步骤如下：
1. 使用XmlPullParserFactory.newInstance方法获取Pull工厂
2. 调用工厂newPullParser方法返回解析器
3. 使用解析器setInput方法设置解析文件
4. 调用next方法解析下一行，调用getEventType方法获取当前的解析情况

3种解析方法的代码如下：
DOM与SAX：
```java
public static void main(String[] args) throws Exception {  
        // 桌面的xml文件,文件内容如下  
        // <all name="testData">  
        // <item first="1" second="A" third="一"/>  
        // <item first="2" second="B" third="二"/>  
        // <item first="3" second="C" third="三"/>  
        // <item first="4" second="D" third="四"/>  
        // <item first="5" second="E" third="五"/>  
        // </all>  
        File file = new File("C:/Users/Administrator/Desktop/test.xml");  
        // 文件流  
        FileInputStream fis = new FileInputStream(file);  
  
        {  
            // DOM解析xml  
            System.out.println("DOM:");  
            // 获取DOM工厂实例  
            DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();  
            // 生成builder  
            DocumentBuilder builder = factory.newDocumentBuilder();  
            // 解析文件  
            Document document = builder.parse(file);  
            // 获取根节点  
            Element element = document.getDocumentElement();  
            System.out.println(element.getTagName() + " " + element.getAttribute("name"));  
            // 获取子节点列表  
            NodeList nodeList = element.getChildNodes();  
            for (int i = 0; i < nodeList.getLength(); i++) {  
                Node node = nodeList.item(i);  
                // 节点类型为元素节点  
                if (node.getNodeType() == Node.ELEMENT_NODE) {  
                    Element child = (Element) node;  
                    // 输出标签名和元素内容  
                    System.out.println(child.getTagName() + " " + child.getAttribute("first") + " "  
                            + child.getAttribute("second") + " " + child.getAttribute("third"));  
                }  
            }  
        }  
        System.out.println(""); // empty line  
        {  
            // SAX解析xml  
            System.out.println("SAX:");  
            // 获取SAX工厂实例  
            SAXParserFactory factory = SAXParserFactory.newInstance();  
            // 获取SAX解析器  
            SAXParser parser = factory.newSAXParser();  
            // 获取reader  
            XMLReader reader = parser.getXMLReader();  
            // 设置解析源和处理器  
            reader.setContentHandler(new MySAXHandler()); // 在parse之前设置  
            reader.parse(new InputSource(fis));  
        }  
    }  
    // 自定义SAX处理器  
    static class MySAXHandler extends DefaultHandler {  
        @Override  
        public void startDocument() throws SAXException {  
            // 解析文档开始时调用  
            super.startDocument();  
        }  
        @Override  
        public void startElement(String uri, String localName, String qName, Attributes attributes)  
                throws SAXException {  
            // 解析元素开始时调用  
            // 打印元素名  
            System.out.print(qName);  
            // 打印元素属性  
            for (int i = 0; i < attributes.getLength(); i++)  
                System.out.print(" " + attributes.getValue(i));  
            System.out.println("");  
            super.startElement(uri, localName, qName, attributes);  
        }  
        @Override  
        public void endElement(String uri, String localName, String qName) throws SAXException {  
            // 解析元素结束时调用  
            super.endElement(uri, localName, qName);  
        }  
        @Override  
        public void endDocument() throws SAXException {  
            // 解析文档结束时调用  
            super.endDocument();  
        }  
    }  
```

Pull（此为Android解析中国天气网省份信息xml文件的例子）：
```java
XmlPullParserFactory factory = XmlPullParserFactory  
        .newInstance();  
XmlPullParser xmlPullParser = factory.newPullParser();  
// use gb2312 encode  
ByteArrayInputStream is = new ByteArrayInputStream(  
        response.getBytes("GB2312"));  
xmlPullParser.setInput(is, "GB2312");  
int eventType = xmlPullParser.getEventType();  
String provinceName = "";  
String provinceCode = "";  
while (eventType != XmlPullParser.END_DOCUMENT) {  
    String nodeName = xmlPullParser.getName();  
    switch (eventType) {  
    // start parse node  
    case XmlPullParser.START_TAG:  
        if ("city".equals(nodeName)) {  
            provinceName = xmlPullParser.getAttributeValue("",  
                    "quName");  
            provinceCode = xmlPullParser.getAttributeValue("",  
                    "pyName");  
            Province province = new Province();  
            province.setProvinceName(provinceName);  
            province.setProvinceCode(provinceCode);  
            coolWeatherDB.saveProvince(province);  
        }  
        break;  
    default:  
        break;  
    }  
    eventType = xmlPullParser.next();  
}
```

本人对三种解析方法的总结如下：
- 需要解析小的xml文件，需要重复解析xml文件或需要对xml文件中的节点进行删除修改排序等操作：使用DOM
- 需要解析较大的xml文件，只需要解析一次的xml文件：使用SAX或Pull
- 只需要手动解析部分的xml文件：使用Pull

# Java与C++对比

Java是由C++发展而来的，保留了C++的大部分内容，其编程方式类似于C++，但是摒弃了C++的诸多不合理之处，Java是纯面向对象的编程语言。Java和C++的区别主要如下：

## 都是类与对象

在Java中，一切的组件都是类与对象，没有单独的函数、方法与全局变量。C++中由于可以使用C代码，故C++中类对象与单独的函数方法共存。

## 多重继承

在C++中类可以多重继承，但这有可能会引起菱形问题。而在Java中类不允许多重继承，一个类只能继承一个基类，但可以实现多个接口，避免了菱形问题的产生。

## 操作符重载

C++允许重载操作符，而Java不允许。

## 数据类型大小

在C++中，不同的平台上，编译器对基本数据类型分别分配不同的字节数，导致了代码数据的不可移植性。在Java中，采用基于IEEE标准的数据类型，无论任何硬件平台上对数据类型的位数分配总是固定的。（然而boolean基本类型要看JVM的实现）

## 内存管理

C++需要程序员显式地声明和释放内存。Java中有垃圾回收器，会在程序内存不足或空闲之时在后台自行回收不再使用的内存，不需要程序员管理。

## 指针

指针是C++中最灵活也最容易出错的数据类型。Java中为了简单安全去掉了指针类型。

## 类型转换

C++中，会出现数据类型的隐含转换,涉及到自动强制类型转换。Java中系统要对对象的处理进行严格的相容性检查，防止不安全的转换。如果需要，必须由程序显式进行强制类型转换。（如int类型不能直接转换为boolean类型）

## 方法绑定

C++默认的方法绑定为静态绑定，如果要使用动态绑定实现多态需要用到关键字virtual。Java默认的方法绑定为动态绑定，只有final方法和static方法为静态绑定。

目前博主就想到这么多，还有的以后再补充。

# Java1.5、1.7与1.8新特性

## JDK1.5

1. 自动装箱与拆箱：基本类型与包装类型自动互换
2. 枚举类型的引入
3. 静态导入：import static，可直接使用静态变量与方法
4. 可变参数类型
5. 泛型
6. for-each循环

## JDK1.7

1. switch允许传入字符串
2. 泛型实例化类型自动推断：List<String> tempList = new ArrayList<>()
3. 对集合的支持，创建List / Set / Map 时写法更简单了，如：List< String> list = ["item"]，String item = list[0]，Set< String > set = {"item"}等
4. 允许在数字中使用下划线
5. 二进制符号加入，可用作二进制字符前加上 0b 来创建一个二进制类型：int binary = 0b1001_1001
6. 一个catch里捕捉多个异常类型，‘|’分隔

## JDK1.8

1. 允许为接口添加默认方法，又称为拓展方法，使用关键字default实现
2. Lambda 表达式
3. Date API
4. 多重注解