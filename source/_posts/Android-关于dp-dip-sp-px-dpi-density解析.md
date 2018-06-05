---
title: Android 关于dp dip sp px dpi density解析
tags: [android,资源文件]
categories: [android]
date: 2016-03-12 11:42:16
description: px、分辨率、dpi、density、dp、sp等概念介绍
---
# px

px即像素（Pixel），1px代表了手机屏幕上一个物理的像素点。由于以px为单位的控件在不同手机上显示大小不一定相同，故Android不推荐使用px来设置控件大小：
![像素示例图](1.png)

# 分辨率

分辨率通常表示为横轴像素长度和纵轴像素长度的乘积，如320*480等。

# dpi

dpi的全称是Dots Per Inch，即点每英寸，一般被称为像素密度，它代表了一英寸里面有多少个像素点。计算方法为屏幕总像素点（即分辨率的乘积除以屏幕大小），常见的取值有120,160,240。
举例：
![dpi示例图](2.png)
比如说这里有一款1920*1200分辨率的7寸平板，根据勾股定理，我们可以算到对角线的像素点约为2264，则其像素密度（dpi）为2264 / 7 = 323

# density

density直译为密度，它的计算公式为屏幕dpi除以160点每英寸，由于单位除掉了，故density只是一个比值，常见取值为1.0,1.5等。
在Android中我们可以通过下面代码获取当前屏幕的density：
```java
getResources().getDisplayMetrics().density;
```

# dp（dip）

dp，也叫做dip，全称为Density independent pixels，叫做设备独立像素。他是Android为了解决众多手机dpi不同所定义的单位，谷歌官方的解释如下：
Density-independent pixel (dp) 
　　A virtual pixel unit that you should use when defining UI layout, to express layout dimensions or position in a density-independent way. 
The density-independent pixel is equivalent to one physical pixel on a 160 dpi screen, which is the baseline density assumed by the system for a "medium" density screen. At runtime, the system transparently handles any scaling of the dp units, as necessary, based on the actual density of the screen in use. The conversion of dp units to screen pixels is simple: px = dp * (dpi / 160). For example, on a 240 dpi screen, 1 dp equals 1.5 physical pixels. You should always use dp units when defining your application's UI, to ensure proper display of your UI on screens with different densities.
从上文我们可以看出，dp是一种虚拟抽象的像素单位，他的计算公式为：px = dp * (dpi / 160) = dp * density。因此在dpi大小为160的手机上，1dp = 1px，而在dpi大小为320的手机上，1dp = 2px，即在屏幕越大的手机上，1dp代表的像素也越大。因此我们定义控件大小的时候应该使用dp代替使用px。

# sp

sp是Android中定义字体大小的一种单位，全称为Scaled Pixels，叫做放大像素。sp会根据用户手机上设定的字体大小而改变，在用户手机字体大小设置为正常的情况下，1sp = 1dp。
sp与px之间的密度比例可以通过如下代码获取：
```java
getResources().getDisplayMetrics().scaledDensity;
```

# 资源文件分辨率

一般而言，我们存放资源文件的目录（res）会有多个子目录，这些子目录代表了不同系统屏幕分辨率：

| 密度 | 中文 | dpi | 分辨率 | 比例 |
| - | - | - | - | - |
| ldpi | 低分辨率 | 120以下 | 240*320 | 3 |
| mdpi | 中分辨率 | 120~160 | 320*480 | 4 |
| hdpi | 高分辨率 | 160~240 | 480*800 | 6 |
| xhdpi | 超高分辨率 | 240~320 | 720*1280 | 8 |
| xxhdpi | 超超高分辨率 | 320~480 | 1080*1920 | 12 |
| xxxhdpi | 超超超高分辨率 | 480~640 | 3840*2160 | 16 |

当我们在手机上加载资源时，系统首先会从手机对应分辨率等级的子目录下找资源文件，如果找不到的情况下，会使用别的分辨率的文件进行缩放处理。

# 找不到对应分辨率资源文件情况

对于drawable资源，当应用在设备对应dpi目录下没有找到某个资源时，遵循“先高再低”原则，会从附近的分辨率获取图片，然后按比例进行缩放：

比如，当前为xhdpi设备，并且只有以下几个目录，则drawable的寻找顺序为： 
xhdpi->xxhdpi->xxxhdpi(如果没有更高的了)->nodpi(如果有的话)->hdpi->mdpi，如果在xxhdpi中找到目标图片，则压缩2/3来使用，如果在mdpi中找到图片，则放大2倍来使用。

因此，以现在主流设备来说一般可能在drawable-xxhdpi放置一份即可，这样可以尽量避免Android为我们放大图片所导致的OOM

对于values资源，当应用设备在当前dpi对应目录的demins.xml中没有找到目标条目时，采用“就近匹配”原则：

比如，当前为hdpi设备，并且只有以下几个目录，则values的寻找顺序为： 
hdpi->xhdpi->mdpi->values，即先向上级dpi目录查找，再向下级dpi目录查找，最后一路向下查找到values目录，如果values下都找不到，就只有找values-ldpi，当然，现在有这个目录的应用不多了。

# 附：dp与px，sp与px转换的代码

```java
public class DisplayUtil {   
        /** 
         * 将px值转换为dip或dp值，保证尺寸大小不变 
         *  
         */   
        public static int px2dip(Context context, float pxValue) {   
            final float scale = context.getResources().getDisplayMetrics().density;   
            return (int) (pxValue / scale + 0.5f);   
        }   
         
        /** 
         * 将dip或dp值转换为px值，保证尺寸大小不变 
         *  
         */   
        public static int dip2px(Context context, float dipValue) {   
            final float scale = context.getResources().getDisplayMetrics().density;   
            return (int) (dipValue * scale + 0.5f);   
        }   
         
        /** 
         * 将px值转换为sp值，保证文字大小不变 
         *  
         */   
        public static int px2sp(Context context, float pxValue) {   
            final float fontScale = context.getResources().getDisplayMetrics().scaledDensity;   
            return (int) (pxValue / fontScale + 0.5f);   
        }   
         
        /** 
         * 将sp值转换为px值，保证文字大小不变 
         *  
         */   
        public static int sp2px(Context context, float spValue) {   
            final float fontScale = context.getResources().getDisplayMetrics().scaledDensity;   
            return (int) (spValue * fontScale + 0.5f);   
        }   
    }  
```

除此以外，我们还可以使用 
```java
android.util.TypedValue#applyDimension(int unit, float value, DisplayMetrics metrics)
```
来进行dp、sp与px间的转换