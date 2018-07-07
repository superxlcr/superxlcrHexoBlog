---
title: Android windowSoftInputMode属性解析
tags: [android,view]
categories: [android]
date: 2016-05-30 20:42:15
description: Android windowSoftInputMode属性解析
---

windowSoftInputMode为Android中activity在Manifest.xml中设置的属性之一，主要用于解决屏幕软键盘与Activity布局的问题。

官方说明如下：
How the main window of the activity interacts with the window containing the on-screen soft keyboard. The setting for this attribute affects two things:
- The state of the soft keyboard — whether it is hidden or visible — when the activity becomes the focus of user attention.
- The adjustment made to the activity's main window — whether it is resized smaller to make room for the soft keyboard or whether its contents pan to make the current focus visible when part of the window is covered by the soft keyboard.


The setting must be one of the values listed in the following table, or a combination of one "state..." value plus one "adjust..." value. Setting multiple values in either group — multiple "state..." values, for example — has undefined results. Individual values are separated by a vertical bar (|).

大意为，该属性主要用于描述activity窗口与软键盘窗口的交互，设置该属性主要会影响两个方面：


- 软键盘的状态：当Activity被用户获取焦点时，软键盘是显示还是隐藏
- Activity窗口的调整：是否通过缩小原视图来为软键盘获取足够的空间，是否通过覆盖的方式来为软键盘获取足够的空间

设置的属性必须是下表的参数之一，或是由“state...”（改变软键盘状态）和"adjust...”（改变Activity窗口调整状态）组合而成，由“|”符号组合两个参数。


windowSoftInputMode参数表


| 值 | 描述 | 
| - | - |
| stateUnspecified | 软键盘的状态未指明，系统会自动根据选择的主题信息执行相应的行为，是系统默认选项 | 
| stateUnchanged | 当该Activity来到前台时，软键盘保持其原有的状态（在前一个Activity中显示就继续显示，隐藏就继续隐藏） | 
| stateHidden | 当该Activity是被直接打开时，隐藏软键盘，当该Activity是由按下back键打开时，保持软键盘状态 | 
| stateAlwaysHidden | 只要进入该Activity软键盘就会被隐藏 | 
| stateVisible | 当该Activity是被直接打开时，显示软键盘，当该Activity是由按下back键打开时，保持软键盘状态 | 
| stateAlwaysVisible | 只要进入该Activity软键盘就会被显示 | 
| adjustUnspecified | Activity窗口的调整未指明，系统会自动根据选择的主题信息执行相应的行为，是系统默认选项，如果存在ScrollView会使用缩小视图的方式，否则使用覆盖的方式 | 
| adjustResize | 使用缩小视图的方式来为软键盘腾出空间，意味着整体布局底部会上移，空间会缩小，控件可能会挤到一起 | 
| adjustPan | 通过覆盖的方式来为软键盘获取足够的空间，软键盘会覆盖布局底部控件，要是软键盘盖住了当前输入框的时候整体布局会往上移动 | 



