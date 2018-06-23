---
title: Android瀑布流的一些坑
tags: [android,view]
categories: [android]
date: 2018-06-23 15:11:51
description: 如何实现瀑布流、瀑布流分割线、瀑布流错位问题
---
最近实现了一个Android中瀑布流的需求，遇到了一些小坑，在此记录一下

# 如何实现瀑布流

想要实现瀑布流，我们可以使用supportv7包中的RecyclerView来处理
使用StaggeredGridLayoutManager，传入每行/每列item的数量以及布局方向即可：
```java
recyclerView = findViewById(R.id.recycler_view);
StaggeredGridLayoutManager layoutManager = new StaggeredGridLayoutManager(2, StaggeredGridLayoutManager.VERTICAL);
recyclerView.setLayoutManager(layoutManager);
```

# 瀑布流分割线

如果我们通过设置item的margin以及padding来实现分割线，会出现中间的分割线过粗的问题：
![分割线过粗示意图](1.png)

可以看到，中间的分割线大小是旁边分割线的两倍
这种情况，我们可以通过自定义RecyclerView.ItemDecoration来实现分割线的功能
通过获取getItemOffsets中view的LayoutParams（StaggeredGridLayoutManager中为**StaggeredGridLayoutManager.LayoutParams**），我们可以获取到具体item在瀑布流中布局的信息（位于某行/某列中的第几位），然后调整相应的绘制区域大小：
```java
    private class StaggeredGridItemDecoration extends RecyclerView.ItemDecoration {

        private int dividerWidth;
        private int maxSpan;

        /**
         * 瀑布流分割线
         *
         * @param dividerWidth 分割线大小
         * @param maxSpan      每行/每列最大元素个数
         */
        public StaggeredGridItemDecoration(int dividerWidth, int maxSpan) {
            this.dividerWidth = dividerWidth;
            this.maxSpan = maxSpan;
        }

        @Override
        public void getItemOffsets(Rect outRect, View view, RecyclerView parent, RecyclerView.State state) {
            if (view.getLayoutParams() instanceof StaggeredGridLayoutManager.LayoutParams) {
                StaggeredGridLayoutManager.LayoutParams layoutParams = (StaggeredGridLayoutManager.LayoutParams) view.getLayoutParams();
                int position = parent.getChildAdapterPosition(view);
                int spanIndex = layoutParams.getSpanIndex();
                int halfDividerWidth = dividerWidth / 2;
                outRect.left = halfDividerWidth;
                outRect.right = halfDividerWidth;
                outRect.top = position < maxSpan ? 0 : dividerWidth;
                if (spanIndex == 0) {
                    // left
                    outRect.left += halfDividerWidth;
                } else if (spanIndex == maxSpan - 1) {
                    outRect.right += halfDividerWidth;
                }
            }
        }
    }
```

值得注意的是，getItemOffsets中的outRect设置了之后其实达到了一个类似于padding的效果，会占用每个item的大小空间（如item设置的width为200，在设置了左右宽度为20分割线后，实际的item宽度变成了200 - 20 * 2 = 160）

# 瀑布流错位问题

某些情况下，当我们在瀑布流往下滑动一段距离后，再往回滑动时，会发现瀑布流的布局出现了错位的情况
刚开始的瀑布流布局：
![瀑布流布局示意图](2.png)

滑动一段距离，往回滑动后，布局出现了错位且顶部出现了空隙：
![瀑布流布局错位示意图](3.png)

一般情况下，在我们停止滑动后，StaggeredGridLayoutManager会自动通过动画效果修复这种空隙的情况
如果不希望StaggeredGridLayoutManager自动修复这种空隙的情况，我们可以通过setGapStrategy来处理：
```java
        StaggeredGridLayoutManager layoutManager = new StaggeredGridLayoutManager(2,
                StaggeredGridLayoutManager.VERTICAL);
        layoutManager.setGapStrategy(StaggeredGridLayoutManager.GAP_HANDLING_NONE);
```

StaggeredGridLayoutManager提供了两种处理空隙的方案，一种是处理，另一种是不处理：
```java
/**
     * Does not do anything to hide gaps.
     */
    public static final int GAP_HANDLING_NONE = 0;

    /**
     * When scroll state is changed to {@link RecyclerView#SCROLL_STATE_IDLE}, StaggeredGrid will
     * check if there are gaps in the because of full span items. If it finds, it will re-layout
     * and move items to correct positions with animations.
     * <p>
     * For example, if LayoutManager ends up with the following layout due to adapter changes:
     * <pre>
     * AAA
     * _BC
     * DDD
     * </pre>
     * <p>
     * It will animate to the following state:
     * <pre>
     * AAA
     * BC_
     * DDD
     * </pre>
     */
    public static final int GAP_HANDLING_MOVE_ITEMS_BETWEEN_SPANS = 2;
```

对于这种布局出现的空隙的问题，网上一般的解决方案是通过设置GAP_HANDLING_NONE，以及在滑动结束后强制刷新布局解决，代码如下：
```java
        layoutManager.setGapStrategy(StaggeredGridLayoutManager.GAP_HANDLING_NONE);
        recyclerView.addOnScrollListener(new RecyclerView.OnScrollListener() {
            @Override
            public void onScrollStateChanged(RecyclerView recyclerView, int newState) {
                // 滑动停止，刷新布局与分割线
                if (newState == RecyclerView.SCROLL_STATE_IDLE) {
                    if (recyclerView.getLayoutManager() instanceof StaggeredGridLayoutManager) {
                        ((StaggeredGridLayoutManager) recyclerView.getLayoutManager()).invalidateSpanAssignments();
                        recyclerView.invalidateItemDecorations();
                    }
                }
            }
        });
```

遗憾的是，这种方案会使得布局出现闪烁的问题
下面，我们通过分析瀑布流错位产生的原因来处理这个问题

## item的width以及height改动导致错位

stackOverFlow上对于这种情况的回答：
https://stackoverflow.com/questions/27636999/staggeredgridlayoutmanager-and-moving-items/27719126#27719126

这种情况错位主要的原因是：由于image加载等情况，在瀑布流布局之后修改了item的width以及height，导致布局出现了空隙
对于这种情况，我们通过在RecyclerView.Adapter#onBindViewHolder方法中事先设置好item的大小，避免后续进行修改即可

## notifyDataSetChanged导致错位问题

在本人进行测试中发现，在没有调用notifyDataSetChanged方法的情况下，瀑布流是一直不会出现错位问题的
但是在调用了notifyDataSetChanged方法后，有可能会导致瀑布流出现错位的情况（不一定百分百出现）
由于源码太长，就没有仔细去分析了，从结果来看，应该是StaggeredGridLayoutManager本来持有了之前的item的布局情况（关于Span的位置信息），因此一般情况下是不会出现瀑布流错位的问题的
但是当我们在调用了notifyDataSetChanged方法之后，由于item被标志为失效了，重新测量布局后可能被放置到了错误的位置，导致了瀑布流错位以及空隙的出现

综上所述，我们应该尽量避免调用**notifyDataSetChanged**方法，调用其他方法来代替，如添加item时调用**notifyItemRangeInserted**方法代替