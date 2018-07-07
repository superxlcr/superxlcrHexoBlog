---
title: Gradle介绍
tags: [gradle]
categories: [gradle]
date: 2016-08-14 15:25:11
description: Groovy 语言简单介绍、Gradle介绍、Android Studio 中常见的 Gradle 问题、补充内容
---

Gradle是一个用于构建Android工程的工具，同时它也是一个编程框架。Gradle用于帮助我们自动管理编译流程，它允许我们定义个性化的编译内容，每个工程拥有自己的源码和资源的同时，还能使用一些通用的配置。而且Gradle和Android Studio是分离的，我们可以在不启动Android Studio的同时，单独使用命令行完成工程的编译工作。
Gradle语言具有以下特点：

- 基于Groovy语言
- 是一种DSL，即Domain Specific Language，领域相关语言，有自己特有的术语

因此对于Gradle的介绍，本文大致分为以下三个部分：

1. Groovy 语言简单介绍
2. Gradle 介绍
3. Android Studio 中常见的 Gradle 问题




# Groovy 语言简单介绍
## 概况


Groovy是在 Java平台上的、 具有像Python， Ruby 和 Smalltalk 语言特性的灵活动态语言， Groovy保证了这些特性像 Java语法一样被 Java开发者使用。我们编写的Java代码可直接当成Groovy Code来执行。Java、Groovy和JVM的关系如下图所示：Groovy Code 与 Java Code一致在执行的时候变成Java字节码在JVM上执行

![Groovy、Java和JVM_pic1](1.png)

Groovy下载地址：http://www.groovy-lang.org/download.html


Groovy文档地址：http://www.groovy-lang.org/documentation.html

Groovy的执行效果大致如下所示：

![Groovy执行效果_pic2](2.png)




## Groovy 语法特点介绍

Groovy在语法上与Java类似，但它有如下特点：



1. Groovy语句可以不用分号结尾
2. Groovy中支持动态类型，即定义变量的时候可以不指定其类型。Groovy中，变量定义可以使用关键字def，也可以不使用（但推荐都加上def，不然有时会报找不到属性值的错误）
3. 函数定义时，参数的类型也可以不指定（不指定意思即参数为动态类型）
4. Groovy中函数的返回值也可以是动态类型的
5. Groovy的函数里，可以不使用return来为函数设置返回值。如果不使用return语句的话，则函数里最后一句代码的执行结果会默认被设置成返回值
6. 函数调用可以不带括号（不过不带括号有可能会使函数被误认为是变量，因此一般是在Groovy API或 Gradle API上比较常用的函数才不带括号） 例如：println 'hello groovy'，一个调用println函数的例子



## Groovy 字符串介绍
Groovy对字符串支持相当强大，充分吸收了一些脚本语言的优点：

1. 单引号''中的内容严格对应Java中的String，就是普通的字符串
2. 双引号""的内容则和脚本语言的处理有点像，如果字符中有$符号的话，则它会先求$变量的值
```
def x = 10
println "x is $x" // x is 10

```
3. 三个引号'''xxx'''中的字符串支持随意换行




## Groovy 数据类型介绍
在这里我们主要介绍Groovy的三种数据类型：（详情请自行查阅文档）


1. Java中的基本数据类型（在实际执行过程中会被转换为对应的包装类型）
2. Groovy中的容器类，容器类型有以下三种：List，Map，Range（和List类似）
3. 闭包

值得注意的是在Groovy中获取对象的某个属性直接访问即可（如：a.b），执行时访问过程会转换为对应的getter和setter方法（因此有时候查看文档没有相关属性时，不妨去看看getter和setter方法）


在此特别介绍下Groovy关键的闭包类型。
闭包，是一种数据类型，它代表了一段可执行的代码。定义一个闭包的方式有三种，如下：

```
def closure = { paramters -> code } // 箭头前面为参数，后面为执行代码
def closure = { code }  // 有一个默认无类型参数 it
def closure = { -> code }  // 没有参数


```


调用闭包的方式有两种：
```
closure.call（paramters）
closure（paramters）

```

闭包在Groovy中大量使用，比如很多类都定义了一些函数，这些函数最后一个参数都是一个闭包，又因为函数调用可以省略圆括号，因此我们经常看到如下代码：


```
// Android Studio 默认生成一段的Gradle脚本
buildscript {
	repositories {
		jcenter()
	}
	dependencies {
		classpath 'com.android.tools.build:gradle:2.1.0'
	}
}

// 脚本补全后形态
buildscript ({ // 函数参数为闭包
	repositories ({ // 函数参数为闭包
		jcenter()
	})
	dependencies ({ // 函数参数为闭包
		add('classpath', 'com.android.tools.build:gradle:2.1.0')
	})
})

```

Closure很方便，但是它一定会和使用它的上下文有极强的关联，当某个函数参数是闭包的时候，我们有时连闭包中传入的参数是什么我们都不知道。

因此我们把闭包当作参数传递的时候一定要注意查看API文档：


Groovy API文档：http://www.groovy-lang.org/api.html


举个例子：
```
// 依次打印List中所有元素
list = [1, 2, 3]
list.each { // each函数的参数是闭包，根据文档会向闭包中传入list的每个元素
	println it // 打印闭包的默认参数it
}

```

each方法的API如下所示：
![each方法API_pic3](3.png)


## Groovy 的GDK
Groovy基于Java的特性，在JDK上添加了一些方法使得它变成了更灵活的GDK

GDK 文档：http://www.groovy-lang.org/gdk.html
下面列出一些GDK的常见文件I/O操作：


```
// 读取文件相关：（targetFile 是一个 File对象）

// 读取文件每一行并打印
targetFile.eachLine {
	String line ->
	    println line
}

// 文件内容一次性读出，返回类型为byte[]
targetFile.getBytes()

// 操作inputStream
targetFile.withInputStream {
    ism ->
        // 操作inputStream，结束后stream会自动close
}

// 写文件相关：
	
// 文件流copy
targetFile.withOutputStream {
    osm ->
        srcFile.withInputStream {
            ism ->
                osm << ism // 利用OutputStream的<<操作符重载完成
        }
}


```

## Groovy 的XML操作

Groovy还提供了一些API用于操作XML：http://www.groovy-lang.org/processing-xml.html

下面是一个打印AndroidManifest的versionName的例子，短短几行即可：


```
androidManifest = new XmlParser().parse('AndroidManifest.xml')
println androidManifest['@android:versionName']
println androidManifest.@'android:versionName'


```

上面是对Groovy语言的一些较基础的介绍，想要深入了解的同学建议去看看官方的文档继续学习。介绍完Groovy的一些比较基础的知识后，我们再来了解Gradle是什么东西。



# Gradle介绍

## Gradle相关资料
首先我们现在这里列出一些Gradle相关的文档资料：
Gradle的官网：http://gradle.org/

Gradle 文档位置：https://docs.gradle.org/current/release-notes

其中，文档中几个标签分别为：


- User Guide：介绍Gradle的一本书
- DSLReference：Gradle API的说明
- Javadoc：Gradle API的函数文档
- Groovydoc：Groovy的文档

## Gradle 基本概念

首先我们来了解一些Gradle中比较重要的基本概念：

- Project：在Gradle中，每个待编译的工程称为Project每个Project对应一个build.gradle文件，用于配置Project的属性，类似Makefile
- Multi-Project： build为了一次性编译多个工程，我们可以构建一个MultiProjectsMultiProjects有一个build.gradle文件，用于配置gradle的配置和其它子Project的属性MultiProjects文还有一个settings.gradle文件，用于指定这个Projects具体包含哪些子Project在Android Studio中，最顶层的MultiProject叫做Project，其子Project叫做Module
- Task：task是编译过程的执行者每个Project都会包含一系列的task，每个task用于执行特定的编译任务多个task之间可能会存在依赖关系，某个task执行前，会自动执行它所依赖的task，task拥有多种类型，一般我们构建的都是DefaultTask类型，但有时为了特殊的需要，我们也可以构建一些特殊类型的task（如：复制文件 Copy，删除文件 Delete），不同类型的task会帮助我们执行不同的操作
- Property：除了task外，Project还包含一系列的Property属性值我们可以通过ext.xxx的方法来为某个Project设置额外的属性值
- Plugin：Plugin即插件，插件包含了多个task以及一些与task相关的Property在Project中，我们可以通过apply方法应用插件，我们通过修改插件的Property，执行插件的task得到不同的结果

## Gradle 终端常用命令


知道了Gradle的一些基本概念后，我们再来了解一下常用的Gradle的命令：

- gradle xxx（task-name） 执行特定的任务
- gradle projects 查看当前Project及其子Project的信息
- gradle tasks 查看可执行的任务信息

一些执行task的例子：


执行gradle xxx：
![gradle xxx_pic4](4.png)


执行gradle projects：
![gradle projects_pic5](5.png)



执行gradle tasks：
![gradle tasks_pic6](6.png)



## Gradle 工作流程
接下来我们来了解一下Gradle的工作流程，Gradle的执行过程可以用下图表示：
![gradle工作流程_pic7](7.png)

从图中可以看出，Gradle的执行过程分为三个步骤：

- Initialization：遍历文件目录，解析settings.gradle，确定project数目
- Configuration：解析每个project下的build.gradle，根据task间的依赖关系生成一个task的有向图
- Execution：根据输入的指令与task有向图执行相应的task

其中，三个步骤之间Gradle为我们提供了一些函数用于Hook，这些函数的顺序可以用下图简略表示（详细的请大家去看文档）：

![gradle hook函数_pic8](8.png)

在Gradle的执行过程中，会产生以下三种关键对象：

- Gradle 对象：Gradle对象在整个执行过程中只有一个，我们可以通过调用它的方法传入闭包，来在工作流程中添加自己的指令（即Hook）。我们在脚本中使用gradle访问该对象
- Project 对象：Project对象与build.gradle文件一一对应，可以获取一些关于project的属性信息（如：project文件目录，build文件目录等）或添加Hook。在脚本中用project访问当前的Project对象
- Settings 对象：Settings对象与settings.gradle对应

下面举个例子说明Gradle的执行过程：

假设有一个multiProjects文件夹，里面有build.gradle与settings.gradle两个文件，以及project1与project2两个文件夹，两个文件夹中都有build.gradle文件。示意图如下：
![文件夹示意图_pic9](9.png)

其中，settings.gradle文件内容如下：
```
println 'Initialization'
include 'project1'
include 'project2'

```
打印了Initialization，并引用了project1与project2两个工程


而build.gradle文件内容均如下：
```
println "Configuration $project.name"

task myTask << {
    println "Execution $project.name"
}

```
其中打印了Configuration 当前工程的名称，并定义了一个名为myTask的任务（任务内容是打印Execution 当前执行工程名称）


下面是使用命令的效果，这些命令均在multiProjects文件夹下运行
当我们使用projects命令查看工程结构时：

![gradle_pic10](10.png)

可以看到我们的顶层project为multiProjects，被包含的两个字Project分别为project1与project2（Android Studio中称为module）


当我们使用tasks查看任务列表时：
![gradle tasks_pic11](11.png)

可以看到一些Gradle为我们内置的任务，以及我们自定义的任务。其中白色的字体是任务分组名称，绿色字体是任务名称，而黄色字体是任务描述。（这里我们的myTask其实有三个，因为每个project都定义了自己的myTask任务，显示的时候就集中显示了）


接下来我们执行gradle myTask指令时：
![gradle myTask_pic12](12.png)

从myTask任务的执行过程，我们可以清楚看到Gradle执行的三个阶段。值得注意的是如果出现了多个同名任务，则会都执行一遍


## 编写settings.gradle
学会了一些Gradle相关的基本概念后，我们就来看看如何编写Gradle脚本。由于settings.gradle与Settings对象对应，因此编写settings.gradle实际就上调用Settings对象的函数

Settings对象文档：https://docs.gradle.org/current/dsl/org.gradle.api.initialization.Settings.html
编写settings.gradle的主要目标：确定multiProjects（顶层Project）包含哪些子Project
一个默认Android工程的settings.gradle文件如下：

![default_settings_pic13](13.png)

脚本中的include就是指包含某个project的意思，这段代码看起来很陌生，其实当我们查阅Settings的函数时我们会发现一个include函数：
![include_pic14](14.png)

结合Groovy语言省略括号与分号的特性，代码就变成了上面略微飘逸的模样。


## 编写build.gradle（顶层Project）
每个project都有自己的build.gradle，接下来讲讲如何编写顶层Project的build.gradle，由于build.gradle与Project对象对应，因此编写build.gradle实际就上调用Project对象的函数

Project对象文档：https://docs.gradle.org/current/dsl/org.gradle.api.Project.html

Android官方文档：https://developer.android.com/studio/build/index.html

顶层project的build.gradle主要任务有：

- 设置gradle整体相关的配置
- 为子Projects进行统一的配置

先来看默认Android Studio工程顶层Project的build.gradle：

![默认build.gradle_pic15](15.png)

这里引入一个Script Block的概念，其实就是一个函数名称加上大括号，实际上就是一个函数的参数是闭包，然后省略了括号与分号而已。
这里调用的函数均可以在Project对象中找到，比如说buildscript函数：
![buildscript函数_pic16](16.png)

可以看到buildscript会把闭包交个ScriptHandler去处理，根据脚本代码，我们应该可以在ScriptHandler中找到dependencies函数：
![dependencies_pic17](17.png)

dependencies函数参数同样是个闭包，而它会把闭包交给DependencyHandler处理，在其中我们可以找到一个add函数（不过为啥add函数可以简写成那个样子我也不是十分清楚了……）：
![add_pic18](18.png)

因此，我们默认的Android Studio工程的build.gradle其实写清楚一点应该是这样子的：
![true_default_root_build_pic19](19.png)



一般而言，当我们需要对Gradle进行整体的设置的时候，对于上述的代码我们只需要看懂即可。因为Android Studio提供了非常人性化的GUI让我们来轻松完成Gradle的设置：
打开File-&gt;Project Sturcture设置即可
![project_struct_pic20](20.png)



点击project项用于设置整体gradle环境：
![set gradle_pic21](21.png)



自动生成的build.gradle如下：
![auto root build_pic22](22.png)



但如果你还需要为子Projects进行统一的配置，那就不得不手打一些脚本了，这里给出了一些例子：
![build_sample_pic23](23.png)



其中findAll函数的文档如下：
![findAll_pic24](24.png)

对于传入其中的闭包，findAll函数会把集合的每个元素当做参数传进去


## 编写build.gradle（子Project）
编写子Project的build.gradle文件主要有两个任务：

- 应用插件，即Plugin
- 为project编写依赖

下面是一个默认Android Studio工程子Project的build.gradle：


![single build_pic25](25.png)



该脚本主要分为三部分：

- 最顶端的插件应用声明
- 中间的插件属性设置
- 底部的project依赖设置

插件声明部分：

Android插件文档：http://google.github.io/android-gradle-dsl/current/index.html

![single project plugin_pic26](26.png)



目前我们要使用的插件主要有三种：

- application：生成apk时使用
- library：lib工程使用
- test：测试使用

每个插件都有自己对应的属性，可以在文档中查到。以application为例：

![property_pic27](27.png)



可以看到上面默认的脚本就设置了一些必要的属性。


与编写root project的build.gradle一样，我们没有必要完全手写脚本，可以使用Android Studio的GUI解决大部分问题：
![project struct_pic28](28.png)



点击对应的子project进行设置即可：
![set project_pic29](29.png)



其中：

- Properties：编译属性相关配置，编译的SDK和工具等
- Signing：数字签名相关配置
- Flavors：打包渠道相关配置，支持的SDK，APP的Id等
- BuildTypes：编译类型相关配置，是否混淆，混淆的文件，是否压缩等
- Dependencies：依赖相关配置

值得注意的是，生成apk的名称也会有对应的变化：

![build apk_pic30](30.png)



## Task相关
由于在Gradle中，最后的工作均由task来执行，因此当我们需要添加某些新的功能时，只需要添加task即可
Task文档：https://docs.gradle.org/current/dsl/org.gradle.api.Task.html

Task文档2：https://docs.gradle.org/current/userguide/more_about_tasks.html



关于创建Task：
下面给出三种正确创建task的方法，值得注意的是红框部分不能去掉：
![new task right_pic31](31.png)



错误的创建task方式如下：
![new task wrong_pic32](32.png)



两者的区别何在呢？看回我们Gradle的执行流程图便一目了然了：
![gradle work_pic33](33.png)



对于前面三种创建task的方式，其内容会在Execution阶段执行。而后两种创建task的方式，其内容会在Configuration阶段执行，即用于配置的。
举例如下：
![new task sample_pic34](34.png)



可以看到我们执行myTask1，然而因为每个task都必将经历Configuration配置阶段，因此myTask5也打印了自己的输出。


为task添加描述：

我们很简单就可以为task添加描述文字
![task description_pic35](35.png)



添加描述后，在使用tasks命令时我们可以看到：
![task description sample_pic36](36.png)



为task分组：

![task group_pic37](37.png)



添加分组后，我们在使用tasks命令也可以看到：
![task group sample_pic38](38.png)



使用task类型：
为了方便我们创建task，Gradle设置了一些默认的task类型供我们使用。声明task是某种类型，然后在配置阶段修改一些参数，我们即可完成复杂的任务，非常方便。
![task type_pic39](39.png)



上面分别是一个删除文件与复制文件的task，task type相关的文档我们可以在Gradle官网找到：
![task type doc_pic40](40.png)



值得注意的是，Task除了包含一些执行的操作外，还包含自己的Property：

![task property_pic41](41.png)

![set task property_pic42](42.png)



例如，通过设置task的inputs与outputs属性，我们可以省略某些执行过程：
![up to date_pic43](43.png)



由于inputs与outputs在两次执行过程中没有变化，因此该任务属于up-to-date，即不需要再次执行。


当我们需要为我们的工程添加新功能时，我们一般由如下两种做法：


- 创建task并依赖已有的task
- 修改已有的task

创建task并依赖已有的task：


task依赖设置 方法如下，执行某个task前，Gradle会自动执行依赖的task

![task depend_pic44](44.png)



执行例子如下（由于myTask2依赖于myTask1，因此执行myTask2前自动执行myTask1）：
![task depend sample_pic45](45.png)



这里又引出另一个问题，当我们一次依赖多个task时，这些被依赖的task执行顺序是怎样的呢？
答案是随机的。当然我们也可以通过脚本规定他们的顺序：
![task order_pic46](46.png)



运行效果如下：
执行myTask3，由于依赖关系必须先执行myTask1与myTask2，再根据脚本规定的顺序先执行myTask2
![task order sample_pic47](47.png)



修改已有的task：
除了创建新的task外，我们还可以通过向已有task添加内容：
![task add new_pic48](48.png)



效果如下：
![task add new sample_pic49](49.png)



另外，也可以重写某个已有的task：
![task overwrite_pic50](50.png)



效果如下：
![task overwrite sample_pic51](51.png)



可以看到，我们添加新的功能都与已有的task相关，这就要求我们必须对已有的task执行流程有个大致的掌握。这里给出一个有向图表示Application plugin 包含的task的执行顺序，具体的大家可以通过gradle tasks --all 来查看：
![plugin already task_pic52](52.png)



## Gradle 相关的文件
下面介绍一些常见的Gradle相关的文件：

- gradle-wrapper.properties：gradle wrapper配置文件，确定gradle wrapper的版本以及存放地点等。gradle wrapper用于解决本地gradle版本与项目不一致问题，在终端使用gradlew代替gradle（即：gradlew xxx）
- proguard-rules.pro：代码混淆配置文件
- gradle.properties：gradle属性配置文件，用于设置gradle下载代理，是否并行编译，是否后台编译，为JVM添加参数 。文档：https://docs.gradle.org/current/userguide/build_environment.html
- local.properties：用于存储一些和Android相关的变量，也可以存储一些自定义的变量贴一段用于获取属性值的代码：![get Property_pic53](53.png)




# Android Studio 中常见的 Gradle 问题
下面分享一些Android Studio中常见问题的解决办法：

- Gradle下载代理问题
- Gradle Wrapper使用本地压缩包
- Dependencies 关键词
- Jar打包
- 开启守护进程编译和并行编译
- 获取工程svn版本号

Gradle下载代理问题：


由于Gradle需要翻墙下载，因此在国内经常会出现在这个步骤卡死的状况，这时我们可以通过设置代理解决问题：
找到gradle.properties文件：
![proxy file_pic54](54.png)



添加http与https代理即可：
![proxy content_pic55](55.png)



Gradle Wrapper使用本地压缩包：

这个问题与上一个类似，都是因为网络原因导致下载缓慢进而卡死。如果你以前曾经下载过Gradle Wrapper或能找到压缩包文件的话，完全可使用本地的压缩包而不需要去下载。
一般而言，以前下载过的gradle wrapper可以在一下路径找到（最后为一串乱码）：

C:\Users\用户名\.gradle\wrapper\dists\gradle-2.10-all\a4w5fzrkeut1ox71xslb49gst\

![local file_pic56](56.png)



然后找到gradle-wrapper.properties文件：
![local file file_pic57](57.png)



修改使用的gradle wrapper地址即可：
![local file content_pic58](58.png)



Dependencies 关键词：

这是博主踩过的一个坑，没弄清楚dependencies里面的关键词导致依赖出现了错误。
如图所示，dependencies里面有六个关键词，分别对应不同的依赖方式：
![dependencies keyword_pic59](59.png)



解释如下：

- Compile：对所有的build type以及favlors都会参与编译并且打包到最终的apk文件中
- Provided：对所有的build type以及favlors只在编译时使用，类似eclipse中的external-libs，只参与编译，不打包到最终apk
- APK：只会打包到apk文件中，而不参与编译，所以不能再代码中直接调用jar中的类或方法，否则在编译时会报错
- Test compile：仅仅是针对单元测试代码的编译编译以及最终打包测试apk时有效，而对正常的debug或者release apk包不起作用
- Debug compile：仅仅针对debug模式的编译和最终的debug apk打包
- Release compile：仅仅针对Release 模式的编译和最终的Release apk打包




Jar打包：

Android Studio的library插件只提供aar打包，当我们需要jar包的时候该怎么做呢？一般方法有两种如下：


方法一：复制build/intermediates/bundles/buildType下的classes.jar

该jar是生成aar包过程中的一个中间文件，它就是我们需要的jar包。下面是一段获取该jar的代码：
![get jar_pic60](60.png)



方法二：依赖Java编译任务，使用Jar任务类型手动打包，使用ProGuardTask任务类型进行代码混淆
由于该方法较为复杂，这里就不展开说明了，有兴趣的同学可以参考下面网址：
http://chaosleong.github.io/blog/2015/08/02/android-studio-shi-yong-gradle-da-bao-jar/


开启守护进程编译和并行编译：

Gradle在开启守护进程编译和并行编译后，可以显著的提示编译速度，但缺点是并行编译可能导致输出混乱。
在gradle.properties文件中，开启守护进程编译与并行编译：
![compile file_pic61](61.png)

![compile content_pic62](62.png)



效果如下（可以看到时间明显减少了）：
![compile sample_pic63](63.png)



获取工程svn版本号：

代码如下，其中exec为执行外部命令的函数：
![get svn_pic64](64.png)

# 补充内容

以下内容转自：https://blog.csdn.net/lisdye2/article/details/79156155

## Groovy中闭包的委托对象
在Closure中有一个属性为delegate,该属性表示我们可以为闭包设置一个代理对象，那么在闭包中可以直接使用这个代理对象的一些方法。

```
class DelegateDemo {

    static void main(String[] args) {
        Closure c = {
            // 调用代理对象的方法
            test()
        }
        // 设置代理对象
        c.delegate = new DelegateDemo()
        //运行闭包
        c.call()
    }

    void test() {
        println("this is delegate")
    }

}

```

## 利用Groovy模仿Gradle中的dependencies依赖声明
关键点便在于委托
声明一个类用于存储所有的依赖，代码如下：

```java
  static class Dependency {
    // 保存所有的依赖
        Set<String> api = new HashSet<>();
        // 添加依赖的方法
        void api(String text){
            api.add(text)
        }
        // 执行方法，暂时只是打印所有依赖
        void exec(){
            println(api)
        }
    }
```

该类同时将作为一个闭包的代理对象。
其次，声明一个方法，该方法的参数为一个闭包。

```
 static void dependencies(Closure closure) {
    // 声明一个代理对象
        def dependency = new Dependency()
        // 设置委托
        closure.delegate = dependency
        // 运行闭包
        closure.call()
        // 执行
        dependency.exec()
    }
```

最后看一下如何使用

```
 static void main(String[] args) {
        dependencies {
            api 'cn.jiguang.sdk.plugin:xiaomi:3.1.0'
            api 'cn.jiguang.sdk.plugin:huawei:3.1.0'
            api 'cn.jiguang.sdk.plugin:meizu:3.1.0'
        }
    }
```

是不是很眼熟。。。。
执行结果：

```
[cn.jiguang.sdk.plugin:meizu:3.1.0, cn.jiguang.sdk.plugin:xiaomi:3.1.0, cn.jiguang.sdk.plugin:huawei:3.1.0]
```

## 利用Groovy模仿一个Task任务
在使用Gradle中，可以通过声明一个task并添加doLast和doFirst
回调，而下面就开始仿写一个类似的逻辑。关键点在于闭包和代理对象。
根据上一个例子，分为三步：
**第一步：声明代理委托对象**

```java
 static class Task {
        String config = ""
        Closure doLast = {}
        Closure doFirst = {}

        void doLast(Closure c) {
            doLast = c
        }

        void doFirst(Closure c) {
            doFirst = c
        }
        // 执行任务，只是简单打印一下配置
        void exec() {
            println(config)
        }
        // 提供一个配置选项
        void config(String text) {
            config = text
        }
    }
```

**第二步：声明一个以闭包作为参数的方法**

```
    static void task(String name, Closure closure) {
        def task = new Task()
        closure.delegate = task
        // 执行闭包，实质是对task坐初始化
        closure()
        // 调用doFirst
        task.doFirst.call()
        // 调用执行方法
        task.exec()
        // 调用doLast()方法
        task.doLast.call()
    }
```

**第三步：使用**

```
 static void main(String[] args) {
         {
            
            doLast {
                
            }
            doFirst {
                
            }
        }
    }
```

在这里，有一个关键，如果闭包作为最后一个参数，那么他可以写到()外面。
同时，如果调用的方法的参数如果仅为一个，则可以使用空格的方式，最终如下：

```
  task("test") {
            config "123"
            doLast {
                println("doLast")
            }
            doFirst {
                println("doFirst")
            }
        }
```

**打印结果**

```
doFirst
123
doLast

```