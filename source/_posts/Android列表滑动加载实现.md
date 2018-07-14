---
title: Android列表滑动加载实现
tags: [android,应用]
categories: [android]
date: 2017-08-03 20:52:26
description: onScrollStateChanged、onScroll
---
Android列表滑动加载主要依靠ListView的OnScrollListener实现，在此先介绍一下ListView的OnScrollListener接口：


```java
public interface OnScrollListener {
    public static int SCROLL_STATE_IDLE = 0;
    public static int SCROLL_STATE_TOUCH_SCROLL = 1;
    public static int SCROLL_STATE_FLING = 2;

    public void onScrollStateChanged(AbsListView view, int scrollState);
    public void onScroll(AbsListView view, int firstVisibleItem, int visibleItemCount,
				int totalItemCount);
}
```


可以看到，OnScrollListener接口有三个静态滚动状态的变量，及两个要实现的方法。

# onScrollStateChanged
滚动状态发生变化时，系统会回调这个方法。滚动状态会被赋值到scrollState，scrollState的值如下：



| scrollState值 | 含义 | 
| - | - |
| SCROLL_STATE_IDLE | 不滚动时的状态，通常会在滚动停止时监听到此状态 | 
| SCROLL_STATE_TOUCH_SCROLL | 正在滚动的状态 | 
| SCROLL_STATE_FLING | 用力快速滑动时可监听到此值 | 


# onScroll
滚动过程中会回调此方法。详细的参数含义：



| onScroll方法参数 | 含义 | 
| - | - |
| firstVisibleItem | 第一个可视的项，这里是整个item都可视的项。被挡住一点的item都不符合 | 
| visibleItemCount | 可视的项的个数 | 
| totalItemCount | 总item的个数 | 


实现滑动加载的例子如下：

```java
public class MainActivity extends AppCompatActivity {

    private ListView listView;
    private BaseAdapter adapter;
    private int counter;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        listView = (ListView)findViewById(R.id.list_view);
        final ArrayList<Integer> list = new ArrayList<>();
        counter = 0;
        int oldCounter = counter;
        for (counter = oldCounter; counter < oldCounter + 15; counter++) {
            list.add(counter);
        }
        adapter = new ArrayAdapter<Integer>(this, android.R.layout.simple_list_item_1, list);
        listView.setAdapter(adapter);
        listView.setOnScrollListener(new AbsListView.OnScrollListener() {
            @Override
            public void onScrollStateChanged(AbsListView view, int scrollState) {

            }

            @Override
            public void onScroll(AbsListView view, int firstVisibleItem, int visibleItemCount, int totalItemCount) {
                if (firstVisibleItem != 0) { // 不为0则表示有下拉动作
                    if ((firstVisibleItem + visibleItemCount) > totalItemCount - 2) { // 当前第一个完全可见的item再下拉一个页面长度，即变为倒数第二个时
                        // 在此加载数据
                        int oldCounter = counter;
                        for (counter = oldCounter; counter < oldCounter + 15; counter++) {
                            list.add(counter);
                        }
                        adapter.notifyDataSetChanged();
                    }
                }
            }
        });
    }
}
```


布局文件为一个listView，在此不贴出了。


效果如下：
![_pic1](1.gif)