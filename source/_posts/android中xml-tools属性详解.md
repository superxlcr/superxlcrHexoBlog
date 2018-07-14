---
title: android中xml tools属性详解
tags: [android,资源文件]
categories: [android]
date: 2017-08-27 10:28:59
description: （转载）android中xml tools属性详解
---

本文转载自：http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2015/0309/2567.html

# 第一部分

安卓开发中，在写布局代码的时候，ide可以看到布局的预览效果。
![_pic1](1.png)

但是有些效果则必须在运行之后才能看见，比如这种情况：TextView在xml中没有设置任何字符，而是在activity中设置了text。因此为了在ide中预览效果，你必须在xml中为TextView控件设置android:text属性

```html
<TextView
  android:id="@+id/text_main"
  android:layout_width="match_parent"
  android:layout_height="wrap_content"
  android:textAppearance="@style/TextAppearance.Title"
  android:layout_margin="@dimen/main_margin"
  android:text="I am a title" />
```


一般我们在这样做的时候都告诉自己，没关系，等写完代码我就把这些东西一并删了。但是你可能会忘，以至于在你的最终产品中也会有这样的代码。

## 用tools吧，别做傻事

以上的情况是可以避免的，我们使用tools命名空间以及其属性来解决这个问题。

```html
xmlns:tools="http://schemas.android.com/tools"
```


tools可以告诉Android Studio，哪些属性在运行的时候是被忽略的，只在设计布局的时候有效。比如我们要让android:text属性只在布局预览中有效可以这样

```html
<TextView
 android:id="@+id/text_main"
 android:layout_width="match_parent"
 android:layout_height="wrap_content"
 android:textAppearance="@style/TextAppearance.Title"
 android:layout_margin="@dimen/main_margin"
 tools:text="I am a title" />
```


tools可以覆盖android的所有标准属性，将android:换成tools:即可。同时在运行的时候就连tools:本身都是被忽略的，不会被带进apk中。

## tools属性的种类

tools属性可以分为两种：一种是影响Lint提示的，一种是关于xml布局设计的。以上介绍的是tools的最基本用法：在UI设计的时候覆盖标准的android属性，属于第二种。下面介绍Lint相关的属性。
**Lint相关的属性**

```html
tools:ignore
tools:targetApi
tools:locale
```


**tools:ignore**
ignore属性是告诉Lint忽略xml中的某些警告。
假设我们有这样的一个ImageView

```html
<ImageView
  android:layout_width="wrap_content"
  android:layout_height="wrap_content"
  android:layout_marginStart="@dimen/margin_main"
  android:layout_marginTop="@dimen/margin_main"
  android:scaleType="center"
  android:src="@drawable/divider" />
```


Lint会提示该ImageView缺少android:contentDescription属性。我们可以使用tools:ignore来忽略这个警告：
```html
<ImageView
  android:layout_width="wrap_content"
  android:layout_height="wrap_content"
  android:layout_marginStart="@dimen/margin_main"
  android:layout_marginTop="@dimen/margin_main"
  android:scaleType="center"
  android:src="@drawable/divider"
  tools:ignore="contentDescription" />
```



**tools:targetApi**
假设minSdkLevel 15，而你使用了api21中的控件比如RippleDrawable

```html
<ripple xmlns:android="http://schemas.android.com/apk/res/android"
  android:color="@color/accent_color" />
```


则Lint会提示警告。
为了不显示这个警告，可以：
```html
<ripple xmlns:android="http://schemas.android.com/apk/res/android"
  xmlns:tools="http://schemas.android.com/tools"
  android:color="@color/accent_color"
  tools:targetApi="LOLLIPOP" />
```

**tools:locale（本地语言）属性**
默认情况下res/values/strings.xml中的字符串会执行拼写检查，如果不是英语，会提示拼写错误，通过以下代码来告诉studio本地语言不是英语，就不会有提示了。

```html
<resources
  xmlns:android="http://schemas.android.com/apk/res/android"
  xmlns:tools="http://schemas.android.com/tools"
  tools:locale="it">
 
  <!-- Your strings go here -->
 
</resources>
```




这篇文章首先介绍了tools的最基本用法-覆盖android的属性，然后介绍了忽略Lint提示的属性。下篇文章中，我们将继续介绍关于UI预览的其他属性（非android标准属性）。
ps：关于忽略Lint的属性，如果不想了解的话也没关系，因为并不影响编译，一般我都不会管这些警告。



# 第二部分

这部分我们将继续介绍关于UI预览的其他属性（非android标准属性）。
- tools:context
- tools:menu
- tools:actionBarNavMode
- tools:listitem/listheader/listfooter
- tools:showIn
- tools:layout


**tools:context**
context属性其实正是的称呼是activity属性，有了这个属性，ide就知道在预览布局的时候该采用什么样的主题。同时他还可以在android studio的java代码中帮助找到相关的文件（Go to Related files）
![_pic2](2.png)

该属性的值是activity的完整包名

```html
<LinearLayout
  xmlns:android="http://schemas.android.com/apk/res/android"
  xmlns:tools="http://schemas.android.com/tools"
  android:id="@+id/container"
  android:layout_width="match_parent"
  android:layout_height="match_parent"
  android:orientation="vertical"
  tools:context="com.android.example.MainActivity">  <!-- ... -->
</LinearLayout>
```



**tools:menu**
告诉IDE 在预览窗口中使用哪个菜单，这个菜单将显示在layout的根节点上（actionbar的位置）。
![_pic3](3.png)

其实预览窗口非常智能，如果布局和一个activity关联（指上面所讲的用tools:context关联）它将会自动查询相关activity的onCreateOptionsMenu方法中的代码，以显示菜单。而**menu属性**则可以覆盖这种默认的行为。
你还可以为**menu属性**定义多个菜单资源，不同的菜单资源之间用逗号隔开。

```html
tools:menu="menu_main,menu_edit"
```


如果你不希望在预览图中显示菜单则：

```html
tools:menu=""
```


最后需要注意，当主题为Theme.AppCompat时，这个属性不起作用。


**tools:actionBarNavMode**
这个属性告诉ide  app bar（Material中对actionbar的称呼）的显示模式，其值可以是

```html
standard
tabs
list
```


```html
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:orientation="vertical"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:actionBarNavMode="tabs" />
```


同样的，当主题是Theme.AppCompat (r21+, at least)*或者Theme.Material*,或者使用了布局包含Toolbar的方式。  该属性也不起作用，只有holo主题才有效。


**listitem, listheader 和listfooter 属性**

顾名思义就是在ListView ExpandableListView等的预览效果中添加头部 尾部 以及子item的预览布局。
```html
<GridView
 android:id="@+id/list"
 android:layout_width="match_parent"
 android:layout_height="wrap_content"
 tools:listheader="@layout/list_header"
 tools:listitem="@layout/list_item"
 tools:listfooter="@layout/list_footer" />
```

 
**layout属性**
tools:layout告诉ide，Fragment在程序预览的时候该显示成什么样

```html
<fragment xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/item_list"
    android:name="com.example.fragmenttwopanel.ItemListFragment"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:layout_marginLeft="16dp"
    android:layout_marginRight="16dp"
    tools:layout="@android:layout/list_content" />
```


![_pic4](4.png)



**tools:showIn**


该属性设置于一个被其他布局&lt;include&gt;的布局的根元素上。这让您可以指向包含此布局的其中一个布局，在设计时这个被包含的布局会带着周围的外部布局被渲染。这将允许您“在上下文中”查看和编辑这个布局。需要 Studio 0.5.8 或更高版本。



关于tools 就介绍完了。

注：原文是两篇文章  

[Tools of the trade — Part 1](https://medium.com/sebs-top-tips/tools-of-the-trade-part-1-f3c1c73de898) 

[Tools of the trade — Part 2](https://medium.com/sebs-top-tips/tools-of-the-trade-part-2-b91271892d10) 。


觉得完全可以在一篇文章中讲完，就翻译在了一起，原文有很多和内容无关的gif图，描述也比较啰嗦，都被我去掉了，这篇文章属于意译。

