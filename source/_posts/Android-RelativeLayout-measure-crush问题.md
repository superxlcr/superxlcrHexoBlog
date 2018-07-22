---
title: Android RelativeLayout measure方法 crush问题
tags: [android,源码,view]
categories: [android]
date: 2018-07-22 15:51:12
description: 情景再现、源码分析、解决方案
---
最近在某些版本较低的手机上，遇到了RelativeLayout在执行measure方法时crush，在此写下博客记录一下

# 情景再现

在做需求的时候，本人inflate了一个RelativeLayout，并手动调用了其measure，打算测量其所需要的高宽，代码如下：
布局文件：
```xml
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent" android:layout_height="match_parent">
	
<!--其他布局....-->

</RelativeLayout>
```

java代码：
```java
View layout = LayoutInflater.from(this).inflate(R.layout.relative_layout, null);
layout.measure(View.MeasureSpec.makeMeasureSpec(0, View.MeasureSpec.UNSPECIFIED),
		View.MeasureSpec.makeMeasureSpec(0, View.MeasureSpec.UNSPECIFIED));
layout.getMeasuredWidth();
layout.getMeasuredHeight();
```

在大多数设备上，这段代码运行没有任何问题
然而，在api 19以下的手机运行，应用crush了，报错的堆栈信息如下：
![堆栈信息](1.png)

从堆栈信息我们可以推断出，这是一个RelativeLayout源码中onMeasure方法的空指针问题

# 源码分析

根据堆栈信息的指示，我们找到了api 19 以及 api 18 的RelativeLayout源码，并把他们的onMeasure方法进行比对：
![BeyongCompare对比图](2.png)

左边为api 18 的代码，右边为api 19 的代码
从比较关键的对比截图看出，代码中主要的区别是为LayoutParams增加了判空处理，因此我们可以大胆猜测，空指针是由于我们inflate出来的RelativeLayout没有设置LayoutParams导致的
api 19 以上的代码也有LayoutParams为空的问题，只不过这边添加了判空处理

这时我们需要看回LayoutInflater的源码
我们调用inflate方法时，传入的root为null，其最终调用的源码如下：
```java
public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
	synchronized (mConstructorArgs) {
		...

		if (TAG_MERGE.equals(name)) {
			if (root == null || !attachToRoot) {
				throw new InflateException("<merge /> can be used only with a valid "
						+ "ViewGroup root and attachToRoot=true");
			}

			rInflate(parser, root, inflaterContext, attrs, false);
		} else {
			// Temp is the root view that was found in the xml
			final View temp = createViewFromTag(root, name, inflaterContext, attrs);

			ViewGroup.LayoutParams params = null;

			if (root != null) {
				if (DEBUG) {
					System.out.println("Creating params from root: " +
							root);
				}
				// Create layout params that match root, if supplied
				params = root.generateLayoutParams(attrs);
				if (!attachToRoot) {
					// Set the layout params for temp if we are not
					// attaching. (If we are, we use addView, below)
					temp.setLayoutParams(params);
				}
			}

			if (DEBUG) {
				System.out.println("-----> start inflating children");
			}

			// Inflate all children under temp against its context.
			rInflateChildren(parser, temp, attrs, true);

			if (DEBUG) {
				System.out.println("-----> done inflating children");
			}

			// We are supposed to attach all the views we found (int temp)
			// to root. Do that now.
			if (root != null && attachToRoot) {
				root.addView(temp, params);
			}

			// Decide whether to return the root that was passed in or the
			// top view found in xml.
			if (root == null || !attachToRoot) {
				result = temp;
			}
		}

		...
	}
}
```

由于我们根布局的tag不为merge，因此代码会走入12行的else语句执行
在第14行，通过反射实例化了我们顶层的ViewGroup布局temp
在第18行，在传入的root参数不为null的情况下，会为我们的temp生成相应的LayoutParams
如果传入的attach参数为false，则在28行为temp设置LayoutParams，否则在第46行addView的时候设置

# 解决方案

在根据源码发现问题后，我们只需修改代码，在inflate的时候传入相应的root参数即可

Ps：其实在传入root参数为null的情况下，一般IDE都会有相应的警告提醒的，看来以后还是要多注意这种warning信息才行
