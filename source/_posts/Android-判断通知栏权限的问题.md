---
title: Android 判断通知栏权限的问题
tags: [android]
categories: [android]
date: 2017-11-11 17:06:01
description: Android 判断通知栏权限的问题
---
在Android中，我们可以在设置中关闭某个应用的通知栏权限（Notification Permission）
在关闭通知栏权限后，应用将无法弹出toast以及Notification通知栏
判断应用的通知栏权限是否关闭的代码如下：

```java
public class NotificationsUtils {

    private static final String CHECK_OP_NO_THROW = "checkOpNoThrow";
    private static final String OP_POST_NOTIFICATION = "OP_POST_NOTIFICATION";

    public static boolean isNotificationEnabled(Context context) {

        AppOpsManager mAppOps = (AppOpsManager) context.getSystemService(Context.APP_OPS_SERVICE);

        ApplicationInfo appInfo = context.getApplicationInfo();

        String pkg = context.getApplicationContext().getPackageName();

        int uid = appInfo.uid;

        Class appOpsClass = null; /* Context.APP_OPS_MANAGER */

        try {

            appOpsClass = Class.forName(AppOpsManager.class.getName());

            Method checkOpNoThrowMethod = appOpsClass.getMethod(CHECK_OP_NO_THROW, Integer.TYPE, Integer.TYPE, String.class);

            Field opPostNotificationValue = appOpsClass.getDeclaredField(OP_POST_NOTIFICATION);
            int value = (int)opPostNotificationValue.get(Integer.class);

            return ((int)checkOpNoThrowMethod.invoke(mAppOps,value, uid, pkg) == AppOpsManager.MODE_ALLOWED);

        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        } catch (NoSuchFieldException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
        return false;
    }
}
```


此方法可以成功获取当前应用是否具有通知栏权限，但是存在一个兼容问题：
在Android 4.4（API19）以下是没有AppOpsManager这个类的（见第8行），因此此方法在Android 4.4以下运行会导致应用crash
那么，我们该如何在Android 4.4以下获取应用通知栏权限呢？
很遗憾，答案是不能，详情可以参考stackoverflow：https://stackoverflow.com/questions/11649151/android-4-1-how-to-check-notifications-are-disabled-for-the-application
另外，除了使用上述的反射方法来获取通知栏权限外，我们也可以使用support包提供的方法（也是官方推荐的方法，不过需要比较新的support.v4包）:

```java
NotificationManagerCompat.from(this).areNotificationsEnabled();
```


值得一提的是，由于在API 19以下的版本无法获得通知栏权限，该方法默认会返回true
            