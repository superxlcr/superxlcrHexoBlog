---
title: Android 自定义Notification颜色适配问题
tags: [android]
categories: [android]
date: 2018-02-04 16:05:57
description: Android 自定义Notification颜色适配问题
---
最近完成了一个需要自定义RemoteViews的Notification的需求，期间遇到了不少关于颜色适配的问题，在此做一下总结
在Notification上使用我们自定义的RemoteViews时，通知栏的背景色、标题文字色跟内容文字色是我们需要注意的三种颜色，如果设置不当，可能会导致我们自定义的通知栏的通知看不清楚
一般而言，我们可以通过：

不设置自定义view的背景色，标题与内容设置系统的style来解决这个问题：
SDK21以下：

```java
// 标题样式
android:textAppearance="@android:style/TextAppearance.StatusBar.EventContent.Title"
// 内容样式
android:textAppearance="@android:style/TextAppearance.StatusBar.EventContent"
```


SDK21以及以上：

```java
// 标题样式
android:textAppearance="@android:style/TextAppearance.Material.Notification.Title"
// 内容样式
android:textAppearance="@android:style/TextAppearance.Material.Notification.Info"
```


但是实际测试中发现，在某些国产机型上会出现设置错误的标题跟内容文字颜色的问题
因此，这边通过先构造一个系统的Notification，再取出其中对应颜色的方法，写了一个获取通知栏颜色的工具类：

```java
/**
 * Created by superxlcr on 2018/2/4.
 *
 * 获取通知栏颜色工具类
 */

public class NotificationUtils {

    //<editor-fold desc="property">

    private static final String TITLE = "title";
    private static final String CONTENT = "content";
    private static final float COLOR_THRESHOLD = 180f;

    private static int backgroundColor = Color.TRANSPARENT;
    private static int titleColor = Color.WHITE;
    private static int contentColor = Color.WHITE;
    private static boolean checkDeviceColors = false;
    //</editor-fold>

    //<editor-fold desc="public">

    /**
     * 获取Notification背景色
     *
     * @param context 上下文
     * @return 背景色，获取失败时默认为 透明色
     */
    public static int getBackgroundColor(Context context) {
        checkDeviceColors(context);
        return backgroundColor;
    }

    /**
     * 获取Notification标题色
     *
     * @param context 上下文
     * @return 标题色，获取失败时默认为 白色
     */
    public static int getTitleColor(Context context) {
        checkDeviceColors(context);
        return titleColor;
    }

    /**
     * 获取Notification内容色
     *
     * @param context 上下文
     * @return 内容色，获取失败时默认为 白色
     */
    public static int getContentColor(Context context) {
        checkDeviceColors(context);
        return contentColor;
    }
    //</editor-fold>


    //<editor-fold desc="private">

    private NotificationUtils() {
    }

    private static void checkDeviceColors(Context context) {
        if (checkDeviceColors || context == null) {
            return;
        }
        checkDeviceColors = true;
        NotificationCompat.Builder builder = new NotificationCompat.Builder(context);
        Notification notification = builder.setContentTitle(TITLE).setContentText(CONTENT).build();
        View notificationView = notification.contentView.apply(context, new LinearLayout(context));
        if (notificationView instanceof ViewGroup) {
            TextView title = findTextViewByText((ViewGroup) notificationView, TITLE);
            if (title != null) {
                titleColor = title.getCurrentTextColor();
                // 黑色标题使用白色背景，其他标题使用透明背景
                if (isSimilarColor(Color.BLACK, titleColor)) {
                    backgroundColor = Color.WHITE;
                } else {
                    backgroundColor = Color.TRANSPARENT;
                }
            }
            TextView content = findTextViewByText((ViewGroup) notificationView, CONTENT);
            if (content != null) {
                contentColor = content.getCurrentTextColor();
            }
        }
    }

    private static TextView findTextViewByText(ViewGroup viewGroup, String text) {
        if (viewGroup == null) {
            return null;
        }
        int size = viewGroup.getChildCount();
        for (int i = 0; i < size; i++) {
            View view = viewGroup.getChildAt(i);
            if (view instanceof TextView) {
                TextView textView = (TextView) view;
                if (TextUtils.equals(textView.getText(), text)) {
                    return textView;
                }
            } else if (view instanceof ViewGroup) {
                TextView textView = findTextViewByText((ViewGroup) view, text);
                if (textView != null) {
                    return null;
                }
            }
        }
        return null;
    }

    private static boolean isSimilarColor(int baseColor, int color) {
        int simpleBaseColor = baseColor | 0xff000000;
        int simpleColor = color | 0xff000000;
        int baseRed = Color.red(simpleBaseColor) - Color.red(simpleColor);
        int baseGreen = Color.green(simpleBaseColor) - Color.green(simpleColor);
        int baseBlue = Color.blue(simpleBaseColor) - Color.blue(simpleColor);
        double value = Math.sqrt(baseRed * baseRed + baseGreen * baseGreen + baseBlue * baseBlue);
        return value < COLOR_THRESHOLD;
    }
    //</editor-fold>
}
```


其中，由于Android的通知栏底色一般为白色、黑色或者透明，因此在工具类中根据标题文字色的颜色，为背景对应设置白色或透明
            