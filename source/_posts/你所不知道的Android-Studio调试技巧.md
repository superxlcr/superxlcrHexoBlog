---
title: 你所不知道的Android Studio调试技巧
tags: [android,应用]
categories: [android]
date: 2017-10-26 19:32:31
description: （转载）单步调试区、断点管理区、修改变量值、变量观察区、断点的分类、调试的两种方式
---

本文转载自：http://www.jianshu.com/p/011eb88f4e0d/


Android Studio目前已经成为开发Android的主要工具，用熟了可谓相当顺手。作为开发者，调试并发现bug，进而解决，可是我们的看家本领。正所谓，工欲善其事必先利其器，和其他开发工具一样，如Eclipse、Idea，Android Studio也为我们提供了强大的调试技巧，今天我们就来看看Android Studio中有关调试的技巧。
首先，来看看Android studio中为我们提供的调试面板（标准情况下）：

![_pic1](1.png)


点击右上角Restore ‘Threads’View可先展示目前相关的线程信息：

![_pic2](2.png)


android studio大体为我们提供了7个功能区：
1. 单步调试区
2. 断点管理区
3. 求值表达式
4. 线程帧栈区
5. 对象变量区
6. 变量观察区

下面我们分别对这七个区域进行介绍。

---

# 单步调试区

该区提供了调试的主要操作，和你所熟知的一样的，主要有：Step over、step into、force step into、step out、drop frame。

## Show Execution Point
![_pic3](3.jpg)


点击该按钮,光标将定位到当前正在调试的位置.

## Step Over
![_pic4](4.png)


单步跳过，点击该按钮将导致程序向下执行一行。如果当前行是一个方法调用，此行调用的方法被执行完毕后再到下一行。比如当前代码是：

```java
int num=10;
int min=Math.min(num,100);
System.out.println(min);
```

如果当前调试的是第二行，当点击step over时，Math.min（num,100)方法先执行完后跳到第三行.

## Step Into
![_pic5](5.png)


单步跳入，执行该操作将导致程序向下执行一行。如果该行有自定义的方法，则进入该方法内部继续执行，需要注意如果是类库中的方法，则不会进入方法内部。

## Force Step Into
![_pic6](6.png)


强制单步跳入，和step into功能类似，主要区别在于：如果当前行有任何方法，则不管该方法是我们自行定义还是类库提供的，都能跳入到方法内部继续执行

## Drop Frame
![_pic7](7.jpg)


没有好记的名字，大意理解为中断执行,并返回到方法执行的初始点,在这个过程中该方法对应的栈帧会从栈中移除.换言之,如果该方法是被调用的,则返回到当前方法被调用处，并且所有上下文变量的值也恢复到该方法未执行时的状态。简单的举例来说明：

```
public class DebugDemo {
    private String name = "default";

    public void alertName() {
        System.out.println(name);
        debug();
    }

    public void debug() {
        this.name = "debug";
    }

    public static void main(String[] args) {
        new DebugDemo().alertName();
    }
}
```

当你在调试debug（）时，执行该操作，将回调到debug（）被调用的地方，也就是alertName()方法。如果此时再继续执行drop frame，将回调到alertName（）被调用的地方，也就是main（）.

## Force Run to Cursor
![_pic8](8.jpg)


非常好用的一个功能,可以忽视已经存在的断点,跳转到光标所在处.举个简单例子说明下:

![_pic9](9.jpg)




比如现在第10行,此时我想调试18行而又不想一步一步调试,能不能一次到位呢?我们只需要将光标定位到相应的位置,然后执行Force Run to Cursor即可:

![_pic10](10.jpg)



## Evaluate expression
![_pic11](11.jpg)


点击该按钮会在当前调试的语句处嵌入一个交互式解释器，在该解释器中，你可以执行任何你想要执行的表达式进行求值操作。比如，我们在调试时执行到以下代码：

![_pic12](12.png)


此时执行Evaluate Expression，就相当于在调试行之前嵌入了一个交互式解释器，那么在该解释器中我们能做什么呢？在这里，我们可以对result进行求值操作：对着你想要求值得位置点击鼠标右键，选择evaluate Expression.此时会显示如下：

![_pic13](13.png)


在弹出的输入框中输入求值表达式，比如这里我们输入
```
Math.min(result,50)
```
,如下图

![_pic14](14.png)


点击执行，我们发现在Result中已经输出了结果，如下：

![_pic15](15.png)



---

# 断点管理区


## Return
![_pic16](16.jpg)


点击该按钮会停止目前的应用,并且重新启动.换言之,就是你想要重新调试时,可以使用该操作,嗯,就是重新来过的意思.

## Pause Program
![_pic17](17.jpg)


点击该按钮将暂停应用的执行.如果想要恢复则可以使用下面提到的Resume Program.

## Resume Program
![_pic18](18.png)


该操作有恢复应用的含义,但是却有两种行为:
1. 在应用处在暂停状态下,点击该按钮将恢复应用运行.
2. 在很多情况下，我们会设置多个断点以便调试。在某些情况下，我们需要从当前断点移动到下一个断点处，两个断点之间的代码自动被执行，这样我们就不需要一步一步调试到下一个断点了，省时又省力。举例说明：


```
public void test(){
    test1();
    ...
    test2();

}
```

假设我们分别在第2行和第4行添加了断点。如果此时我们调试在第2行，此时点击执行该操作，当前调试位置会自动执行到第4行，也就是第2到第4行之间的代码会自动被执行。

## Stop
![_pic19](19.jpg)


点击该按钮会通过相关的关闭脚本来终止当前进程.换言之,对不同类型的工程可能有不同的停止行为,比如:对普通的Java项目,点击该按钮意味着退出调试模式,但是应用还会执行完成.而在Android项目中,点击该按钮,则意味这app结束运行.
这里我们以一个普通的JAVA工程为例:

![_pic20](20.jpg)


此时如果我们执行停止操作，发现程序退出调试模式，并正常执行完毕，Console中结果如下：

![_pic21](21.jpg)



## View Breakpoints
![_pic22](22.png)


点击该按钮会进入断点管理界面，在这里你可以查看所有断点,管理或者配置断点的行为,如:删除，修改属性信息等：

![_pic23](23.jpg)



## Mute Breakpoints
![_pic24](24.jpg)


使用该按钮来切换断点的状态:启动或者禁用.在调试过程中,你可以禁用暂时禁用所有的断点,已实现应用正常的运行.该功能非常有用,比如当你在调试过程中,突然不想让断点干扰你所关心的流程时,可以临时禁用断点.

## Get thread dump
![_pic25](25.jpg)


获取线程Dump,点击该按钮将进入线程Dump界面:

![_pic26](26.jpg)


借此我们顺便介绍一下dump界面:

线程工具区中最常用的是
![_pic27](27.jpg)


,可以用来过滤线程,其他的不做解释了
解析来我们来认识一下线程的类型,表示为不同的图标:

| 线程状态描述 | 图标 | 
| - | - |
| Thread is suspended. | ![_pic28](28.png) | 
| Thread is waiting on a monitor lock. | ![_pic29](29.png) | 
| Thread is running. | ![_pic30](30.png) | 
| Thread is executing network operation, and is waiting for data to be passed. | ![_pic31](31.png) | 
| Thread is idle. | ![_pic32](32.png) | 
| Event Dispatch Thread that is busy. | ![_pic33](33.png) | 
| Thread is executing disk operation. | ![_pic34](34.png) | 

## Settings
![_pic35](35.jpg)


点击该按钮将打开有关设置的列表:

![_pic36](36.jpg)


我们对其中的几个进行说明:

### Show Values Inline

调试过程中开启该功能,将会代码右边显示变量值,即下图中红框所示部分:

![_pic37](37.jpg)



### Show Method Return Values

调试过程中启用该功能,将在变量区显示最后执行方法的返回值.举个例子来说,首先,关闭该功能,我们调试这段代码并观察其变量区:

![_pic38](38.jpg)




开启该功能之后,再来观察变量区的变化:

![_pic39](39.jpg)


继续往下调试:

![_pic40](40.jpg)


继续往下调试:

![_pic41](41.jpg)


这个功能简直是棒极了,在调试一段代码,并想看该代码中最后调用方法的最终结果时就非常有用了.

### Auto-Variables Mode

开启这个功能后,idea的Debugger会自动评估某些变量,大概就是当你执行在某个断点时,Debugger会检测当前调试点之前或者之后的变量的状态,然后在变量区选择性输出.举个例子来说明,未开启该功能之前,变量区输出所有的变量信息:

![_pic42](42.jpg)


开启之后,当你调试到第13行时,Debugger检测到num变量在之后没有被使用,那么在变量区就不会输出该变量的信息.

![_pic43](43.jpg)



### Sort values alphabetically

开启这个功能的化,变量区中的输出内容会按照按字母顺序进行排序,很简单,不常用,还是按照默认的顺序好.

## Help
![_pic44](44.jpg)


这个不用说了,有任何不明白的都可以查看官方帮助文档,这是我见到最好的文档之一.

其他几个操作:Settings,Pin,Close留给各位自己去使用.

---

# 修改变量值

在调试过程中，我们可以方便的修改某个变量的值，如下：

![_pic45](45.png)




在上图中，当前result的值经过计算为10，这里我们通过Set Value将其计算结果修改为100.

---

# 变量观察区

该区域将显示你所感兴趣的变量的值。在调试模式下，你可以通过Add to Watches将某个变量添加到观察区，该值的变化将会在变量观察区显示。操作如下：

![_pic46](46.jpg)




这里我们对name比较感兴趣，希望看到它的值的变化情况，因此我们将其“特殊关照”。需要注意，此时因为name是成员变量，因此在对象观察区也可看到该值。如果是局部变量，无疑只能用这种方式了。

---

# 断点的分类

到目前为止，我们已经简单的介绍了调试功能区，断点管理区，求值表达式，这三个区域的功能。在上面，我们不断的提到了断点一次，但是断点是什么呢？想必大部分人已经知道了，我们这里在简单的说明下：

断点是调试器的功能之一，可以让程序暂停在需要的地方，帮助我们进行分析程序的运行过程。

在Android Studio中，断点又被以下五类：
1. 条件断点
2. 日志断点
3. 异常断点
4. 方法断点
5. 属性断点

其中方法断点是我们最熟悉的断点类型，相信没有人不会。下面我们着重介绍其他四种类型的断点。

## 条件断点

所谓的条件断点就是在特定条件发生的断点，也就是，我们可将某个断点设置为只对某种事件感兴趣，最典型的应用就是在列表循环中，我们希望在某特定的元素出现时暂停程序运行。比如，现在我们有个list中，其中包含了q，1q，2q,3q四个元素，我们希望在遍历到2q时暂停程序运行，那么需要进行如下操作：

在需要的地方添加断点，如下：

![_pic47](47.png)


断点处左键单击，在Condition处填写过滤条件.此处我们只关心2q，因此填写
```
s.equals("2q")
```


![_pic48](48.png)



## 日志断点

该类型的断点不会使程序停下来，而是在输出我们要它输出的日志信息，然后继续执行。具体操作如下：

同样在断点处左键单击，在弹出的对话框中取消选中Suspend。

![_pic49](49.png)


在弹出的控制面板中，选中Log evaluated expression，然后再填写想要输出的日志信息，如下：

![_pic50](50.png)


当调试过程遇到该断点将会输出结果，如下：

![_pic51](51.png)



## 异常断点

所谓的异常断点就是在调试过程中，一旦发生异常（可以指定某类异常），则会立刻定位到异常抛出的地方。比如在调试异常中，我们非常关注运行时异常，希望在产生任何运行异常时及时定位，那么此时就可以利用该类型异常，在上线之前，进行异常断点调试非常有利于减少正式环境中发生crash的几率。

具体操作如下：在Run菜单项中，选择View Breakpoints（也可以在断点管理面板中点击
![_pic52](52.png)


），如下：

![_pic53](53.png)


在管理断点面板中，点击**+**

![_pic54](54.png)


在弹出的下拉选择列表中，我们选择Java Exception Breakpoints

![_pic55](55.png)


这里我们选中Search By Name,在下面的输入框中输入我们所关心的异常类型。此处我们关心NullPointerException，在调试过程一旦发生NullPointerException，调试器就会定位到异常发生处。

![_pic56](56.png)



## 方法断点
![_pic57](57.jpg)


（略过吧，应该没人不知道了）

## Filed WatchPoint
![_pic58](58.jpg)


Filed WatchPoint是本质上是一种特殊的断点，也称为属性断点：当我们某个字段值被修改的时候，程序暂停在修改处。通常在调试多线程时尤为可用，能帮我们及时的定位并发错误的问题。其使用和添加普通的断点并无不同，断点图标稍有不同

---

# 调试的两种方式

到目前，调试的相关基础我们已经介绍完了，但是不少童鞋对Android Studio中
![_pic59](59.png)


这两个按钮感到困惑：Debug和Attach process。

这里我们就简单介绍一下这两者的区别：
- Debug：以调试模式安装运行，断点可以在运行之前设置，也可在运行后设置，是多数人最常用的调式方式
- Attach process：和Debug方式相比，能够将调试器attach到任何正在运行的进程。比如，我们可以通过attach process到想要调试的进程。然后，在需要的地方设置相关断点即可。


