---
title: Android音视频学习笔记（二）
tags: [android,音视频]
categories: [android]
date: 2019-05-21 19:16:37
description: Camera视频预览、采集视频YUV数据、输出H264格式
---

上篇博客传送门：[Android音视频学习笔记（一）](/2019/05/18/Android音视频学习笔记（一）/)

# Camera视频预览

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

# 采集视频YUV数据

YUV（亮度、明度、对比度）是视频最原始的数据，跟音频的PCM类似
参考上面的代码，我们可以使用Camera#setPreviewCallback来设置回调，获取相机预览回调的YUV数据
值得注意的是，我们需要通过Camera#setPreviewFormat，把预览回调的数据格式设置为ImageFormat#NV21

NV21是YUV420-Semeplanar格式中的一种，相关的介绍可以参考下面的文章：
https://blog.csdn.net/jk198310/article/details/79084283

# 输出H264格式

获取到以NV21格式表示的YUV数据后，我们就可以把它们编码压缩成H264格式了
相关代码如下：
```java
/**
 * Created by superxlcr on 2019/4/7.
 */
public class H264Encoder {

    private static final String TAG = "H264Encoder";

    private MediaCodec mediaCodec;

    private BlockingQueue<byte[]> queue;
    private BlockingQueue<H264Frame> outputQueue;
    private FileOutputStream fos;

    private int frameRate;
    private int width;
    private int height;
    private int inputIndexCounter;
    private ByteBuffer[] inputByteBuffers;
    private ByteBuffer[] outputByteBuffers;
    private byte[] configBytes;
    private byte[] tempBuffer;

    private boolean work = false;
    private Thread workThread;

    private Callback callback;

    /**
     * 根据 stackOverFlow 上老哥的回答
     * https://stackoverflow.com/questions/36114808/android-setting-presentation-time-of-mediacodec
     * 其实 h264 只有流，并没有时间相关的信息，在某些硬件上可能播放会有问题（画面一闪而过，时间戳对不上）
     * 考虑封装成 mp4 处理这个问题
     */
    H264Encoder(BlockingQueue<byte[]> inputQueue, String outputFile, int width, int height, int frameRate, BlockingQueue<H264Frame> outputQueue) {
        if (inputQueue == null) {
            throw new IllegalArgumentException("inputQueue can not be null !");
        }
        queue = inputQueue;
        this.outputQueue = outputQueue;
        String mime;
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
            mime = MediaFormat.MIMETYPE_VIDEO_AVC;
        } else {
            mime = "video/avc";
        }
        try {
            fos = new FileOutputStream(new File(outputFile));
            mediaCodec = MediaCodec.createEncoderByType(mime);
        } catch (IOException e) {
            throw new IllegalArgumentException(e.toString());
        }

        // 我们需要把视频内容旋转90度，长宽对调
        MediaFormat mediaFormat = MediaFormat.createVideoFormat(mime, height, width);
        mediaFormat.setInteger(MediaFormat.KEY_COLOR_FORMAT,
                MediaCodecInfo.CodecCapabilities.COLOR_FormatYUV420SemiPlanar); // NV12
        mediaFormat.setInteger(MediaFormat.KEY_BIT_RATE, width * height * 5);
        mediaFormat.setInteger(MediaFormat.KEY_FRAME_RATE, frameRate);
        mediaFormat.setInteger(MediaFormat.KEY_I_FRAME_INTERVAL, 1);

        Bundle bundle = new Bundle();
        bundle.putInt(MediaFormat.KEY_I_FRAME_INTERVAL, 1);
        mediaCodec.setParameters(bundle);
        mediaCodec.configure(mediaFormat, null, null, MediaCodec.CONFIGURE_FLAG_ENCODE);

        this.frameRate = frameRate;
        this.width = width;
        this.height = height;

        tempBuffer = new byte[width * height * 3 / 2];

        Log.i(TAG, String.format("width is %d height is %d frameRate is %d tempBuffer size is %d",
                width, height, frameRate, tempBuffer.length));
    }

    public void setCallback(Callback callback) {
        this.callback = callback;
    }

    public synchronized void start() {
        if (work) {
            return;
        }
        work = true;
        mediaCodec.start();
        inputByteBuffers = mediaCodec.getInputBuffers();
        outputByteBuffers = mediaCodec.getOutputBuffers();
        workThread = new Thread(new Runnable() {
            @Override
            public void run() {
                inputIndexCounter = 0;
                while (work || !queue.isEmpty()) {
                    try {
                        Log.i(TAG, "try take data, left data " + queue.size());
                        byte[] data = queue.take();
                        NV21ToNV12(data, tempBuffer);
                        NV12Rotate90(tempBuffer, data);
                        Log.i(TAG, "take data size " + data.length);
                        int inputIndex = mediaCodec.dequeueInputBuffer(-1);
                        Log.i(TAG, "get input index " + inputIndex);
                        if (inputIndex >= 0) {
                            ByteBuffer inputBuffer = inputByteBuffers[inputIndex];
                            inputBuffer.clear();
                            inputBuffer.put(data);
                            inputBuffer.limit(data.length);
                            int flags = !work && queue
                                    .isEmpty() ? MediaCodec.BUFFER_FLAG_END_OF_STREAM : 0;
                            long time = getPresentationTimeUs(inputIndexCounter);
                            Log.i(TAG, "input indexCounter " + inputIndexCounter + " time " + time);
                            mediaCodec.queueInputBuffer(inputIndex, 0, data.length,
                                    time, flags);
                            inputIndexCounter++;
                        }

                        MediaCodec.BufferInfo bufferInfo = new MediaCodec.BufferInfo();
                        int outputIndex = mediaCodec.dequeueOutputBuffer(bufferInfo, 0);
                        Log.i(TAG, "get output index " + outputIndex);
                        while (outputIndex != MediaCodec.INFO_TRY_AGAIN_LATER) {
                            if (outputIndex == MediaCodec.INFO_OUTPUT_FORMAT_CHANGED) {
                                Log.i(TAG, "get output format");
                                if (callback != null) {
                                    callback.onFormatChange(mediaCodec.getOutputFormat());
                                }
                            } else if (outputIndex == MediaCodec.INFO_OUTPUT_BUFFERS_CHANGED) {
                                Log.i(TAG, "output buffers change !");
                                outputByteBuffers = mediaCodec.getOutputBuffers();
                            } else {
                                ByteBuffer outputBuffer = outputByteBuffers[outputIndex];
                                byte[] outData = new byte[bufferInfo.size];
                                outputBuffer.position(bufferInfo.offset);
                                outputBuffer.get(outData);
                                if ((bufferInfo.flags & MediaCodec.BUFFER_FLAG_CODEC_CONFIG) != 0) {
                                    Log.i(TAG, "flag codec config !");
                                    configBytes = outData;
                                } else if ((bufferInfo.flags & MediaCodec.BUFFER_FLAG_KEY_FRAME) != 0) {
                                    Log.i(TAG, "flag key frame");
                                    byte[] keyframe = new byte[bufferInfo.size + configBytes.length];
                                    System.arraycopy(configBytes, 0, keyframe, 0,
                                            configBytes.length);
                                    System.arraycopy(outData, 0, keyframe, configBytes.length,
                                            outData.length);
                                    if (outputQueue != null) {
                                        Log.i(TAG, "output time " + bufferInfo.presentationTimeUs);
                                        outputQueue
                                                .add(new H264Frame(keyframe, bufferInfo.flags,
                                                        bufferInfo.presentationTimeUs));
                                    }
                                    try {
                                        fos.write(keyframe);
                                        fos.flush();
                                    } catch (IOException e) {
                                        // ignore
                                    }
                                } else {
                                    Log.i(TAG, "flag nothing");
                                    if (outputQueue != null) {
                                        Log.i(TAG, "output time " + bufferInfo.presentationTimeUs);
                                        outputQueue
                                                .add(new H264Frame(outData, bufferInfo.flags,
                                                        bufferInfo.presentationTimeUs));
                                    }
                                    try {
                                        fos.write(outData);
                                        fos.flush();
                                    } catch (IOException e) {
                                        // ignore
                                    }
                                }
                                mediaCodec.releaseOutputBuffer(outputIndex, false);
                            }
                            outputIndex = mediaCodec.dequeueOutputBuffer(bufferInfo, 0);
                            Log.i(TAG, "get output index " + outputIndex);
                        }
                    } catch (InterruptedException e) {
                        // ignore
                    }
                }
                try {
                    fos.close();
                } catch (IOException e) {
                    Log.e(TAG, Log.getStackTraceString(e));
                }
                mediaCodec.stop();
                mediaCodec.release();
                Log.i(TAG, "finish h264 encode");
                if (callback != null) {
                    callback.onEncodeFinish();
                }
            }
        });
        workThread.start();
    }

    public synchronized void stop() {
        if (!work) {
            return;
        }
        work = false;
        if (workThread != null && workThread.isAlive()) {
            workThread.interrupt();
        }
    }

    private void NV21ToNV12(byte[] nv21, byte[] nv12) {
        if (nv21 == null || nv12 == null) return;
        int frameSize = width * height;
        int j;
        // Y
        System.arraycopy(nv21, 0, nv12, 0, frameSize);
        // U
        for (j = 0; j < frameSize / 2; j += 2) {
            nv12[frameSize + j] = nv21[frameSize + j + 1];
        }
        // V
        for (j = 0; j < frameSize / 2; j += 2) {
            nv12[frameSize + j + 1] = nv21[frameSize + j];
        }
    }

    private void NV12Rotate90(byte[] nv12, byte[] nv12rotate90) {
        if (nv12 == null || nv12rotate90 == null) return;
        int frameSize = width * height;
        // Y
        for (int i = 0; i < frameSize; i++) {
            int x = i % height;
            int y = i / height;
            int oldX = y;
            int oldY = height - x - 1;
            nv12rotate90[i] = nv12[oldX + oldY * width];
        }
        int halfHeight = height / 2;
        // U
        for (int j = 0; j < frameSize / 2; j += 2) {
            int x = j % height;
            int y = j / height;
            int oldX = y * 2;
            int oldY = halfHeight - 1 - x / 2;
            int rotateIndex = frameSize + j;
            int originIndex = frameSize + oldX + oldY * width;
            try {
                nv12rotate90[rotateIndex] = nv12[originIndex];
            } catch (IndexOutOfBoundsException e) {
                Log.e(TAG, String.format(
                        "error with j %d x %d y %d oldX %d oldY %d rotateIndex %d originIndex %d",
                        j, x, y, oldX, oldY, rotateIndex, originIndex));
                throw e;
            }
        }
        // V
        for (int j = 1; j < frameSize / 2; j += 2) {
            int x = j % height;
            int y = j / height;
            int oldX = y * 2 + 1;
            int oldY = halfHeight - 1 - (x - 1) / 2;
            nv12rotate90[frameSize + j] = nv12[frameSize + oldX + oldY * width];
        }
    }

    private long getPresentationTimeUs(int frameIndex) {
        return frameIndex * 1000 * 1000 / frameRate;
    }

    public interface Callback {

        void onEncodeFinish();

        void onFormatChange(MediaFormat format);
    }

}
```

这里我们通过MediaCodec创建硬件编码器，并传入一些关键的参数来进行H264编码
官方的MediaCodec文档如下：
https://developer.android.com/reference/android/media/MediaCodec

传入的关键参数如下：
1. 视频长宽：这个我们可以通过Camera#Parameters#getPreferredPreviewSizeForVideo来获取推荐的视频录制大小（**长宽自己随便填可能会导致编码失败**）
2. MediaFormat#KEY_COLOR_FORMAT：YUV格式，我们使用的是NV21，但这里只支持NV12，因此我们传入MediaCodecInfo#CodecCapabilities#COLOR_FormatYUV420SemiPlanar
3. MediaFormat#KEY_BIT_RATE：码率，一般使用的是 长 x 宽 x 一个系数（这里取的5），码率越高视频越大，也越清晰
4. MediaFormat#KEY_FRAME_RATE：帧率，这个我们可以通过Camera#Parameters#getSupportedPreviewFpsRange来获取，在这个例子中我们则是写死了30
5. MediaFormat#KEY_I_FRAME_INTERVAL：每秒关键帧数，例子填的是每秒1帧关键帧

值得注意的是，在Android 6.0中，通过MediaCodec#configure来设置关键帧在某些机型上会出现失效的情况，进而影响视频的编码
这时我们可以通过MediaCodec#setParameters来设置相应的关键帧参数，具体代码如下：
```java
Bundle bundle = new Bundle();
bundle.putInt(MediaFormat.KEY_I_FRAME_INTERVAL, 1);
mediaCodec.setParameters(bundle);
```

另外值得注意的有以下几个点：

1. MediaCodec硬件编码器仅支持NV12格式，因此我们需要把Camera回调的NV21格式转为NV12格式，主要是U分量与V分量的对调，相关代码可见H264Encoder#NV21ToNV12
2. 正如上面所说Camera预览时需要旋转90度摆正，而Camera回调回来的YUV数据是没有旋转的，因此在编码前我们需要旋转YUV数据，相关代码可见H264Encoder#NV12Rotate90
3. 不同的手机上硬件编码效率不一致，相差甚远，有的手机非常快（存放数据的阻塞队列几乎不留任何数据），而有的则会囤积200多条数据（估计这也是用FFMPEG进行软编码的原因？）