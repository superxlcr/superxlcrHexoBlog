---
title: Android常见问题总结（八）
tags: [android,基础知识]
categories: [android]
date: 2019-05-06 14:13:12
description: Android全屏方式对比
---
上一篇博客传送门：[Android常见问题总结（七）](/2019/03/29/Android常见问题总结（七）/)

# Android全屏方式对比

博主目前发现两种在Android中实现全屏的方案，分别是：
- 通过 View#setSystemUiVisibility 方法，使用 View#SYSTEM_UI_FLAG_FULLSCREEN
- 通过 Window#addFlags 方法，使用 WindowManager.LayoutParams#FLAG_FULLSCREEN

两者在视觉效果上并没有什么不同，其中，View#SYSTEM_UI_FLAG_FULLSCREEN 说明如下：
```java
    /**
     * Flag for {@link #setSystemUiVisibility(int)}: View has requested to go
     * into the normal fullscreen mode so that its content can take over the screen
     * while still allowing the user to interact with the application.
     *
     * <p>This has the same visual effect as
     * {@link android.view.WindowManager.LayoutParams#FLAG_FULLSCREEN
     * WindowManager.LayoutParams.FLAG_FULLSCREEN},
     * meaning that non-critical screen decorations (such as the status bar) will be
     * hidden while the user is in the View's window, focusing the experience on
     * that content.  Unlike the window flag, if you are using ActionBar in
     * overlay mode with {@link Window#FEATURE_ACTION_BAR_OVERLAY
     * Window.FEATURE_ACTION_BAR_OVERLAY}, then enabling this flag will also
     * hide the action bar.
     *
     * <p>This approach to going fullscreen is best used over the window flag when
     * it is a transient state -- that is, the application does this at certain
     * points in its user interaction where it wants to allow the user to focus
     * on content, but not as a continuous state.  For situations where the application
     * would like to simply stay full screen the entire time (such as a game that
     * wants to take over the screen), the
     * {@link android.view.WindowManager.LayoutParams#FLAG_FULLSCREEN window flag}
     * is usually a better approach.  The state set here will be removed by the system
     * in various situations (such as the user moving to another application) like
     * the other system UI states.
     *
     * <p>When using this flag, the application should provide some easy facility
     * for the user to go out of it.  A common example would be in an e-book
     * reader, where tapping on the screen brings back whatever screen and UI
     * decorations that had been hidden while the user was immersed in reading
     * the book.
     *
     * @see #setSystemUiVisibility(int)
     */
```

主要需要注意的细节有这种方式实现的全屏效果是**短暂的（transient）**，会因为用户的某些操作（如：跳转到其他应用，下拉状态栏等）退出全屏模式

而 WindowManager.LayoutParams#FLAG_FULLSCREEN 说明如下：
```java
        /**
         * Window flag: hide all screen decorations (such as the status bar) while
         * this window is displayed.  This allows the window to use the entire
         * display space for itself -- the status bar will be hidden when
         * an app window with this flag set is on the top layer. A fullscreen window
         * will ignore a value of {@link #SOFT_INPUT_ADJUST_RESIZE} for the window's
         * {@link #softInputMode} field; the window will stay fullscreen
         * and will not resize.
         *
         * <p>This flag can be controlled in your theme through the
         * {@link android.R.attr#windowFullscreen} attribute; this attribute
         * is automatically set for you in the standard fullscreen themes
         * such as {@link android.R.style#Theme_NoTitleBar_Fullscreen},
         * {@link android.R.style#Theme_Black_NoTitleBar_Fullscreen},
         * {@link android.R.style#Theme_Light_NoTitleBar_Fullscreen},
         * {@link android.R.style#Theme_Holo_NoActionBar_Fullscreen},
         * {@link android.R.style#Theme_Holo_Light_NoActionBar_Fullscreen},
         * {@link android.R.style#Theme_DeviceDefault_NoActionBar_Fullscreen}, and
         * {@link android.R.style#Theme_DeviceDefault_Light_NoActionBar_Fullscreen}.</p>
         */
```

主要需要注意的是跟软键盘弹出的一些交互问题