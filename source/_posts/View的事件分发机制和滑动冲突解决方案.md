---
title: View的事件分发机制和滑动冲突解决方案
tags: []
categories: []
date: 2019-11-14 15:48:07
description: （转载）View的事件分发机制、滑动冲突、滑动冲突的处理方法、后记
---

本文转载自：https://www.jianshu.com/p/057832528bdd

这篇文章会先讲Android中View的事件分发机制，然后再介绍Android滑动冲突的形成原因并给出解决方案。因水平有限，讲的不会太过深入，只希望各位看了之后对事件分发机制的流程有个大概的概念，并且以后能自己解决有关滑动冲突的问题，用语浅薄，文笔生疏，见谅。

# View的事件分发机制

View的事件分发机制说白了就是点击事件的传递，也就是一个Down事件,若干个Move事件，一个Up事件构成的事件序列的传递。
当你手指按了屏幕，点击事件就会遵循Activity-&gt;Window-&gt;View这一顺序传递。
这一传递过程有三个重要的方法，分别是：

boolean dispatchTouchEcent(MotionEvent ev),

boolean onInterceptTouchEvent(MotionEvent event),

boolean onTouchEvent(MotionEvent event)
先一个一个简单介绍下：

## dispatchTouchEcent：

只要事件传递到了当前View,那么dispatchTouchEcent方法就一定会被调用。返回结果表示是否消耗当前事件。

## onInterceptTouchEvent：

在dispatchTouchEcent方法内部调用此方法，用来判断是否拦截某个事件。如果当前View拦截了某个事件，那么在这同一个事件序列中，此方法不会再次被调用。返回结果表示是否拦截当前事件。

## onTouchEvent：

在dispatchTouchEcent方法内调用此方法，用来处理事件。返回结果表示是否处理当前事件，如果不处理，那么在同一个事件序列里面，当前View无法再收到后续的事件。
上面的解释听起来比较抽象，我们可以用一段伪代码来表示上面三个方法的关系：

```java
public boolean dispatchTouchEvent(MotionEvent ev){
    boolean consum = false;
    if(onInterceptTouchEvent(ev)){
        consum = onTouchEvent(ev);
    }else{
        consum = child.dispatchTouchEvent(ev);
    }
    
    return consum;
}

```

上面代码很好的解释了三个方法之间的关系，我们也可以从代码中大致摸索到事件传递的顺序规则：当点击事件传递到根ViewGroup里，会执行dispatchTouchEvent，在其内部会先调用onInterceptTouchEvent询问是否拦截事件，若拦截，则执行onTouchEvent方法处理这个事件；若不拦截，则执行子元素的dispatchTouchEvent，进入向下分发的传递，直到事件被处理。
在处理一个事件的时候，是有优先级的，如果设置了OnTouchListener,会先执行其内部的onTouch方法，这时若onTouch方法返回true，那么表示事件被处理了，不会向下传递了；如果返回了false，那么事件会继续传递给onTouchEvent方法处理,在onTouchEvent方法中如果当前设置了OnClickListener,那么就会调用其onClick方法。所以其优先级为：OnTouchListen&gt;onTouchEvent&gt;OnClickListen。
这里有一种情况，如果一个View的onTouchEvent返回了false,那么它父容器的onTouchEvent方法将会被调用。我们写个例子来试一下：

```java
@Override
public boolean onTouchEvent(MotionEvent event) {
    Toast.makeText(mContext, "Button", Toast.LENGTH_SHORT).show();
    return false;
}

```

先自定义一个Button,重写其onTouchEvent方法返回false。在自定义一个MyTouchView作为父布局。效果如下：




![pic1](1.gif)



大家可以自己试试。
既然如此，在开头我们说过事件的传递顺序是Activity-&gt;Window-&gt;View,所以如果所有的元素都返回了false,那么最后事件就会再次传递到Activity里，由Activity的onTouchEvent方法来处理。
《Android开发艺术探索》这本书里总结了11条关于事件传递的结论：
1. 同一个事件序列是指手机接触屏幕那一刻起，到离开屏幕那一刻结束，有一个down事件，若干个move事件，一个up事件构成。
2. 某个View一旦决定拦截事件，那么这个事件序列之后的事件都会由它来处理，并且不会再调用onInterceptTouchEvent。
3. 正常情况下，一个事件序列只能被一个View拦截并消耗。这个原因可以参考第2条，因为一旦拦截了某个事件，那么这个事件序列里的其他事件都会交给这个View来处理，所以同一事件序列中的事件不能分别由两个View同时处理，但是我们可以通过特殊手段做到，比如一个View将本该自己处理的事件通过onTouchEvent强行传递给其他View处理。
4. 一个View如果开始处理事件，如果它不处理down事件（onTouchEvent里面返回了false）,那么这个事件序列的其他事件就不会交给它来继续处理了，而是会交给它的父元素去处理。
5. 如果一个View处理了down事件，却没有处理其他事件，那么这些事件不会交给父元素处理，并且这个View还能继续受到后续的事件。而这些未处理的事件，最终会交给Activity来处理。
6. ViewGroup的onInterceptToucheEvent默认返回false,也就是默认不拦截事件。
7. View没有InterceptTouchEvent方法，如果有事件传过来，就会直接调用onTouchEvent方法。
8. View的onTouchEvent方法默认都会消耗事件，也就是默认返回true,除非他是不可点击的（longClickable和clickable同时为false）。
9. View的enable属性不会影响onTouchEvent的默认返回值。就算一个View是不可见的，只要他是可点击的（clickable或者longClickable有一个为true）,它的onTouchEvent默认返回值也是true。
10. onClick方法会执行的前提是当前View是可点击的，并且它收到了down和up事件。
11. 事件传递过程是由外向内的，也就是事件会先传给父元素在向下传递给子元素。但是子元素可以通过requestDisallowInterceptTouchEvent来干预父元素的分发过程，但是down事件除外（因为down事件方法里，会清除所有的标志位）。

# 滑动冲突

介绍完了事件分发机制的基本流程，我们来看看滑动冲突。滑动冲突的基本形式分为两种，其他复杂的滑动冲突都可以拆成这两种基本形式：
1. 外部滑动方向与内部方向不一致。
2. 外部方向与内部方向一致。
先来看第一种，比如你用ViewPaper和Fragment搭配，而Fragment里往往是一个竖直滑动的ListView,这种情况是就会产生滑动冲突,但是由于ViewPaper本身已经处理好了滑动冲突，所以我们无需考虑，不过若是换成ScrollView，我们就得自己处理滑动冲突了。图示如下：




![pic2](2.jpg)



再看看第二种，这种情况下，因为内部和外部滑动方向一致，系统会分不清你要滑动哪个部分，所以会要么只有一层能滑动，要么两层一起滑动得很卡顿。图示如下:




![pic3](3.jpg)



对于这两种情况，我们处理的方法也很简单，并且都有相应的套路。

1. 第一种的冲突主要是一个横向一个竖向的，所以我们只要判断滑动方向是竖向还是横向的，再让对应的View滑动即可。判断的方法有很多，比如竖直距离与横向距离的大小比较；滑动路径与水平形成的夹角等等。
2. 对于这种情况，比较特殊，我们没有通用的规则，得根据业务逻辑来得出相应的处理规则。举个最常见的例子，ListView下拉刷新，需要ListView自身滑动，但是当滑动到头部时需要ListView和Header一起滑动，也就是整个父容器的滑动。如果不处理好滑动冲突，就会出现各种意想不到情况。

# 滑动冲突的处理方法

滑动冲突的拦截方法有两种：

一种是让事件都经过父容器的拦截处理，如果父容器需要则拦截，如果不需要则不拦截，成为外部拦截法，其伪代码如下:

```java
public boolean onInterceptTouchEvent(MotionEvent event) {
    boolean intercepted = false;
    int x = (int)event.getX();
    int y = (int)event.getY();
    switch (event.getAction()) {
        case MotionEvent.ACTION_DOWN: {
            intercepted = false;
           break;
       }
       case MotionEvent.ACTION_MOVE: {
           if (满足父容器的拦截要求) {
                intercepted = true;
           } else {
                intercepted = false;
           }
           break;
       }
       case MotionEvent.ACTION_UP: {
           intercepted = false;
           break;
       }
       default:
           break;
       }
            mLastXIntercept = x;
            mLastYIntercept = y;
            return intercepted;
}

```

在这里，首先down事件父容器必须返回false ，因为若是返回true，也就是拦截了down事件，那么后续的move和up事件就都会传递给父容器，子元素就没有机会处理事件了。其次是up事件也返回了false，一是因为up事件对父容器没什么意义，其次是因为若事件是子元素处理的，却没有收到up事件会让子元素的onClick事件无法触发。
另一种是父容器不拦截任何事件，将所有事件传递给子元素，如果子元素需要则消耗掉，如果不需要则通过requestDisallowInterceptTouchEvent方法交给父容器处理，称为内部拦截法，使用起来稍显麻烦。伪代码如下：

首先我们需要重写子元素的dispatchTouchEvent方法：

```java
  @Override
 public boolean dispatchTouchEvent(MotionEvent event) {
     int x = (int) event.getX();
     int y = (int) event.getY();

     switch (event.getAction()) {
     case MotionEvent.ACTION_DOWN: {
         parent.requestDisallowInterceptTouchEvent(true);
         break;
     }
     case MotionEvent.ACTION_MOVE: {
         int deltaX = x - mLastX;
         int deltaY = y - mLastY;
         if (父容器需要此类点击事件) {
             parent.requestDisallowInterceptTouchEvent(false);
         }
         break;
     }
     case MotionEvent.ACTION_UP: {
         break;
     }
     default:
         break;
     }

     mLastX = x;
     mLastY = y;
     return super.dispatchTouchEvent(event);
 }  

```

然后修改父容器的onInterceptTouchEvent方法：

```java
 @Override
 public boolean onInterceptTouchEvent(MotionEvent event) {

     int action = event.getAction();
     if (action == MotionEvent.ACTION_DOWN) {
         return false;
     } else {
         return true;
     }
 }  

```

这里父容器也不能拦截down事件。
看代码看不出所以然，我们通过实例来看看滑动冲突是怎么样的。

## 外部滑动方向与内部方向不一致

我们先模拟第一种场景，内外滑动方向不一致，我们先自定义一个父控件，让其可以左右滑动，类似于ViewPaper：




![pic4](4.gif)



然后将里面换成三个ListView:




![pic5](5.gif)


 

可以看到左右滑动失效了，说明确实冲突了。那么我们就来解决一下，首先我们要明白滑动规则是什么，这个例子中如果我们竖直滑动就让ListView消耗事件，水平滑动就让我们自定义的父容器滑动。知道了这个我们只需要将其替换到之前伪代码里的拦截条件里即可。
先用外部拦截法：

```java
    @Override
    public boolean onInterceptTouchEvent(MotionEvent event) {
    boolean intercepted = false;
    int x = (int) event.getX();
    int y = (int) event.getY();

    switch (event.getAction()) {
        case MotionEvent.ACTION_DOWN: {
            Log.d(TAG, "onInterceptTouchEvent: ACTION_DOWN");
            intercepted = false;
            break;
        }
        case MotionEvent.ACTION_MOVE: {
            Log.d(TAG, "onInterceptTouchEvent: ACTION_MOVE");
            int deltaX = x - mLastXIntercept;
            int deltaY = y - mLastYIntercept;
            if (Math.abs(deltaX) > Math.abs(deltaY)) {
                intercepted = true;
            } else {
                intercepted = false;
            }
            break;
        }
        case MotionEvent.ACTION_UP: {
            intercepted = false;
            break;
        }
        default:
            break;
    }

    Log.d(TAG, "intercepted=" + intercepted);
    mLastX = x;
    mLastY = y;
    mLastXIntercept = x;
    mLastYIntercept = y;

    return intercepted;
}  

```

这里我们判断横向滑动的距离与竖直滑动距离的长短。若是竖直滑动的长，则判断为竖直滑动，那么就是ListView的滑动，就将intercepted置为false，让父容器不拦截，交由子元素ListView处理。若是横向，则intercepted置为true，交由父容器处理。
效果如下：




![pic6](6.gif)



接下来看看内部拦截法：

先自定义一个MyListView继承ListView，重写其dispatchTouchEvent方法：

```java
 @Override
 public boolean dispatchTouchEvent(MotionEvent event) {
     int x = (int) event.getX();
     int y = (int) event.getY();

     switch (event.getAction()) {
     case MotionEvent.ACTION_DOWN: {
         mHorizontalScrollViewEx.requestDisallowInterceptTouchEvent(true);
         break;
     }
     case MotionEvent.ACTION_MOVE: {
         int deltaX = x - mLastX;
         int deltaY = y - mLastY;
         Log.d(TAG, "dx:" + deltaX + " dy:" + deltaY);
         if (Math.abs(deltaX) > Math.abs(deltaY)) {
             mHorizontalScrollViewEx.requestDisallowInterceptTouchEvent(false);
         }
         break;
     }
     case MotionEvent.ACTION_UP: {
         break;
     }
     default:
         break;
     }

     mLastX = x;
     mLastY = y;
     return super.dispatchTouchEvent(event);
 }  

```

再重写外部父容器的oninterceptTouchEvent方法:

```java
  @Override
  public boolean onInterceptTouchEvent(MotionEvent event) {
      int x = (int) event.getX();
      int y = (int) event.getY();
      int action = event.getAction();
      if (action == MotionEvent.ACTION_DOWN) {
          mLastX = x;
          mLastY = y;
          return false;
      } else {
          return true;
      }
  }  

```

效果和外部拦截法一样。

## 外部方向与内部方向一致

接下来看看同方向的滑动冲突,这里我们用一个竖直的ScrollView嵌套一个ListView做例子。首先看看没有解决滑动冲突的时候是咋样的：




![pic7](7.gif)



我们看到只要ScrollView可以滑动，内部的ListView是不能滑动的。那我们现在来解决这个问题，同向滑动冲突和与不同向滑动冲突不一样，得根据实际的需求来确定拦截的规则。这里我们的需求是当ListView滑到顶部了，并且继续向下滑就让ScrollView拦截掉；当ListView滑到底部了，并且继续向下滑，就让ScrollView拦截掉，其余时候都交给ListView自身处理事件。

首先用外部拦截法，我们需要重写ScrollView的onInterceptTouchEvent方法，代码如下:

```java
@Override
public boolean onInterceptTouchEvent(MotionEvent event) {
    boolean intercepted = false;
    int y = (int) event.getY();

    switch (event.getAction()) {

        case MotionEvent.ACTION_DOWN: {
            nowY = y;
            intercepted = super.onInterceptTouchEvent(event);
            break;
        }
        case MotionEvent.ACTION_MOVE: {
            if(mListView.getFirstVisiblePosition()==0
                    && y>nowY){
                intercepted = true;
                break;
            }
            else if(mListView.getLastVisiblePosition()==mListView.getCount()-1
                    && y<nowY){
                intercepted = true;
                break;
            }
            intercepted = false;
            break;
        }
        case MotionEvent.ACTION_UP: {
            intercepted = false;
            break;
        }
        default:
            break;
    }

    return intercepted;
} 

```

这里我们看到Down事件里我们并没有返回false而是返回了super.onInterceptTouchEvent(event),这是因为ScrollView在Down方法时需要初始化一些参数如果我们直接返回false,会导致滑动出现问题。并且前面说过ViewGroup

的onInterceptTouchEvent方法是默认返回false的，所以我们这里直接返回super方法即可。

处理了滑动冲突后效果如下：




![pic8](8.gif)



接下来看看内部拦截法：

先重写ScrollView的onInterceptTouchEvent方法，让其拦截除了Down事件以外的其他方法：

```java
@Override
    public boolean onInterceptTouchEvent(MotionEvent event) {
        int x = (int) event.getX();
        int y = (int) event.getY();
        int action = event.getAction();
        if (action == MotionEvent.ACTION_DOWN) {
            mLastX = x;
            mLastY = y;
            return super.onInterceptTouchEvent(event);
        } else {
            return true;
        }
    }

```

在重写ListView的dispatchTouchEvent方法，规则已经说明过了：

```java
 @Override
public boolean dispatchTouchEvent(MotionEvent event) {
    int y = (int) event.getY();

    switch (event.getAction()) {
        case MotionEvent.ACTION_DOWN: {
            nowY = y;
            mHorizontalScrollViewEx2.requestDisallowInterceptTouchEvent(true);
            break;
        }
        case MotionEvent.ACTION_MOVE: {
            if(this.getFirstVisiblePosition()==0
                    && y>nowY){
                mHorizontalScrollViewEx2.requestDisallowInterceptTouchEvent(false);
                break;
            }
            else if(this.getLastVisiblePosition()==this.getCount()-1
                    && y<nowY){
                mHorizontalScrollViewEx2.requestDisallowInterceptTouchEvent(false);
                break;
            }
            mHorizontalScrollViewEx2.requestDisallowInterceptTouchEvent(true);
            break;
        }
        case MotionEvent.ACTION_UP: {
            break;
        }
        default:
            break;
    }
    
    return super.dispatchTouchEvent(event);
}

```

最终效果和外部拦截法一样。
好了，这篇文章到此结束，希望各位看了能对事件分发机制有个大致的了解，并且遇到了滑动冲突的问题能够迎刃而解。谢谢忍受我的文章。

# 后记

2月17号更新：感谢 @ShadowXv所指出的问题，我已改正，改正如下：

将listView是否滑动到顶部或者底部的判断改为

```java
public boolean isBottom(final ListView listView) {
        boolean result=false;
        if (listView.getLastVisiblePosition() == (listView.getCount() - 1)) {
            final View bottomChildView = listView.getChildAt(listView.getLastVisiblePosition() - listView.getFirstVisiblePosition());
            result= (listView.getHeight()>=bottomChildView.getBottom());
        };
        return  result;
    }

    public boolean isTop(final ListView listView) {
        boolean result=false;
        if(listView.getFirstVisiblePosition()==0){
            final View topChildView = listView.getChildAt(0);
            result=topChildView.getTop()==0;
        }
        return result ;
    }

```

那么onInterceptTouchEvent方法也该改为如下所示：

```java
@Override
    public boolean onInterceptTouchEvent(MotionEvent event) {
        boolean intercepted = false;
        int y = (int) event.getY();

        switch (event.getAction()) {

            case MotionEvent.ACTION_DOWN: {
                nowY = y;
                intercepted = super.onInterceptTouchEvent(event);
                break;
            }
            case MotionEvent.ACTION_MOVE: {
                if(isTop(mListView)
                        && y>nowY){
                    intercepted = true;
                    break;
                }
                else if(isBottom(mListView)
                        && y<nowY){
                    intercepted = true;
                    break;
                }
                intercepted = false;
                break;
            }
            case MotionEvent.ACTION_UP: {
                intercepted = false;
                break;
            }
            default:
                break;
        }

        return intercepted;
    }

```

内部拦截法也是将判断条件换一下就可以了，这里就不贴代码了。。。
最后再一次感谢指出问题！