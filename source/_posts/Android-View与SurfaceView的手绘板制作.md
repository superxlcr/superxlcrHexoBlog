---
title: Android View与SurfaceView的手绘板制作
tags: [android,view,应用]
categories: [android]
date: 2016-03-21 21:53:59
description: 自定义VIew实现手绘板、SurfaceView实现手绘板
---
最近学习了如何使用View与SurfaceView制作简单的手绘板，在此做个小结。

# 自定义VIew实现手绘板

首先是使用View来实现手绘板：
```java
public class NormalDrawBoardView extends View {  
  
    // 点击的坐标  
    private float lastX = 0, lastY = 0;  
    private Path path;  
    private Paint paint;  
    // 使用内存中的图片作为缓冲区  
    private Bitmap cacheBitmap;  
    // 缓冲区上的Canvas对象  
    private Canvas cacheCanvas;  
  
    public NormalDrawBoardView(Context context, int width, int height) {  
        super(context);  
        // 创建确定大小的bitmap  
        cacheBitmap = Bitmap.createBitmap(width, height, Bitmap.Config.ARGB_8888);  
        // 初始化缓存画布，把bitmap内容画到缓存画布上  
        cacheCanvas = new Canvas();  
        cacheCanvas.setBitmap(cacheBitmap);  
        path = new Path();  
        // 初始化画笔  
        // 防抖动  
        paint = new Paint(Paint.DITHER_FLAG);  
        paint.setColor(Color.BLACK);  
        paint.setStyle(Paint.Style.STROKE);  
        paint.setStrokeWidth(1);  
        // 反锯齿  
        paint.setAntiAlias(true);  
        paint.setDither(true);  
    }  
  
    /** 
     * 设置画笔的颜色 
     * @param color，颜色的字符串 
     */  
    public void setPaintColor(String color) {  
        switch (color) {  
        case "red" :  
            paint.setColor(Color.RED);  
            break;  
        case "green" :  
            paint.setColor(Color.GREEN);  
            break;  
        case "blue" :  
            paint.setColor(Color.BLUE);  
            break;  
        case "yellow" :  
            paint.setColor(Color.YELLOW);  
            break;  
        default:  
            paint.setColor(Color.BLACK);  
            break;  
        }  
    }  
  
    /** 
     * 设置画笔粗细 
     * @param width 
     */  
    public void setPaintWidth(int width) {  
        paint.setStrokeWidth(width);  
    }  
  
    public void clearCanvas() {  
        // 清除path轨迹  
        path.reset();  
        path.moveTo(lastX, lastY);  
        // 清除cacheCanvas图像  
        Paint paint = new Paint();  
        paint.setXfermode(new PorterDuffXfermode(PorterDuff.Mode.CLEAR));  
        cacheCanvas.drawPaint(paint);  
        invalidate();  
//        paint.setXfermode(new PorterDuffXfermode(PorterDuff.Mode.SRC));  
    }  
  
    @Override  
    public boolean onTouchEvent(MotionEvent event) {  
        float x = event.getX(), y = event.getY();  
        switch (event.getAction()) {  
        case MotionEvent.ACTION_DOWN: // 记录点击坐标，把当前点定义为线段的前一个点  
            lastX = x;  
            lastY = y;  
            path.moveTo(x, y);  
            break;  
        case MotionEvent.ACTION_MOVE: // 绘制线段  
            path.quadTo(lastX, lastY, x, y);  
            lastX = x;  
            lastY = y;  
            break;  
        case MotionEvent.ACTION_UP: // 把线段绘制到画布上  
            cacheCanvas.drawPath(path, paint);  
            path.reset();  
            break;  
        }  
        invalidate();  
        return true;  
    }  
  
    @Override  
    protected void onDraw(Canvas canvas) {  
        Paint bmpPaint = new Paint();  
        // 绘制之前画的轨迹  
        canvas.drawBitmap(cacheBitmap, 0, 0, bmpPaint);  
        // 绘制正在画的轨迹  
        canvas.drawPath(path, paint);  
    }  
}  
```

在上面的代码中我们首先初始化了以下变量：
- path：用于记录用户手指滑动的路径类
- paint：画笔类
- cacheBitmap：用于存储用户画的手绘的图片缓存
- cacheCanvas：用于描绘用户画的手绘的画布缓存
- lastX，lastY：记录上一个点击的点的坐标

然后我们重写了onTouchEvent方法来处理用户的手指点击事件：
- Motion.Action.DOWN：代表手指按下的事件，此时我们记录下手指点击坐标，并把path的起点设置为该坐标
- Motion.Action.MOVE：代表手指拖动事件，此时我们根据获取的坐标与前一个点的坐标画出一条线段，并更新记录的坐标
- Motion.Action.UP：代表手指抬起事件，此时我们把path记录的路径绘制到缓存中，并重置path

在每次触发onTouchEvent方法的时候我们都在最后调用invalidate方法触发我们的View调用onDraw进行重绘。
我们在onDraw方法中先把bitmap缓存的手绘记录绘制到画布上，再把当前的路径也绘制到画布上。
此时我们自定义View实现的手绘板就大功告成啦：
![View自定义手绘板效果图](1.jpg)

# SurfaceView实现手绘板

接下来我们来谈谈使用SurfaceView实现手绘板：
```java
public class SurfaceViewDrawBoardView extends SurfaceView  
        implements SurfaceHolder.Callback {  
  
    private final String TAG = "SurfaceViewDrawBoard";  
  
    // 绘制背景的线程  
    private MyThread myThread;  
    // 缓存用的bitmap和canvas  
    private Bitmap cacheBitmap;  
    private Canvas cacheCanvas;  
    // 画笔和路径  
    private Paint paint;  
    private Path path;  
    // 上一个点的坐标  
    private float lastX, lastY;  
  
    public SurfaceViewDrawBoardView(Context context, int width, int height) {  
        super(context);  
        // 设置bitmap和canvas  
        cacheBitmap = Bitmap.createBitmap(width, height, Bitmap.Config.ARGB_8888);  
        cacheCanvas = new Canvas();  
        cacheCanvas.setBitmap(cacheBitmap);  
        // 设置画笔  
        paint = new Paint(Paint.DITHER_FLAG);  
        paint.setStyle(Paint.Style.STROKE);  
        paint.setStrokeWidth(1);  
        paint.setColor(Color.BLACK);  
        paint.setAntiAlias(true);  
        paint.setDither(true);  
        // 初始化路径  
        path = new Path();  
        // 设置SurfaceHolder的回调函数  
        getHolder().addCallback(this);  
        // 初始化绘画线程  
        myThread = new MyThread(getHolder());  
        Log.v(TAG, Thread.currentThread().getName());  
    }  
  
    /** 
     * 设置画笔的颜色 
     * 
     * @param color，颜色的字符串 
     */  
    public void setPaintColor(String color) {  
        switch (color) {  
        case "red":  
            paint.setColor(Color.RED);  
            break;  
        case "green":  
            paint.setColor(Color.GREEN);  
            break;  
        case "blue":  
            paint.setColor(Color.BLUE);  
            break;  
        case "yellow":  
            paint.setColor(Color.YELLOW);  
            break;  
        default:  
            paint.setColor(Color.BLACK);  
            break;  
        }  
    }  
  
    /** 
     * 设置画笔粗细 
     * 
     * @param width 
     */  
    public void setPaintWidth(int width) {  
        paint.setStrokeWidth(width);  
    }  
  
    public void clearCanvas() {  
        // 清除path轨迹  
        path.reset();  
        path.moveTo(lastX, lastY);  
        // 清除cacheCanvas图像  
        Paint paint = new Paint();  
        paint.setXfermode(new PorterDuffXfermode(PorterDuff.Mode.CLEAR));  
        cacheCanvas.drawPaint(paint);  
        //        paint.setXfermode(new PorterDuffXfermode(PorterDuff.Mode.SRC));  
    }  
  
    @Override  
    public boolean onTouchEvent(MotionEvent event) {  
        float x = event.getX(), y = event.getY();  
        switch (event.getAction()) {  
        case MotionEvent.ACTION_DOWN: // 记录点击坐标，把当前点定义为线段的前一个点  
            lastX = x;  
            lastY = y;  
            path.moveTo(x, y);  
            break;  
        case MotionEvent.ACTION_MOVE: // 绘制线段  
            path.quadTo(lastX, lastY, x, y);  
            lastX = x;  
            lastY = y;  
            break;  
        case MotionEvent.ACTION_UP: // 把线段绘制到画布上  
            cacheCanvas.drawPath(path, paint);  
            path.reset();  
            break;  
        }  
        return true;  
    }  
  
    @Override  
    public void surfaceCreated(final SurfaceHolder holder) {  
        Log.v(TAG, "surfaceCreated");  
        myThread.isRun = true;  
        myThread.start();  
    }  
  
    @Override  
    public void surfaceChanged(SurfaceHolder holder, int format, int width,  
            int height) {  
        Log.v(TAG, "surfaceChanged");  
    }  
  
    @Override  
    public void surfaceDestroyed(SurfaceHolder holder) {  
        Log.v(TAG, "surfaceDestoryed");  
        myThread.isRun = false;  
    }  
  
    class MyThread extends Thread {  
        private SurfaceHolder holder;  
        public boolean isRun = false;  
        private int red = 0, green = 0, blue = 0;  
        private int colorValue = 0;  
        private float hsbValue[];  
  
        MyThread(SurfaceHolder holder) {  
            this.holder = holder;  
            hsbValue = new float[]{0, 1, 1};  
        }  
  
        @Override  
        public void run() {  
            while (isRun) {  
                Log.v(TAG, Thread.currentThread().getName());  
                Canvas canvas = holder.lockCanvas();  
                // 背景色渐变  
                hsbValue[0] = hsbValue[0] + 1 <= 360 ? hsbValue[0] + 1 : 0;  
                if (canvas != null) {  
                    // 绘制背景色  
                    canvas.drawColor(Color.HSVToColor(hsbValue));  
                    Paint bmpPaint = new Paint();  
                    // 绘制之前画的轨迹  
                    canvas.drawBitmap(cacheBitmap, 0, 0, bmpPaint);  
                    // 绘制正在画的轨迹  
                    canvas.drawPath(path, paint);  
                    holder.unlockCanvasAndPost(canvas);  
                }  
                try {  
                    // 休眠20ms  
                    TimeUnit.MILLISECONDS.sleep(20);  
                } catch (Exception e) {  
                    e.printStackTrace();  
                }  
            }  
        }  
    }  
}  
```

surfaceView允许我们在新开的线程中更新UI界面，我们首先实现了SurfaceHolder.Callback接口，该接口有以下几个重要方法：
- surfaceCreated：在Surface创建的时候调用，我们在该方法中开启了绘制UI的子线程
- surfaceChanged：在Surface大小发生改变时调用
- surfaceDestroyed：在Surface销毁时调用，我们在该方法中停止了绘制UI的子线程

实现了SurfaceHolder.Callback接口后，我们在初始化时使用getHolder方法可以获取SurfaceHolder，然后调用其addCallback方法加入实现的接口即可。
代码有大部分都与自定义View类似，此处我们重点讲讲子线程执行的工作。
由于我们使用的是SurfaceView，因此重写其onDraw方法并不能在界面上绘制出图像，正确的方法是调用SurfaceHolder的lockCanvas方法获取画布（有可能为null），然后绘制完成后调用unlockCanvasAndPost把画布给更新到屏幕上，所以我们就在子线程中每过20ms就调用一次以上方法来刷新我们的界面，此处本人还加入了背景变色功能：利用颜色的HSV属性，色调从0～360变化，亮度和对比度恒定为1。
最终效果如下图：
![SurfaceView自定义手绘板效果图](2.jpg)

实际上的手绘效果感觉没有自定义View来的好，有时候感觉会有卡顿现象出现，本人分析的原因为子线程绘制界面的间隔时间不够短的缘故。

经过本次自定义View和SurfaceView手绘板的实战，本人有如下体会：
1. View适合实现被动更新画面的。比如棋类，这种用view就好了。因为画面的更新是依赖于 onTouch 来更新，可以直接使用 invalidate。 因为这种情况下，这一次Touch和下一次的Touch需要的时间比较长些，不会产生影响。 
2. 而主动更新的画面应该使用SurfaceView。比如背景在一直变色。因为这需要一个单独的thread不停的重绘背景的状态，而如果使用View通过postInvalidated来实现的话，并不能保证固定频率刷新界面。所以view不合适实现这种固定频率主动更新界面的做法，用surfaceView来控制更为合适。 