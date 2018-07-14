---
title: Android BuildConfig类
tags: [android]
categories: [android]
date: 2017-08-06 15:04:02
description: 什么是BuildConfig类、如何在BuildConfig类中添加常量、使用BuildConfig.DEBUG的一些问题
---
# 什么是BuildConfig类
BuildConfig是android studio在打包时自动生成的一个java类，在项目工程的build/generated/source/buildConfig目录下，打开这个目录可以发现会有多个不同的目录来存放BuildConfig.java类，一般会有androidTest、debug、release等多个目录，对应着不同的buildTypes以及flavors。一般情况下，这些目录中的BuildConfig类中有相同的常量字段：


| 字段 | 含义 | 
| - | - |
| DEBUG | 是否为调试版本 | 
| APPLICATION_ID | 应用id（包名） | 
| BUILD_TYPE | 构建类型（一般为debug与release） | 
| FLAVOR | 渠道 | 
| VERSION_CODE | 版本号 | 
| VERSION_NAME | 版本名 | 


一般而言，我们可以通过访问BuildConfig类的常量来进行一些判断处理，比如：

- 通过判断DEBUG来确定是否为调试模式，是否打印日志
- 通过判断VERSION_CODE与VERSION_NAME来确定应用版本，是否需要进行升级
- 通过判断FLAVOR来获取应用的渠道信息，进行渠道统计等


# 如何在BuildConfig类中添加常量
BuildConfig是android studio自动生成的一个类，就是说我们不能手动去修改这个类里面的内容，但我们可以在Gradle构建脚本中通过相应的函数来为BuildConfig类添加常量。

例子如下：

```java
buildTypes {  
        release {  
            minifyEnabled true  
            signingConfig signingConfigs.release  
            proguardFiles getDefaultProguardFile('proguard-project.txt'), 'proguard-rules.pro'  
            buildConfigField("String", "TEST", 'Release')  
        }  
  
        debug {  
            signingConfig signingConfigs.debug  
            proguardFiles getDefaultProguardFile('proguard-project.txt'), 'proguard-rules.pro'  
            buildConfigField("String", "TEST", 'debug')  
        }  
}
```


在上面的例子中，我们通过buildConfigField函数，在release与debug两种buildTypes的BuildConfig类中，添加了TEST常量字段，并设置了相应的值。
buildConfigField函数如下：

```java
/** 
 * Adds a new field to the generated BuildConfig class. 
 * 
 * <p>The field is generated as: <code><type> <name> = <value>;</code> 
 * 
 * <p>This means each of these must have valid Java content. If the type is a String, then the 
 * value should include quotes. 
 * 
 * @param type the type of the field 
 * @param name the name of the field 
 * @param value the value of the field 
 */  
public void buildConfigField(  
        @NonNull String type,  
        @NonNull String name,  
        @NonNull String value) {  
    ClassField alreadyPresent = getBuildConfigFields().get(name);  
    if (alreadyPresent != null) {  
        logger.info("BuildType({}): buildConfigField '{}' value is being replaced: {} -> {}",  
                getName(), name, alreadyPresent.getValue(), value);  
    }  
    addBuildConfigField(AndroidBuilder.createClassField(type, name, value));  
} 
```


该方法需要传入三个参数，第一个参数为要定义的常量的类型，第二个参数为该常量的命名，第三个参数为该常量的值。在Gradle中调用该方法后，我们可以在BuildConfig类中找到对应的常量



# 使用BuildConfig.DEBUG的一些问题
当我们的项目包含一些jar或者aar的module，在使用BuildConfig.DEBUG这个常量时，会遇到如下问题：
尽管我们的app的module使用debug的buildType来进行构建，但库module却使用的是release的buildType进行构建。
也就是说，在app的module中，我们获取BuildConfig.DEBUG变量为true，但在依赖的库module中BuildConfig.DEBUG确是false。
该问题的解决方法有以下几种：

- 在library的配置里面使用 publishNonDefault true
- 在library的配置里面使用 defaultPublishConfig "debug"
- 在主程序的配置里面使用 compile project(path: ':library', configuration:'debug')
- 使用一些Gradle的hook的黑科技

该问题的更详细讨论见：http://www.jianshu.com/p/1907bffef0a3
