---
title: Java Enum枚举替代方案--Android IntDef/StringDef Annotation注解
tags: [android]
categories: [android]
date: 2017-08-23 15:48:17
description: （翻译）Java Enum枚举替代方案--Android IntDef/StringDef Annotation注解
---
原文链接：https://noobcoderblog.wordpress.com/2015/04/12/java-enum-and-android-intdefstringdef-annotation/


当我们想把一个变量x的取值限制在几个预先定义的常量时，我们会怎么做呢？我们可以先定义一些常量值，然后从这些常量中选择赋值给x。下面，让我们假设变量x为currentDay，它的取值包含了星期天到星期五。我们可以在Java中，通过Integer的常量写出下面的代码：

```java
public class Main {
 
    public static final int SUNDAY = 0;
    public static final int MONDAY = 1;
    public static final int TUESDAY = 2;
    public static final int WEDNESDAY = 3;
    public static final int THURSDAY = 4;
    public static final int FRIDAY = 5;
    public static final int SATURDAY = 6;
 
    private int currentDay = SUNDAY;
 
    public static void main(String[] args) {
        Main obj = new Main();
        obj.setCurrentDay(WEDNESDAY);
 
        int today = obj.getCurrentDay();
 
        switch (today) {
        case SUNDAY:
            System.out.println("Today is SUNDAY");
            break;
        case MONDAY:
            System.out.println("Today is MONDAY");
            break;
        case TUESDAY:
            System.out.println("Today is TUESDAY");
            break;
        case WEDNESDAY:
            System.out.println("Today is WEDNESDAY");
            break;
        case THURSDAY:
            System.out.println("Today is THURSDAY");
            break;
        case FRIDAY:
            System.out.println("Today is FRIDAY");
            break;
        case SATURDAY:
            System.out.println("Today is SATURDAY");
            break;
 
        default:
            break;
        }
    }
 
    public void setCurrentDay(int currentDay) {
        this.currentDay = currentDay;
    }
 
    public int getCurrentDay() {
        return currentDay;
    }
 
}
```


但上面的代码会出现的问题是：我可以为currentDay设置任何的int值

```java
obj.setCurrentDay(100);
```


这种情况编译器并不会给出任何的错误提示。然后我们的switch/case语句会忽略掉处理这种情况。因此，对于这种情况，Java为我们提供了一种叫Enumeration（或称Enum，枚举）的解决方案。如果使用了Enum枚举，我们的Java代码将会写成下面这样：

```java
public class Main {
 
    public enum WeekDays {
        SUNDAY, MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY, SATURDAY
    }
 
    private WeekDays currentDay = WeekDays.SUNDAY;
 
    public static void main(String[] args) {
        // TODO Auto-generated method stub
        Main obj = new Main();
        obj.setCurrentDay(WeekDays.WEDNESDAY);
 
        WeekDays today = obj.getCurrentDay();
 
        switch (today) {
        case SUNDAY:
            System.out.println("Today is SUNDAY");
            break;
        case MONDAY:
            System.out.println("Today is MONDAY");
            break;
        case TUESDAY:
            System.out.println("Today is TUESDAY");
            break;
        case WEDNESDAY:
            System.out.println("Today is WEDNESDAY");
            break;
        case THURSDAY:
            System.out.println("Today is THURSDAY");
            break;
        case FRIDAY:
            System.out.println("Today is FRIDAY");
            break;
        case SATURDAY:
            System.out.println("Today is SATURDAY");
            break;
 
        default:
            break;
        }
    }
 
    public void setCurrentDay(WeekDays currentDay) {
        this.currentDay = currentDay;
    }
 
    public WeekDays getCurrentDay() {
        return currentDay;
    }
 
}
```


现在我们拥有了类型安全的保障。这里我们不能再为currentDay赋予任何在WeekDays以外的值了。这是一个很大的进步，我们都应该使用多多使用。不过在Android中，这里会存在一些问题。
Enum枚举在Android中：Enum在Java中是一个完全成熟的class。在Enum枚举中的每一个值，都是Enum枚举类型中的一个对象实例。因此，Enum枚举值会比我们之前使用的常量类型占用更多的内存。即使在旧Android设备（版本 &lt;= 2.2）上，这里也存在一些由JIT即使编译器解决的，由Enum枚举类型引发的性能问题。现在我们可以在Android应用中使用Enum枚举类型，但如果我们的应用是一种非常吃紧内存的类型或者是游戏应用，那么我们最后使用int常量来代替Enum枚举。但这导致我们上面提及的问题依然存在。


现在我们有了别的解决方案。Android的support Annotation注解库有一些很好的annotation注解来帮助我们更早的发现bug（在编译时期）。IntDef 和 StringDef 是两个非常有魔力的Constant Annotation注解，我们可以使用它们来代替Enum枚举。它们会帮助我们像Enum枚举一样，在编译时期检查变量的赋值情况。下面的代码展示给我们如何使用IntDef代替Enum：

```java
public class MainActivity extends Activity {
 
    public static final int SUNDAY = 0;
    public static final int MONDAY = 1;
    public static final int TUESDAY = 2;
    public static final int WEDNESDAY = 3;
    public static final int THURSDAY = 4;
    public static final int FRIDAY = 5;
    public static final int SATURDAY = 6;
 
    @IntDef({SUNDAY, MONDAY,TUESDAY,WEDNESDAY,THURSDAY,FRIDAY,SATURDAY})
    @Retention(RetentionPolicy.SOURCE)
    public @interface WeekDays {}
 
    @WeekDays int currentDay = SUNDAY;
 
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        setCurrentDay(WEDNESDAY);
 
        @WeekDays int today = getCurrentDay();
 
        switch (today){
            case SUNDAY:
                break;
            case MONDAY:
                break;
            case TUESDAY:
                break;
            case WEDNESDAY:
                break;
            case THURSDAY:
                break;
            case FRIDAY:
                break;
            case SATURDAY:
                break;
            default:
                break;
        }
 
    }
 
    public void setCurrentDay(@WeekDays int currentDay) {
        this.currentDay = currentDay;
    }
 
    @WeekDays
    public int getCurrentDay() {
        return currentDay;
    }
}
```


现在我们不能再为currentDay和today赋予任何在WeekDays以外的值了。编译器会检查变量的赋值情况，并反馈给我们相应的错误信息。如果我们使用Android Studio，IDE还会向我们提供变量建议的功能（代码提示）。当我们使用时，首先需要定义一些常量：

```java
public static final int SUNDAY = 0;
public static final int MONDAY = 1;
public static final int TUESDAY = 2;
public static final int WEDNESDAY = 3;
public static final int THURSDAY = 4;
public static final int FRIDAY = 5;
public static final int SATURDAY = 6;
```


然后用@IntDef注解声明这些变量

```java
@IntDef({SUNDAY, MONDAY,TUESDAY,WEDNESDAY,THURSDAY,FRIDAY,SATURDAY})
@Retention(RetentionPolicy.SOURCE)
public @interface WeekDays {}
```

我们可以通过下面的代码，设置一个变量为WeekDays类型，让WeekDays以外的值都无法赋给该变量

```java
@WeekDays int currentDay ;
```


现在，当我们想为currentDay赋予任何在WeekDays以外的值时，编译器会提示我们相应的错误信息。设置方法的参数以及返回值为WeekDays的方法如下：

```java
public void setCurrentDay(@WeekDays int currentDay) {
    this.currentDay = currentDay;
}
 
@WeekDays
public int getCurrentDay() {
    return currentDay;
```


@StringDef也能以同样的方式应用

```java
public class MainActivity extends Activity {
 
    public static final String SUNDAY = "sunday";
    public static final String MONDAY = "monday";
    public static final String TUESDAY = "tuesday";
    public static final String WEDNESDAY = "wednesday";
    public static final String THURSDAY = "thursday";
    public static final String FRIDAY = "friday";
    public static final String SATURDAY = "saturday";
 
 
    @StringDef({SUNDAY, MONDAY,TUESDAY,WEDNESDAY,THURSDAY,FRIDAY,SATURDAY})
    @Retention(RetentionPolicy.SOURCE)
    public @interface WeekDays {}
 
 
    @WeekDays String currentDay = SUNDAY;
 
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        setCurrentDay(WEDNESDAY);
 
        @WeekDays String today = getCurrentDay();
 
 
        switch (today){
            case SUNDAY:
                break;
            case MONDAY:
                break;
            case TUESDAY:
                break;
            case WEDNESDAY:
                break;
            case THURSDAY:
                break;
            case FRIDAY:
                break;
            case SATURDAY:
                break;
            default:
                break;
        }
 
    }
 
    public void setCurrentDay(@WeekDays String currentDay) {
        this.currentDay = currentDay;
    }
 
    @WeekDays
    public String getCurrentDay() {
        return currentDay;
    }
}
```


想要使用以上的功能，你需要为你的工程添加support-annotation库的依赖。如果你使用的是Android Studio，那么请在你的Gradle依赖脚本下添加下面代码

```java
dependencies {
    compile fileTree(include: ['*.jar'], dir: 'libs')
    ...
    compile 'com.android.support:support-annotations:22.0.0'
}
```


你可以点击[这里](https://developer.android.com/studio/write/annotations.html)查看更多关于Android support annotation库的内容。
            