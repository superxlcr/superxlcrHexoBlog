---
title: 查看gradle各任务依赖关系
tags: [gradle]
categories: [gradle]
date: 2020-05-14 22:28:59
description: （转载）查看gradle各任务依赖关系
---

本文转载自：https://blog.csdn.net/u010389578/article/details/80630964

在gradle中一个任务一般都是由多个子任务组成，各个子任务之间存在依赖关系，gradle会按我们预定的依赖关系去选择合适的顺序去执行各个子任务，最终保证任务的正确执行。在使用gradle构建工程的时候，有时我们需要去看某个任务由哪些子任务组成，各个子任务之间存在着什么样的依赖关系。这时可以选择使用下面的命令去执行需要查看的任务，该命令只会按正确的顺序列出所有的子任务而不会去真正执行

```
./gradlew 任务名字 –dry-run（或 -m）
```

运行./gradlew -h可以看到官方的使用说明

```
-m, –dry-run Run the builds with all task actions disabled.
```

我在最初使用gradle的时候不知道Android中的build、assemble、assembleDebug 、assembleRelease等任务的区别。这时就可以使用上面说到的方法，运行./gradlew build –dry-run，列出所有子任务，可以看到所有子任务都是被跳过的，并不会真正执行。

```
:app:preBuild SKIPPED
:app:preDebugBuild SKIPPED
:app:compileDebugAidl SKIPPED
:app:compileDebugRenderscript SKIPPED
:app:checkDebugManifest SKIPPED
.......................................
:app:preReleaseUnitTestBuild SKIPPED
:app:javaPreCompileReleaseUnitTest SKIPPED
:app:compileReleaseUnitTestJavaWithJavac SKIPPED
:app:processReleaseUnitTestJavaRes SKIPPED
:app:testReleaseUnitTest SKIPPED
:app:test SKIPPED
:app:check SKIPPED
:app:build SKIPPED
```

通过对比对列出来的子任务最后发现以下关系：

build = assemble + lint + test相关
assemble = assembleDebug + assembleRelease