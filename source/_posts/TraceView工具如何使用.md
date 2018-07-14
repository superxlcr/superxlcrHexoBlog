---
title: TraceView工具如何使用
tags: []
categories: []
date: 2017-10-12 20:53:07
description: （转载）TraceView工具如何使用、TraceView工具面板介绍、如何进行具体的分析、相关资料
---

# TraceView工具如何使用

TraceView有4种启动/关闭分析方式：




## 第一种使用方法演示


###  选择跟踪范围

在想要根据的代码片段之间使用以下两句代码：




```java
Debug.startMethodTracing("love_world_");
Debug.stopMethodTracing();
```


例如，onCreate与onStart方法之间方法跟踪




```java
public class MainActivity extends Activity {  
  
    @Override  
    protected void onCreate(Bundle savedInstanceState) {  
        super.onCreate(savedInstanceState);  
        setContentView(R.layout.activity_main);  
          
        Debug.startMethodTracing("Love_World_");  
    }  
  
    @Override  
    protected void onStart() {  
        super.onStart();  
          
        Debug.stopMethodTracing();  
    }  
      
}  
```


### 添加SD卡访问权限



```java
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"/>  
<uses-permission android:name="android.permission.MOUNT_UNMOUNT_FILESYSTEMS"/>    
```


如果不添加，执行项目会出现以下异常




```
java.lang.RuntimeException:Unable to open trace file '/mnt/sdcard/Love_World_.trace': Permission denied  
```


如果手机没有SD卡也会出现同样的问题




### 导出traceview文件


#### 首先执行项目，查看trace文件是否生成

进入shell模式




```
adb shell  
```


查看是否已经生成这个文件




```
ls sdcard/Love_World_.trace  
```


#### 导出trace文件



```
adb pull sdcard/Love_World_.trace
```


### 打开trace文件


打开trace文件需要Android提供的traceview.bat工具，工具所在目录：sdk\tools\traceview.bat， 有两种方式执行：


1. 在命令行中切换到此目录
2. 将此目录添加到系统环境变量中


```
//  cmd在calc.trace所在目录执行  
traceview C:\Users\YourName\Desktop\Love_World_.trace  
```


其中“C:\Users\YourName\Desktop\” 表示trace所在你系统中的目录，此工具需要输入trace文件的绝对路径才行




在新版本的SDK 会有以下提示：




```
The standalone version of traceview is deprecated.  
Please use Android Device Monitor (tools/monitor) instead.  
```


所以建议使用tools/monitor 启动后跟Eclipse DDMS界面差不多，然后File -&gt; Open File -&gt; 选择trace文件

### 异常处理

#### 异常处理


```
'C:\Windows\system32\java.exe' 不是内部或外部命令，也不是可运行的程序  
或批处理文件。  
SWT folder '' does not exist.  
Please set ANDROID_SWT to point to the folder containing swt.jar for your platfo  
rm.  
```


#### 异常信息


```
The standalone version of traceview is deprecated.  
Please use Android Device Monitor (tools/monitor) instead.  
Failed to read the trace filejava.io.IOException: Key section does not have an *  
end marker  
        at com.android.traceview.DmTraceReader.parseKeys(DmTraceReader.java:420)  
        at com.android.traceview.DmTraceReader.generateTrees(DmTraceReader.java:91)  
        at com.android.traceview.DmTraceReader.<init>(DmTraceReader.java:87)  
        at com.android.traceview.MainWindow.main(MainWindow.java:286)  
```


通常是trace文件有异常，再重新生成并导出试试



#### 没有SD卡会出现异常


```
Unable to open trace file '/sdcard/Love_World_.trace': Permission denied  
 Caused by: java.lang.RuntimeException: Unable to open trace file '/sdcard/Love_World_.trace': Permission denied  
```


生成的trace系统自动放在SDCARD上，没有sd卡所以会出现这种异常




## 第二种使用方法演示

Eclipse -&gt; DDMS -&gt; Start Method Profiling



二者的区别，第一种方式更精确到方法，不方便的地方是自己需要添加方法并且要导出文件，第二种方式的优缺点刚好相反。




## 任意时间点启动与关闭trace

启动：am profile &lt;PROCESS&gt; start &lt;FILE&gt;

关闭：am profile &lt;PROCESS&gt; stop


&lt;PROCESS&gt; 填写进程名，例如AndroidManifest.xml中声明的包名，通常都是主进程名


例如：  

adb shell am profile com.example start ./mnt/sdcard/test.trace


adb shell am profile com.example stop




## 启动指定Activity并进行trace

am start -n &lt;Package Name&gt;/&lt;Package Name&gt;.&lt;Activity Name&gt; --start-profiler &lt;FILE&gt;

例如：

adb shell am start -n com.example/com.example.MainActivity --start-profiler ./mnt/sdcard/test.trace


细节详见官方文档：

http://developer.Android.com/tools/help/shell.html




# TraceView工具面板介绍

有两方面用途： 

1. 查看跟踪代码的执行时间，分析哪些是耗时操作  

2. 可以用于跟踪方法的调用，尤其是android Framework层的方法调用关系


获取方法的调用顺序

1. 在traceview中搜索响应的方法名不能使用大写字母

2. 搜索出的方法会自动展开，其中包含Parents 和 Children 两组信息

3. 点击Parents下的方法名，直接跳转到调用当前的方法处。Children相反

![_pic1](1.gif)







Traceview 面板分上下两部分
**上面是时间轴面板 (Timeline Panel)**
左侧显示的是线程信息
右侧黑色部分是显示执行时间段、白色是线程暂停时间段，
右侧鼠标放在上面会出现时间线纵轴，在顶部会显示当前时间线所执行的具体函数信息
**下面是分析面板(Profile Panel) **
每一列内容:
- Inclusive time  - 函数本身运行花费时间 + 函数调用其他函数时间
- Exclusive time - 函数本身运行花费时间。
- Calls + RecurCall/Total 调用 + 重复调用次数 / 函数总调用次数
- Cpu Time/Call 总的Cpu时间与总的调用次数之比

表1-1  Profile Panel各列作用说明

| 列名 | 描述 | 
| - | - |
| Name | 该线程运行过程中所调用的函数名 | 
| Incl Cpu Time | 某函数占用的CPU时间，包含内部调用其它函数的CPU时间 | 
| Excl Cpu Time | 某函数占用的CPU时间，但不含内部调用其它函数所占用的CPU时间 | 
| Incl Real Time | 某函数运行的真实时间（以毫秒为单位），内含调用其它函数所占用的真实时间 | 
| Excl Real Time | 某函数运行的真实时间（以毫秒为单位），不含调用其它函数所占用的真实时间 | 
| Call+Recur Calls/Total | 某函数被调用次数以及递归调用占总调用次数的百分比 | 
| Cpu Time/Call | 某函数调用CPU时间与调用次数的比。相当于该函数平均执行时间 | 
| Real Time/Call | 同CPU Time/Call类似，只不过统计单位换成了真实时间 | 






# 如何进行具体的分析


有两个问题需要解决：


1. 如何定位到所关心的地方？ 
	上面只是介绍了如何使用TraceView且有两种用法，但是有时使用第一种方式范围又不太精确，使用第二种添加代码的方式，可能有些地方又监听不到。这种情况可以尝试把开始或者结束放到延迟线程中，延迟一段时间在执行开始或者结束。
2. 如何查找出哪些地方比较耗时？
	TraceView罗列出了是所有监听到的方法，当然也包括Android系统很多方法的耗时，如何在这么多方法里面查找到自己关心的？ 可以通过TraceView 底部的find 来查找，通常Android app都是有包名的，可以先针对某些关心的列排序后，在通过包名进行一个个查找，这些就省去自己筛选出自己app 方法耗时排行的时间。




# 相关资料

[Android系统性能调优工具介绍](http://blog.csdn.net/innost/article/details/9008691) （还有具体如何使用进行性能分析的例子，非常棒）
[Profiling with Traceview and dmtracedump](http://developer.android.com/tools/debugging/debugging-tracing.html#timelinepanel)
[Android 编程下的 TraceView 简介及其案例实战](http://www.cnblogs.com/sunzn/p/3192231.html) （含例子）
[Android代码调试工具 traceview 和 dmtracedump的波折演绎](http://blog.csdn.net/yiyaaixuexi/article/details/6716884)



原文地址： [http://blog.csdn.net/love_world_/article/details/8223805](http://blog.csdn.net/androiddevelop/article/details/8223805)

