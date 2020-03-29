---
title: gradle插件编写（二）
tags: [gradle]
categories: [gradle]
date: 2020-03-29 12:17:45
description: Transform执行顺序、buildSrc放置本地插件代码、Transform增量编译、Transform处理一些非.class文件
---

上一篇博客可见：[gradle插件编写](2018/09/06/gradle插件编写/)

博主最近有在编写gradle相关的插件，总结了一些新的相关问题，在此记录一下

# Transform执行顺序

从1.5.0-beta1开始，Android gradle plugin 加入了 [Transform API](http://tools.android.com/tech-docs/new-build-system/transform-api)，允许第三方插件在 class 文件转换成 dex 文件之前来操控 class 文件，这样就很容易在不入侵代码的情况下实现一些 AOP 的操作

![Android编译流程图](1.png)

这里 Android 官方也实现了很多 transform，这里有一个DesugarTransform，是 Android 给使用了 java8 的代码做脱糖处理的，所以说在 Android 上面使用 java8，其实只是语法糖而已

![官方Transform列表](2.png)

下图给出了 transform 的工作方式，即上一个 transform 的输出作为下一个 transform 的输入。

![Transform工作方式](3.png)

既然这样，那么肯定有先后顺序的问题，比如我们自定义了一个 CustomTransfrom，作用是修改某个类中的某些东西，但是官方又实现了一个 ProGuardTransform 用来处理代码混淆，如果是先执行 ProGuardTransform ，后执行 CustomTransfrom ，那么代码先被混淆了，包名类名都变了， CustomTransfrom 肯定不生效了。那么怎么控制这些 transform 的执行顺序呢？

答案在 com.android.build.gradle.internal.TaskManager#createPostCompilationTasks(VariantScope)，代码太长就不贴了，他会在编译完成之后，准备生成 dex 文件之前

因此Transform执行顺序如下：
Desugar -> MergeJavaRes -> **apply all the external transforms** -> Android studio profiling transforms -> JavaCodeShrinker (Proguard / BuiltInShrinker)-> ResourcesShrinker -> ……

所以，我们大概率不用担心我们自定义的 transform 和 Android 官方提供的 transform 冲突，而在 apply all the external transforms 这一步，就是循环遍历项目中引用的第三方的自定义的 plugin 中的 transform ，先后执行顺序也就提现在 build.gradle 中 apply plugin: xxx 的先后顺序上。**谁写在前面，谁就先执行。**

# buildSrc放置本地插件代码

当我们需要应用自定义插件时，尤其在自测阶段，往往需要上传到本地的maven仓库，然后再下载下来应用

除此以外，我们其实还可以在需要应用插件的目标project中，创建**名称为buildSrc的module**，这样gradle会自动把该module下的代码编译依赖为gradle相关插件的仓库代码

# Transform增量编译

当我们自定义Transform的时候，一个高效的自定义Transform应该是能支持增量编译的

首先我们需要在返回的配置中支持增量编译：
```
public boolean isIncremental() { // 是否支持增量返回 true
	return true
}
```

改动文件列表对应的接口api如下：
```
public interface DirectoryInput extends QualifiedContent {
    Map<File, Status> getChangedFiles();
}
```

我们可以通过以下代码，获取本次增量编译中改动过的文件：
```
input.directoryInputs.each { directoryInput ->
	directoryInput.changedFiles.each{ changeFileEntry->
        def status = changeFileEntry.value;
    }
}
```
这样我们可以遍历所有改动的文件，而且可以获取每个改动文件的状态，有4种：
```
public enum Status {
    NOTCHANGED,
    ADDED,
    CHANGED,
    REMOVED;

    private Status() {
    }
}
```

JarInput 和DirectoryInput不同，JarInput只能获取状态，也有4种状态：
```
public interface JarInput extends QualifiedContent {
    Status getStatus();
}
```

PS：值得注意的是，根据网上查到的一些资料说，删除一个java文件，对应的class文件输入不会出现REMOVED状态，也就是不能从changeFiles里面获取被删除的文件

# Transform处理一些非.class文件

当我们实现自定义Transform的时候，需要定义该Transform处理的文件类型，一般来说我们要处理的都是class文件，就返回TransformManager.CONTENT_CLASS，如果我们是想要处理资源文件，可以使用TransformManager.CONTENT_RESOURCES，这里按需要来就好

最近博主需要在Transform中处理非.class相关的文件（一些apt生成的非.class的文件，也可理解为资源文件）

比较奇怪的一点是：
- 在jar包中的资源文件，需要我们定义处理类型为 TransformManager.CONTENT_RESOURCES 才返回
- 而在我们编译主module下的文件，即在Transform中以dir文件夹输入的资源文件，则是定义处理类型为 TransformManager.CONTENT_CLASS 才返回

其中具体的处理逻辑跟处理class文件没有什么特别大的区别，只是获取输出文件的地址时把contentTypes定义为资源类型即可

比较疑惑，在此先记录一下