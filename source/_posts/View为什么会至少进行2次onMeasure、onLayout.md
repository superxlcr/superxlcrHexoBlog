---
title: View为什么会至少进行2次onMeasure、onLayout
tags: [android,源码]
categories: [android]
date: 2018-12-11 20:03:29
description: （转载）前言、原因猜想、验证一、二、验证三、总结
---

本文转载自：https://www.jianshu.com/p/733c7e9fb284

# 前言

郭前辈的[ListView源码解析](https://link.jianshu.com?t=http://blog.csdn.net/guolin_blog/article/details/44996879)一文，曾提到View至少会进行2次onMeasure、onLayout，但限于篇幅，并未解释原因，好奇就尝试找了找原因。

# 原因猜想




![pic1](1.jpg)

害怕.jpg


由于不知道具体原因，只能结合已有的知识，先做出如下猜想：
1. View自身进行了2次onMeasure、onLayout
2. ViewGroup对Child进行了2次measure、layout
3. 我们知道View的绘制流程都始于ViewRootImpl的performTraversals方法，有理由怀疑performTranversals执行了2次。

**PS:**赶时间的朋友，可直接阅读验证三

# 验证一、二

按照上面的猜想，先进入View类查找总共就2处调用了onMeasure方法如下：

```java
 public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
 .........
  final boolean forceLayout = (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT;
 .........
  int cacheIndex = forceLayout ? -1 : mMeasureCache.indexOfKey(key);
  if (cacheIndex < 0 || sIgnoreMeasureCache) {
                // measure ourselves, this should set the measured dimension flag back
                onMeasure(widthMeasureSpec, heightMeasureSpec);
                mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
  }
.........
  }

```

第二处：

```java
public void layout(int l, int t, int r, int b) {
  if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
            onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);
            mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
        }
   ..........
}
 /**
     * Flag indicating that a call to measure() was skipped and should be done
     * instead when layout() is invoked.
     */
    static final int PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT = 0x8;

```

第二次onMeasure，但是注释说的很明确，只有当measure方法未被调用的时候，才会在layout里面执行一次onMeasure方法，正常的view树测量流程，每个view的measure方法确实的都被调用过，所以猜想一排除。
关于猜想二，以FrameLayout为例，确实是在onMeasure方法中对child进行了2次测量，但这是有条件限制的，需要FrameLayout的layout_width/height属性不能为match_parent或具体的值，且child的layout属性必须为match_parent，具有特殊性，实际上即使不满足以上条件依旧会进行2次测量，故排除猜想二。
PS：关于一、二的源码分析，可参考[View measure源码分析]()

# 验证三

看了看代码，发现会执行2次performTranversals，也就会执行2次测量。

ViewRootImpl#performTraversals()代码片段一

```
        //1.由于第一次执行newSurface必定为true，需要先创建Surface嘛
        //为true则会执行else语句，所以第一次执行并不会执行 performDraw方法，即View的onDraw方法不会得到调用
        //第二次执行则为false，并未创建新的Surface，第二次才会执行 performDraw方法
        if (!cancelDraw && !newSurface) {
            if (!skipDraw || mReportNextDraw) {
                if (mPendingTransitions != null && mPendingTransitions.size() > 0) {
                    for (int i = 0; i < mPendingTransitions.size(); ++i) {
                        mPendingTransitions.get(i).startChangingAnimations();
                    }
                    mPendingTransitions.clear();
                }

                performDraw();
            }
        } else {
            //2.viewVisibility是wm.add的那个View的属性，View的默认值都是可见的
            if (viewVisibility == View.VISIBLE) {
                // Try again
                //3.再执行一次 scheduleTraversals，也就是会再执行一次performTraversals
                scheduleTraversals();
            } else if (mPendingTransitions != null && mPendingTransitions.size() > 0) {
                for (int i = 0; i < mPendingTransitions.size(); ++i) {
                    mPendingTransitions.get(i).endChangingAnimations();
                }
                mPendingTransitions.clear();
            }
        }

```

断点SDK-23源码结果如下图所示：

**PS:**断点源码建议使用模拟器，真机一般都是修改过的，与SDK代码不一致。



![pic2](2.png)

第一次performTranversals




![pic3](3.png)

第二次performTranversals

既然确定了performTravelsals会执行2次，那么肯定会执行2次measure方法，但是执行2次measure方法就一定会执行2次onMeasure方法吗？

答案是否定，分析过View measure方法源码的都应知道measure方法做了2级测量优化：
1. 如果flag不为forceLayout或者与上次测量规格（MeasureSpec）相比未改变，那么将不会进行重新测量（执行onMeasure方法），直接使用上次的测量值；
2. 如果满足非强制测量的条件，即前后二次测量规格不一致，会先根据目前测量规格生成的key索引缓存数据，索引到就无需进行重新测量;如果targetSDK小于API 20则二级测量优化无效，依旧会重新测量，不会采用缓存测量值。
照理第二次测量应该会取测量的缓存值，并不会重新测量（调用onMeasure）的。然而实际上确重新测量了，那么极有可能就是第二次performMeasure传入的测量规格与第一次不同，因为在layout执行中已经将flag force_layout置为false了，代码如下：

```java
  public void layout(int l, int t, int r, int b) {
        .........
        //mPrivateFlags第16位设置为0,0表示不强制layout
        //PFLAG_FORCE_LAYOUT = 0x00001000
        mPrivateFlags &= ~PFLAG_FORCE_LAYOUT;
        mPrivateFlags3 |= PFLAG3_IS_LAID_OUT;
    }

```

按照刚才的分析，前后二次的传入的测量规格应该不一致，然而事实是2次传入onMeasure()的测量规格一致，结果如下：



![pic4](4.png)

log信息


那么问题又来了，为什么会测量三次呢？首先声明的是，并不是因为FrameLayout的多次测量，此处的自定义View并不满足FrameLayout测量2次child的条件。经过断点跟踪SDK源码发现：

**第一次performTranversals会执行2次performMeasure：**

先执行measureHierarchy方法中的performMeasure方法



![pic5](5.png)

第一次执行处




![pic6](6.png)

方法调用栈


接着执行后面的performMeasure，



![pic7](7.png)

第二次执行处




![pic8](8.png)

方法调用栈


**第二次performTranversals则是只执行measureHierarchy中的performMeasure方法**

这就能解释为什么前2次测量都执行了onMeasure方法，而未采用测量优化策略，因为前2次performMeasure并未经过performLayout，也即forceLayout的标志位一直为true，自然不会取缓存优化。理论上第三次测量经过第一次performTranversals中的performLayout，强制layout的flag应该为false，然而实际上却又变成了true，至于在哪儿恢复为true的，我在源码中并没有找到答案。

但是这在api24、25上却及其符合我们的推论，第三次强制layout的flag为false，即第二次performTranversals并不会导致View的onMeasure方法的调用，由于未调用onMeasure方法，也不会调用onLayout方法，即api 25只会执行2次onMeasure、一次onLayout、一次onDraw，如下图所示：



![pic9](9.png)

SDK-24




![pic10](10.png)

SDK-25


# 总结

**api25-24：**执行2次onMeasure、2次onLayout、1次onDraw，理论上执行三次测量，但由于测量优化策略，第三次不会执行onMeasure。

**api23-21：**执行3次onMeasure、2次onLayout、1次onDraw，forceLayout标志位，离奇被置为true，导致无测量优化。

**api19-16：**执行2次onMeasure、2次onLayout、1次onDraw，原因第一次performTranversals中只会执行measureHierarchy中的performMeasure，forceLayout标志位，离奇被置位true，导致无测量优化。

总之，造成这个现象的根本原因是performTranversal函数在View的测量流程中会执行2次。
