---
title: Android音视频学习笔记（二）
tags: [android,音视频]
categories: [android]
date: 2019-05-21 19:16:37
description: Camera采集视频YUV数据，输出H264格式
---

上篇博客传送门：[Android音视频学习笔记（一）](/2019/05/18/Android音视频学习笔记（一）/)

# Camera采集视频YUV数据，输出H264格式

## Camera视频预览

Camera是官方提供的相机相关的api，官方文档如下：
https://developer.android.com/reference/android/hardware/Camera
如果是api 21以上的版本，可以使用Camera2，官方文档如下：
https://developer.android.com/reference/android/hardware/camera2/package-summary.html

xml布局文件较为简单，这里不再贴出
CameraActivity相关代码如下：
```java
public class CameraActivity extends AppCompatActivity implements SurfaceHolder.Callback {

    private static final String TAG = "CameraActivity";

    private static final String DIR = Environment.getExternalStorageDirectory() + File.separator + "cameraTest" + File.separator;
    private static final String H264_FILE = DIR + "camera.h264";

    private BlockingQueue<byte[]> queue = new LinkedBlockingQueue<>();
    private H264Encoder h264Encoder;
    private boolean work = false;

    private Button btn;
    private Camera camera;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_camera);

        btn = (Button) findViewById(R.id.btn);
        btn.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                work = !work;
                if (work) {
                    btn.setText("stop");
                    h264Encoder.start();
                } else {
                    btn.setText("start");
                    h264Encoder.stop();
                }
            }
        });
        SurfaceView surfaceView = (SurfaceView) findViewById(R.id.surface_view);
        surfaceView.getHolder().addCallback(this);

        new File(DIR).mkdirs();

        camera = Camera.open();
        camera.setDisplayOrientation(90);
        Camera.Parameters parameters = camera.getParameters();
        Camera.Size size = parameters.getPreferredPreviewSizeForVideo();
        Log.i(TAG, "video preferred size width " + size.width + " height " + size.height);
        parameters.setPreviewSize(size.width, size.height);
        parameters.setPreviewFormat(ImageFormat.NV21);
        parameters.setFocusMode(Camera.Parameters.FOCUS_MODE_CONTINUOUS_VIDEO);
        camera.setParameters(parameters);
        camera.setPreviewCallback(new Camera.PreviewCallback() {
            @Override
            public void onPreviewFrame(byte[] data, Camera camera) {
                if (data != null && work) {
                    Log.i(TAG, "data size " + data.length);
                    queue.add(data);
                }
            }
        });

        h264Encoder = new H264Encoder(queue, H264_FILE, size.width, size.height, 30, null);
    }

    @Override
    public void surfaceCreated(SurfaceHolder holder) {
        try {
            camera.setPreviewDisplay(holder);
            camera.startPreview();
        } catch (IOException e) {
            Log.e(TAG, Log.getStackTraceString(e));
        }
    }

    @Override
    public void surfaceChanged(SurfaceHolder holder, int format, int width, int height) {

    }

    @Override
    public void surfaceDestroyed(SurfaceHolder holder) {
        camera.release();
    }
}
```

在这个Activity中，我们使用了SurfaceView来显示Camera提供的预览数据
有兴趣的童鞋也可以使用TextureView来代替实现相应的功能

SurfaceView官方文档如下：
https://developer.android.com/reference/android/view/SurfaceView
TextureView官方文档如下：
https://developer.android.com/reference/android/view/TextureView

二者的区别可以参考下面的博客：
https://www.cnblogs.com/wytiger/p/5693569.html
https://www.jianshu.com/p/b9a1e66e95ea

简单来说，一般而言，如果我们不需要对视频容器的View进行一些动画或者平移翻转等变化的话，可以优先考虑使用SurfaceView（消耗更低内存，减少绘制延迟）

有几个地方值得我们稍微注意一下：
1. Camera返回的数据可能是旋转过后的，一般我们需要使用Camera#setDisplayOrientation旋转90度摆正
2. Camera预览时可能因为没有对焦而导致视频模糊，我们可以使用Camera#setFocusMode，并传入Camera#Parameters#FOCUS_MODE_CONTINUOUS_VIDEO获得一个自动对焦的效果

## 采集视频YUV数据

## 输出H264格式