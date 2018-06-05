---
title: Android 沉浸式适配
tags: []
categories: []
description: 什么是沉浸式、修改状态栏颜色、全屏导致的键盘弹出问题、修改状态栏文字颜色
date: 2018-05-19 10:52:18
---

最近公司的产品需要进行一波沉浸式UI的改动，在此记录一下躺过的坑

# 什么是沉浸式

传统的手机状态栏是呈现出黑色条状的，有的和手机主界面有很明显的区别。这一样就在一定程度上牺牲了视觉宽度，界面面积变小。
沉浸式是APP界面图片延伸到状态栏， 应用本身沉浸于状态栏，如下图所示：

![沉浸式示意图](1.png)

因此，应用想要实现沉浸式的体验，我们主要解决的问题有：
- 对于一般的toolbar，我们需要修改状态栏的背景颜色
- 对于其他非纯色作为顶部的页面，我们需要把页面布局延伸至状态栏，这就需要解决全屏带来的键盘无法弹出的问题
- 对于某些顶部较明亮的配色方案，我们还需要考虑状态栏文字颜色的转换

下面我们来分别讨论下这三个问题

# 修改状态栏颜色

首先，对于api 19 (Android 4.4)以下的状态栏，由于既不支持状态栏透明，也不支持设置状态栏颜色，因此是没法实现沉浸式的

然后，对于api 21 (Android 5.0)以下的状态栏，由于不支持设置状态栏颜色，因此我们只能通过把状态栏设置成透明实现沉浸式：
```java
activity.getWindow().addFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS);
```

不过在添加了状态栏透明的flag之后，会导致应用变为全屏模式，因此我们实际的toolbar会被延伸至状态栏，如下图所示：
![toolbar延伸示意图](2.png)
因此，这种情况我们需要自行写一个状态栏去填充位置，并添加相应的padding保持界面布局不与状态栏混在一起
综上，对于api 21 (Android 5.0)以下的状态栏，实现代码如下：
```java
	@TargetApi(Build.VERSION_CODES.KITKAT)
    public void setApi19StatusBarColor(Activity activity, @ColorInt int color) {
        activity.getWindow().addFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS);
        View fakeStatusBar = addFakeStatusBar(activity);
        fakeStatusBar.setBackgroundColor(color);
    }

    private View addFakeStatusBar(Activity activity) {
        int statusBarHeight = getStatusBarHeight(activity);
        View fakeStatusBarView = new View(activity);
        FrameLayout.LayoutParams params = new FrameLayout.LayoutParams(
                ViewGroup.LayoutParams.MATCH_PARENT, statusBarHeight);
        FrameLayout decorView = (FrameLayout) activity.getWindow().getDecorView();
        decorView.addView(fakeStatusBarView, params);
        ViewGroup contentView = decorView.findViewById(android.R.id.content);
        for (int i = 0; i < contentView.getChildCount(); i++) {
            View child = contentView.getChildAt(i);
            child.setPadding(child.getPaddingLeft(), child.getPaddingTop() + statusBarHeight,
                    child.getPaddingRight(), child.getPaddingBottom());
            if (child instanceof ViewGroup) {
                ((ViewGroup) child).setClipToPadding(true);
            }
        }
        return fakeStatusBarView;
    }

    private int getStatusBarHeight(Activity activity) {
        int statusBarHeight = 0;
        int resourceId = getResources().getIdentifier("status_bar_height", "dimen", "android");
        if (resourceId > 0) {
            //根据资源ID获取响应的尺寸值
            statusBarHeight = getResources().getDimensionPixelSize(resourceId);
        }
        return statusBarHeight;
    }
```

对于api 21 (Android 5.0)以上的状态栏，由于系统已经为沉浸式提供了良好的支持，直接调用相应的api即可：
```java
	@TargetApi(Build.VERSION_CODES.LOLLIPOP)
    public void setApi21StatusBarColor(Activity activity, @ColorInt int color) {
        activity.getWindow().addFlags(WindowManager.LayoutParams.FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS);
        activity.getWindow().clearFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS);
        activity.getWindow().setStatusBarColor(color);
    }
```

## 如何把页面布局延伸至状态栏

想要把页面布局延伸至状态栏，我们可以通过设置 FLAG_TRANSLUCENT_STATUS 这个flag来完成：
```java
		getWindow().addFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS);
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
            getWindow().clearFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS);
            getWindow().addFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_NAVIGATION);
            getWindow().setStatusBarColor(Color.TRANSPARENT);
        }
```
对于api 21 (Android 5.0)以上的状态栏，我们还可以通过 FLAG_TRANSLUCENT_NAVIGATION 这个flag跟 setStatusBarColor 这个方法来吧状态栏的阴影去掉达到更好的显示效果

# 全屏导致的键盘弹出问题

当我们通过设置 FLAG_TRANSLUCENT_STATUS 这个flag来把页面布局延伸至状态栏后，我们会发现一个新的问题：
对于某些底部拥有输入栏EditText的界面，这个flag属性会导致我们设置的windowSoftInputMode的adjustResize属性失效，导致底部的EditText输入栏无法顶到键盘之上：
![bottombar示意图](3.png)
![键盘覆盖bottombar示意图](4.png)

这种情况下，我们就可以通过重写顶部ViewGroup的 onApplyWindowInsets 以及 fitSystemWindows 方法解决
在设置了fitsSystemWindows属性后，通过重写上述两个方法把除下方外的系统边距去除：
```java
public class MyRelativeLayout extends RelativeLayout {

    public MyRelativeLayout(Context context) {
        super(context);
        setFitsSystemWindows(true);
    }

    public MyRelativeLayout(Context context, AttributeSet attrs) {
        super(context, attrs);
        setFitsSystemWindows(true);
    }

    public MyRelativeLayout(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        setFitsSystemWindows(true);
    }

    @Override
    public WindowInsets onApplyWindowInsets(WindowInsets insets) {
        if (Build.VERSION.SDK_INT > Build.VERSION_CODES.KITKAT) {
            WindowInsets onlyBottomInsets = insets.replaceSystemWindowInsets(0, 0, 0,
                    insets.getSystemWindowInsetBottom());
            return super.onApplyWindowInsets(onlyBottomInsets);
        }
        return super.onApplyWindowInsets(insets);
    }

    @Override
    protected boolean fitSystemWindows(Rect insets) {
        insets.top = 0;
        insets.left = 0;
        insets.right = 0;
        return super.fitSystemWindows(insets);
    }
}
```

# 修改状态栏文字颜色

由于Android状态栏文字默认的颜色是白色，在我们自定义状态栏的颜色后，对于某些配色较明亮的状态栏背景色而言，可能会导致状态栏文字颜色看不清
此时，我们就需要通过去修改状态栏文字颜色来解决这个问题

对于Android 6.0 以下的系统，我们就只能够修改已经公布了相应api的魅族跟小米系统的状态栏文字颜色，而6.0以上的系统则可以通过系统提供的api来解决问题：
```java
public class StatusTextHelper {

    private static IStatusTextManager manager = null;

    static {
        try {
            WindowManager.LayoutParams.class.getDeclaredField("MEIZU_FLAG_DARK_STATUS_BAR_ICON");
            WindowManager.LayoutParams.class.getDeclaredField("meizuFlags");
            manager = new FlymeStatusTextManagerImpl();
        } catch (Exception e) {

        }

        try {
            Class.forName("android.view.MiuiWindowManager$LayoutParams");
            manager = new MIUIStatusTextManagerImpl();
        } catch (Exception e) {

        }

        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
            manager = new AndroidMStatusTextManagerImpl();
        }
    }

    /**
     * 是否能设置状态栏文字颜色
     *
     * @return boolean
     */
    public static boolean canSetStatusTextMode() {
        return manager != null;
    }

    /**
     * 设置状态栏文件颜色
     *
     * @param window 窗体
     * @param dark   是否为黑色
     */
    public static void setStatusTextMode(Window window, boolean dark) {
        if (manager != null) {
            manager.updateStatusTextMode(window, dark);
        }
    }

    private static class FlymeStatusTextManagerImpl implements IStatusTextManager {
        @Override
        public void updateStatusTextMode(Window window, boolean dark) {
            if (window != null) {
                try {
                    WindowManager.LayoutParams lp = window.getAttributes();
                    Field darkFlag = WindowManager.LayoutParams.class
                            .getDeclaredField("MEIZU_FLAG_DARK_STATUS_BAR_ICON");
                    Field meizuFlags = WindowManager.LayoutParams.class
                            .getDeclaredField("meizuFlags");
                    darkFlag.setAccessible(true);
                    meizuFlags.setAccessible(true);
                    int bit = darkFlag.getInt(null);
                    int value = meizuFlags.getInt(lp);
                    if (dark) {
                        value |= bit;
                    } else {
                        value &= ~bit;
                    }
                    meizuFlags.setInt(lp, value);
                    window.setAttributes(lp);
                } catch (Exception e) {

                }
            }
        }
    }

    private static class MIUIStatusTextManagerImpl implements IStatusTextManager {
        @Override
        public void updateStatusTextMode(Window window, boolean dark) {
            if (window != null) {
                Class clazz = window.getClass();
                try {
                    int darkModeFlag;
                    Class layoutParams = Class.forName(
                            "android.view.MiuiWindowManager$LayoutParams");
                    Field field = layoutParams.getField("EXTRA_FLAG_STATUS_BAR_DARK_MODE");
                    darkModeFlag = field.getInt(layoutParams);
                    Method extraFlagField = clazz.getMethod("setExtraFlags", int.class, int.class);
                    if (dark) {
                        extraFlagField.invoke(window, darkModeFlag, darkModeFlag);//状态栏透明且黑色字体
                    } else {
                        extraFlagField.invoke(window, 0, darkModeFlag);//清除黑色字体
                    }

                    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
                        //开发版 7.7.13 及以后版本采用了系统API，旧方法无效但不会报错，所以两个方式都要加上
                        if (dark) {
                            window.getDecorView().setSystemUiVisibility(
                                    View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN | View.SYSTEM_UI_FLAG_LIGHT_STATUS_BAR);
                        } else {
                            window.getDecorView().setSystemUiVisibility(
                                    View.SYSTEM_UI_FLAG_VISIBLE);
                        }
                    }
                } catch (Exception e) {

                }
            }
        }
    }

    private static class AndroidMStatusTextManagerImpl implements IStatusTextManager {
        @Override
        public void updateStatusTextMode(Window window, boolean dark) {
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
                if (dark) {
                    window.getDecorView().setSystemUiVisibility(
                            View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN | View.SYSTEM_UI_FLAG_LIGHT_STATUS_BAR);
                } else {
                    window.getDecorView().setSystemUiVisibility(
                            View.SYSTEM_UI_FLAG_VISIBLE);
                }
            }
        }
    }

    interface IStatusTextManager {
        void updateStatusTextMode(Window window, boolean dark);
    }

}
```