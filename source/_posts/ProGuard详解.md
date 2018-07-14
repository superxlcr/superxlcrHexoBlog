---
title: ProGuard详解
tags: [android,编译]
categories: [android]
date: 2017-08-10 16:31:48
description: （转载）简介、ProGuard作用、混淆注意事项、ProGuard使用
---

# 简介
[ProGuard](http://proguard.sourceforge.net/)是一个开源的Java代码混淆器。它可以混淆Android项目里面的java代码，对的，你没看错，仅仅是java代码。它是无法混淆Native代码，资源文件drawable、xml等。
# ProGuard作用
- 压缩: 移除无效的类、属性、方法等
- 优化: 优化字节码，并删除未使用的结构
- 混淆: 将类名、属性名、方法名混淆为难以读懂的字母，比如a,b,c；

# 混淆注意事项
## 不能混淆
- 在AndroidManifest中配置的类，比如四大组件
- JNI调用的方法
- 反射用到的类
- WebView中JavaScript调用的方法
- Layout文件引用到的自定义View
- 一些引入的第三方库（一般都会有混淆说明的）

推荐两个开源项目，里面收集了一些第三方库的混淆规则
[android-proguard-snippets](https://github.com/krschultz/android-proguard-snippets)
[android-proguard-cn](https://github.com/msdx/android-proguard-cn)

## Crash信息处理

代码混淆的时候记得加上在混淆文件里面记得加上这句： 

```
# keep住源文件以及行号
-keepattributes SourceFile,LineNumberTable
```


否则你看到的崩溃信息就会变成这样子:（图片来自bugly）

![_pic1](1.png)

这里推荐bugly的一篇文章： [
http://bugly.qq.com/bbs/forum.php?mod=viewthread&tid=26&extra=page%3D1](http://bugly.qq.com/bbs/forum.php?mod=viewthread&tid=26&extra=page=1)
# ProGuard使用
## 常用语法

```
// 从给定的文件中读取配置参数
-include {filename} 
// 指定基础目录为以后相对的档案名称
-basedirectory {directoryname}
// 指定要处理的应用程序jar,war,ear和目录   
-injars {class_path} 
// 指定处理完后要输出的jar,war,ear和目录的名称 
-outjars {class_path} 
// 指定要处理的应用程序jar,war,ear和目录所需要的程序库文件   
-libraryjars {classpath} 
// 指定不去忽略非公共的库类。
-dontskipnonpubliclibraryclasses
//  指定不去忽略包可见的库类的成员。
-dontskipnonpubliclibraryclassmembers
```

### 保留

```
// 保护指定的类文件和类的成员
-keep {Modifier} {class_specification} 
// 保护指定类的成员，如果此类受到保护他们会保护的更好
-keepclassmembers {modifier} {class_specification} 
// 保护指定的类和类的成员，但条件是所有指定的类和类成员是要存在。
-keepclasseswithmembers {class_specification} 
// 保护指定的类和类的成员的名称（如果他们不会压缩步骤中删除）
-keepnames {class_specification} 
// 保护指定的类的成员的名称（如果他们不会压缩步骤中删除）
-keepclassmembernames {class_specification} 
// 保护指定的类和类的成员的名称，如果所有指定的类成员出席（在压缩步骤之后）
-keepclasseswithmembernames {class_specification} 
// 列出类和类的成员-keep选项的清单，标准输出到给定的文件
-printseeds {filename}
```

### 压缩

```
-dontshrink 不压缩输入的类文件
-printusage {filename}
-whyareyoukeeping {class_specification}
```

### 优化

```
-dontoptimize 不优化输入的类文件
-assumenosideeffects {class_specification} 优化时假设指定的方法，没有任何副作用
-allowaccessmodification 优化时允许访问并修改有修饰符的类和类的成员
```

### 混淆

```
// 不混淆输入的类文件
-dontobfuscate 
// 使用给定文件中的关键字作为要混淆方法的名称
-obfuscationdictionary {filename} 
// 混淆时应用侵入式重载
-overloadaggressively 
// 确定统一的混淆类的成员名称来增加混淆
-useuniqueclassmembernames 
// 重新包装所有重命名的包并放在给定的单一包中
-flattenpackagehierarchy {package_name} 
// 重新包装所有重命名的类文件中放在给定的单一包中
-repackageclass {package_name} 
// 混淆时不会产生形形色色的类名
-dontusemixedcaseclassnames 
// 保护给定的可选属性，例如LineNumberTable, LocalVariableTable, SourceFile, Deprecated, Synthetic, Signature, and InnerClasses.
-keepattributes {attribute_name,…} 
// 设置源文件中给定的字符串常量
-renamesourcefileattribute {string}
```

### 通配符匹配规则

```
？      
匹配单个字符

*
匹配类名中的任何部分，但不包含额外的包名

**
匹配类名中的任何部分，并且可以包含额外的包名

%
匹配任何基础类型的类型名

***
匹配任意类型名 ,包含基础类型/非基础类型

...
匹配任意数量、任意类型的参数

<init>
匹配任何构造器

<ifield>
匹配任何字段名

<imethod>
匹配任何方法

*(当用在类内部时)
匹配任何字段和方法

$
指内部类
```


更详细的语法请戳:http://proguard.sourceforge.net/manual/usage.html#classspecification

## Android Studio中使用方法
按照上面的语法规则编写proguard-rules.pro后，需要在build.gradle中配置，需要混淆的时候，设置minifyEnabled为true即可

```
buildTypes {
    debug {
        minifyEnabled false
    }
    release {
        signingConfig signingConfigs.release
        minifyEnabled true
        proguardFiles 'proguard-rules.pro'
    }
}
```

## Eclipse 中使用方法
- 在工程目录下有个描述文件project.properties, 注意不是proguard-project.txt文件(当时因为这个原因一直失败)，添加一句话，启用ProGuard;
```
// 原文件内容：
# proguard.config=${sdk.dir}/tools/proguard/proguard-android.txt:proguard-project.txt
// 修改后内容(其实只是去除注释，并未添加):
proguard.config=${sdk.dir}/tools/proguard/proguard-android.txt:proguard-project.txt
```
	这样，Proguard就可以使用了。当我们正常通过Android Tools导出Application Package时（或者使用ant执行release打包），Proguard就会自动启用，优化混淆你的代码。
- 这一步并不是必要的，第一步中已经添加了sdk目录下的混淆工具，但是为了避免各个项目出现混乱（直接添加导致所有的项目都是使用sdk目录下的ProGard工具）；因此往往会将**proguard-android.txt复制**到项目的跟目录下，使每个项目各自拥有独立的ProGuard文件；
```
// 因此project.properties修改后：
proguard.config=proguard-android.txt:proguard-project.txt
// proguard-project.txt表示项目目录下的proguard-project.txt文件
```


## ProGuard的输出文件说明

混淆后，会在/build/proguard/目录下输出下面的文件 (Eclipse使用Export Android Application会在项目根目录下产生proguard目录；

- dump.txt 描述apk文件中所有类文件间的内部结构。
- mapping.txt 列出了原始的类，方法，和字段名与混淆后代码之间的映射。
- seeds.txt 列出了未被混淆的类和成员;
- usage.txt 列出了从apk中删除的代码 ;


