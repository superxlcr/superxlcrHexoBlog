---
title: Android音视频学习笔记（三）
tags: [android,音视频]
categories: [android]
date: 2019-11-27 15:16:23
description: 录制界面、音频录制、视频录制、音视频打包混合
---

上篇博客传送门：[Android音视频学习笔记（二）](/2019/05/21/Android音视频学习笔记（二）/)

经过了前面的一系列联系，这篇博客我们来总结一下如何实现一个完整的音视频录制功能

# 录制界面

录制界面由一个预览的窗口以及控制按钮组成，相关的布局文件如下：

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/main_layout"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:gravity="center"
    android:keepScreenOn="true"
    android:orientation="vertical">

    <SurfaceView
        android:id="@+id/surface_view"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1" />

    <Button
        android:id="@+id/btn"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:gravity="center"
        android:text="start" />

</LinearLayout>
```

界面代码如下所示：

```java
public class VideoActivity extends AppCompatActivity implements SurfaceHolder.Callback, View.OnClickListener {

    private static final String TAG = "VideoActivity";

    private static final String DIR = Environment.getExternalStorageDirectory() + File.separator + "videoTest" + File.separator;
    private static final String PCM_FILE = DIR + "videoTest.pcm";
    private static final String AAC_FILE = DIR + "videoTest.aac";
    private static final String H264_FILE = DIR + "videoTest.h264";
    private static final String MP4_FILE = DIR + "videoTest.mp4";

    private Camera camera;
    private PcmRecorder pcmRecorder;
    private AacEncoder aacEncoder;
    private H264Encoder h264Encoder;
    private MediaMuxerWrapper mediaMuxerWrapper;

    private boolean working = false;
    private BlockingQueue<byte[]> audioQueue = new LinkedBlockingQueue<>();
    private BlockingQueue<byte[]> videoQueue = new LinkedBlockingQueue<>();
    private BlockingQueue<AacFrame> audioEncodeQueue = new LinkedBlockingQueue<>();
    private BlockingQueue<H264Frame> videoEncodeQueue = new LinkedBlockingQueue<>();
    private Handler handler = new Handler();

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_video);

        findViewById(R.id.btn).setOnClickListener(this);

        SurfaceView surfaceView = (SurfaceView) findViewById(R.id.surface_view);
        surfaceView.getHolder().addCallback(this);

        checkDir();

        pcmRecorder = new PcmRecorder(44100, 1, 16, PCM_FILE, audioQueue);
        aacEncoder = new AacEncoder(audioQueue, AAC_FILE, pcmRecorder.getSampleRate(),
                pcmRecorder.getChannel(), pcmRecorder.getBitWidth(), audioEncodeQueue);
        aacEncoder.setCallback(new AacEncoder.Callback() {
            @Override
            public void onEncodeFinish() {
                mediaMuxerWrapper.stopAudio();
                toast("aac encode finish !");
            }

            @Override
            public void onFormatChange(MediaFormat format) {
                mediaMuxerWrapper.startMuxAudio(format);
            }
        });

        camera = Camera.open();
        camera.setDisplayOrientation(90);
        Camera.Parameters parameters = camera.getParameters();
        List<int[]> frameRateList = parameters.getSupportedPreviewFpsRange();
        int frameRateRange[] = frameRateList.get(frameRateList.size() / 2);
        parameters.setPreviewFpsRange(frameRateRange[0], frameRateRange[1]);
        List<Camera.Size> sizeList = parameters.getSupportedVideoSizes();
        Camera.Size size = sizeList.get(sizeList.size() / 2);
        parameters.setPreviewSize(size.width, size.height);
        parameters.setPreviewFormat(ImageFormat.NV21);
        camera.setParameters(parameters);
        camera.setPreviewCallback(new Camera.PreviewCallback() {
            @Override
            public void onPreviewFrame(byte[] data, Camera camera) {
                if (working && data != null) {
                    Log.i(TAG, "data size " + data.length);
                    videoQueue.add(data);
                }
            }
        });

        int frameRate = (frameRateRange[0] + frameRateRange[1]) / 2 / 1000;
        h264Encoder = new H264Encoder(videoQueue, H264_FILE, size.width, size.height, frameRate,
                videoEncodeQueue);
        h264Encoder.setCallback(new H264Encoder.Callback() {
            @Override
            public void onEncodeFinish() {
                mediaMuxerWrapper.stopVideo();
                toast("h264 encode finish !");
            }

            @Override
            public void onFormatChange(MediaFormat format) {
                mediaMuxerWrapper.startMuxVideo(format);
            }
        });

        mediaMuxerWrapper = new MediaMuxerWrapper(videoEncodeQueue, audioEncodeQueue, MP4_FILE);
        mediaMuxerWrapper.setCallback(new MediaMuxerWrapper.Callback() {
            @Override
            public void onMuxFinish() {
                toast("mux finish !");
            }
        });
    }

    @Override
    public void surfaceCreated(SurfaceHolder holder) {
        try {
            camera.setPreviewDisplay(holder);
            camera.startPreview();
        } catch (IOException e) {
            Log.e(TAG, "error : " + e.toString());
        }
    }

    @Override
    public void surfaceChanged(SurfaceHolder holder, int format, int width, int height) {

    }

    @Override
    public void surfaceDestroyed(SurfaceHolder holder) {

    }

    @Override
    public void onClick(View v) {
        if (!working) {
            ((Button) v).setText("stop");
            start();
        } else {
            ((Button) v).setText("finish");
            stop();
            v.setClickable(false);
        }
    }

    private void start() {
        if (working) {
            return;
        }
        working = true;
        pcmRecorder.start();
        aacEncoder.start();
        h264Encoder.start();
        toast("start recording !");
    }

    private void stop() {
        if (!working) {
            return;
        }
        working = false;
        pcmRecorder.stop();
        aacEncoder.stop();
        h264Encoder.stop();
        toast("finish recording !");
    }

    private void checkDir() {
        File dir = new File(DIR);
        if (!dir.exists()) {
            dir.mkdirs();
        }
    }

    private void toast(final String str) {
        handler.post(new Runnable() {
            @Override
            public void run() {
                Toast.makeText(VideoActivity.this, str, Toast.LENGTH_SHORT).show();
            }
        });
    }

}
```

界面的Activity主要是把各个模块的逻辑汇总协同工作的地方，接下来我们来单独分析每个部分

# 音频录制

音频录制部分主要有：

- PCM录制器
- AAC格式编码器

## PCM录制器

```java
public class PcmRecorder {

    private static final String TAG = "PcmRecorder";

    private int sampleRate;
    private int channel;
    private int bitWidth;

    private AudioRecord audioRecord;
    private int bufferSize;
    private byte buffer[];

    private FileOutputStream fos = null;
    private BlockingQueue<byte[]> queue = null;

    private Handler workHandler;
    private boolean work;

    PcmRecorder(int sampleRate, int channel, int bitWidth, String fileOutputPath, BlockingQueue<byte[]> queue) {
        this(sampleRate, channel, bitWidth);
        File outputFile = new File(fileOutputPath);
        try {
            fos = new FileOutputStream(outputFile);
        } catch (IOException e) {
            throw new IllegalStateException("create file output stream fail with " + e.toString());
        }
        if (queue == null) {
            throw new IllegalArgumentException("queue can not be null");
        }
        this.queue = queue;
    }

    private PcmRecorder(int sampleRate, int channel, int bitWidth) {
        this.sampleRate = sampleRate;
        this.channel = channel == 2 ? channel : 1;
        this.bitWidth = bitWidth == 8 ? bitWidth : 16;

        int channelConfig = channel == 2 ? AudioFormat.CHANNEL_IN_STEREO : AudioFormat.CHANNEL_IN_MONO;
        int audioFormat = bitWidth == 8 ? AudioFormat.ENCODING_PCM_8BIT : AudioFormat.ENCODING_PCM_16BIT;
        bufferSize = AudioRecord.getMinBufferSize(sampleRate, channelConfig, audioFormat);
        buffer = new byte[bufferSize];
        Log.i(TAG,
                "sampleRate : " + sampleRate + " channel : " + channel + " bitWidth : " + bitWidth + " bufferSize : " + bufferSize);
        audioRecord = new AudioRecord(MediaRecorder.AudioSource.MIC, sampleRate, channelConfig,
                audioFormat, bufferSize);
        Log.i(TAG, "state : " + audioRecord.getState());

        HandlerThread workThread = new HandlerThread("recordThread");
        workThread.start();
        workHandler = new RecordHandler(workThread.getLooper());
    }

    public int getSampleRate() {
        return sampleRate;
    }

    public int getChannel() {
        return channel;
    }

    public int getBitWidth() {
        return bitWidth;
    }

    synchronized void start() {
        if (work) {
            return;
        }
        work = true;
        audioRecord.startRecording();
        workHandler.sendEmptyMessage(0);
    }

    synchronized void stop() {
        if (!work) {
            return;
        }
        work = false;
        audioRecord.stop();
    }

    private class RecordHandler extends Handler {

        RecordHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
            Log.i(TAG, "work handler start");
            while (work) {
                int result = audioRecord.read(buffer, 0, bufferSize);
                if (result == AudioRecord.ERROR_INVALID_OPERATION || result == AudioRecord.ERROR_BAD_VALUE) {
                    Log.e(TAG, "record read fail ! just continue !");
                    continue;
                }
                Log.i(TAG, "read result : " + result);

                if (fos != null) {
                    try {
                        fos.write(buffer, 0, result);
                        fos.flush();
                    } catch (IOException e) {
                        Log.e(TAG, Log.getStackTraceString(e));
                    }
                }

                if (queue != null) {
                    boolean offerResult = queue.offer(buffer);
                    Log.i(TAG, "offerResult " + offerResult + " queue size " + queue.size());
                }
            }
            if (fos != null) {
                try {
                    fos.close();
                } catch (IOException e) {
                    Log.e(TAG, Log.getStackTraceString(e));
                }
            }
            Log.i(TAG, "work handler finish");
        }
    }

}
```

PCM录制使用的是官方的AudioRecord

官方API文档：https://developer.android.com/reference/android/media/AudioRecord

相关要注意的事项已经在学习笔记（一）中提及，这里不再赘述

## AAC格式编码器

录制出视频的PCM数据后，我们需要把其编码为AAC格式，相关代码如下：

```java
public class AacEncoder {

    private static final String TAG = "AacEncoder";

    private static final int AAC_PROFILE = MediaCodecInfo.CodecProfileLevel.AACObjectLC;
    private static final int ADTS_HEADER_SIZE = 7;

    private MediaCodec mediaCodec;

    private BlockingQueue<byte[]> queue;
    private BlockingQueue<AacFrame> outputQueue;
    private FileOutputStream fos;

    private long totalSize;
    private int bufferSize;
    private ByteBuffer[] inputByteBuffers;
    private ByteBuffer[] outputByteBuffers;

    private boolean work = false;
    private Thread workThread;

    private int sampleRate;
    private int channel;
    private int bitWidth;

    private Callback callback;

    // 有些手机压缩后质量奇差，硬件不好
    AacEncoder(BlockingQueue<byte[]> inputQueue, String outputFile, int sampleRate, int channel, int bitWidth, BlockingQueue<AacFrame> outputQueue) {
        if (inputQueue == null) {
            throw new IllegalArgumentException("inputQueue can not be null !");
        }
        queue = inputQueue;
        this.outputQueue = outputQueue;
        String mime;
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
            mime = MediaFormat.MIMETYPE_AUDIO_AAC;
        } else {
            mime = "audio/mp4a-latm";
        }
        try {
            fos = new FileOutputStream(new File(outputFile));
            mediaCodec = MediaCodec.createEncoderByType(mime);
        } catch (IOException e) {
            throw new IllegalArgumentException(e.toString());
        }
        MediaFormat mediaFormat = new MediaFormat();
        mediaFormat.setString(MediaFormat.KEY_MIME, mime);
        mediaFormat.setInteger(MediaFormat.KEY_AAC_PROFILE, AAC_PROFILE);
        mediaFormat.setInteger(MediaFormat.KEY_BIT_RATE, sampleRate * channel * bitWidth);
        mediaFormat.setInteger(MediaFormat.KEY_CHANNEL_COUNT, channel);
        // TODO lmr 帧率对不上视频 ？
        this.sampleRate = sampleRate;
        this.channel = channel == 2 ? 2 : 1;
        this.bitWidth = bitWidth;
        int channelConfig = channel == 2 ? AudioFormat.CHANNEL_IN_STEREO : AudioFormat.CHANNEL_IN_MONO;
        int audioFormat = bitWidth == 8 ? AudioFormat.ENCODING_PCM_8BIT : AudioFormat.ENCODING_PCM_16BIT;
        bufferSize = AudioRecord.getMinBufferSize(sampleRate, channelConfig, audioFormat);
        mediaFormat.setInteger(MediaFormat.KEY_MAX_INPUT_SIZE, bufferSize);
        mediaFormat.setInteger(MediaFormat.KEY_SAMPLE_RATE, sampleRate);
        mediaCodec.configure(mediaFormat, null, null, MediaCodec.CONFIGURE_FLAG_ENCODE);

        totalSize = 0;
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
                Log.i(TAG, "begin aac encode");
                while (work || !queue.isEmpty()) {
                    try {
                        Log.i(TAG, "before take data");
                        byte[] data = queue.take();
                        Log.i(TAG, "wait for input index");
                        int inputIndex = mediaCodec.dequeueInputBuffer(-1);
                        if (inputIndex >= 0) {
                            Log.i(TAG,
                                    "input index : " + inputIndex + " data size : " + data.length);
                            ByteBuffer inputByteBuffer = inputByteBuffers[inputIndex];
                            inputByteBuffer.clear();
                            inputByteBuffer.put(data);
                            inputByteBuffer.limit(data.length);
                            long time = getPresentationTimeUs();
                            totalSize += data.length;
                            Log.i(TAG, "input data total size " + totalSize + " time " + time);
                            mediaCodec.queueInputBuffer(inputIndex, 0, data.length, time, 0);
                        }
                        MediaCodec.BufferInfo bufferInfo = new MediaCodec.BufferInfo();
                        int outputIndex = mediaCodec.dequeueOutputBuffer(bufferInfo, 0);
                        Log.i(TAG,
                                "output index : " + outputIndex + " data size : " + bufferInfo.size + " offset : " + bufferInfo.offset);
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
                                int packetSize = bufferInfo.size + ADTS_HEADER_SIZE;

                                ByteBuffer outputByteBuffer = outputByteBuffers[outputIndex];
                                outputByteBuffer.position(bufferInfo.offset);
                                outputByteBuffer.limit(bufferInfo.offset + bufferInfo.size);

                                byte[] dataBytes = new byte[bufferInfo.size];
                                outputByteBuffer.get(dataBytes);
                                Log.i(TAG, "output buffer info time " + bufferInfo.presentationTimeUs);
                                if (outputQueue != null) {
                                    // 这个时间戳要保持一直增加，不然的话mux的时候会卡死
                                    // 取 bufferInfo.presentationTimeUs 会出现有增有减的情况
                                    outputQueue.add(
                                            new AacFrame(dataBytes, getPresentationTimeUs()));
                                }

                                byte[] packetBytes = new byte[packetSize];
                                addADTSHeader(packetBytes, packetSize);
                                System.arraycopy(dataBytes, 0, packetBytes, 7, bufferInfo.size);

                                try {
                                    fos.write(packetBytes);
                                    fos.flush();
                                    Log.i(TAG, "write bytes finish");
                                } catch (IOException e) {
                                    Log.e(TAG, Log.getStackTraceString(e));
                                }

                                mediaCodec.releaseOutputBuffer(outputIndex, false);
                            }
                            outputIndex = mediaCodec.dequeueOutputBuffer(bufferInfo, 0);
                            Log.i(TAG,
                                    "output index : " + outputIndex + " data size : " + bufferInfo.size + " offset : " + bufferInfo.offset);
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
                Log.i(TAG, "finish aac encode");
                if (callback != null) {
                    callback.onEncodeFinish();
                }
            }
        });
        workThread.start();
    }

    private void addADTSHeader(byte[] packetBytes, int totalSize) {
        if (packetBytes.length < 7) {
            throw new IllegalStateException("the adts header bytes size is less than 7 !");
        }
        for (int i = 0; i < 7; i++) {
            packetBytes[i] = 0;
        }
        // adts header contains 2 part
        // first part : adts fixed header

        // syncword 0xFFF size:12
        packetBytes[0] = (byte) 0xFF;
        packetBytes[1] |= 0xF << 4;
        // ID MPEG Version: 0 for MPEG-4，1 for MPEG-2 size:1
        packetBytes[1] |= 1 << 3;
        // Layer 00 size:2
        packetBytes[1] |= 0 << 1;
        // protection_absent Warning, set to 1 if there is no CRC and 0 if there is CRC size:1 (we use 7 header size , means is 1)
        packetBytes[1] |= 1;
        // profile aac 级别 size:2
        packetBytes[2] |= (AAC_PROFILE - 1) << 6;
        // sampling_frequency_index 采样率下标 size:4 44100hz means 0x4
        packetBytes[2] |= 4 << 2;
        // private_bit 私有位，没用 size 1
        packetBytes[2] |= 0 << 1;
        // channel_configuration 声道数 size 3
        packetBytes[2] |= channel >>> 2;
        packetBytes[3] |= channel << 6;
        // original_copy：编码时设置为0，解码时忽略 size 1
        packetBytes[3] |= 0 << 5;
        // home：编码时设置为0，解码时忽略 size 1
        packetBytes[3] |= 0 << 4;

        // second part : adts variable header

        // copyrighted_id_bit：编码时设置为0，解码时忽略 size 1
        packetBytes[3] |= 0 << 3;
        // copyrighted_id_start：编码时设置为0，解码时忽略 size 1
        packetBytes[3] |= 0 << 2;
        // aac_frame_length：ADTS帧长度包括ADTS长度和AAC声音数据长度的和 size 13
        packetBytes[3] |= totalSize >>> 11;
        packetBytes[4] |= totalSize >>> 3;
        packetBytes[5] |= totalSize << 5;
        // adts_buffer_fullness：固定为0x7FF。表示是码率可变的码流 size 11
        packetBytes[5] |= 0x7FF >>> 6;
        packetBytes[6] |= 0x7FF << 2;
        // number_of_raw_data_blocks_in_frame：表示当前帧有number_of_raw_data_blocks_in_frame + 1 个原始帧(一个AAC原始帧包含一段时间内1024个采样及相关数据) size 2
        packetBytes[6] |= 0;
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

    private long getPresentationTimeUs() {
        return (long) (totalSize * 8.0 / sampleRate / channel / bitWidth * 1000 * 1000);
    }

    public interface Callback {

        void onEncodeFinish();

        void onFormatChange(MediaFormat format);
    }

}
```

这里使用的是系统提供的MediaCodec接口进行硬件编码，官方文档如下：
https://developer.android.com/reference/android/media/MediaCodec
这里我们使用的MIME为audio/mp4a-latm，同时我们用了一个BlockingQueue阻塞队列来循环编码，每当一段PCM数据准备好之后，就读取并塞到MediaCodec中进行编码

由于MediaCodec编码出来的AAC数据只是原始的数据流，因此我们需要为其补上元数据（即adts头部信息）
有关adts头部信息相关的内容可以参考下面的博客：
https://blog.csdn.net/simongyley/article/details/8754157
具体的头部信息填写可参考上面代码中的方法addADTSHeader

有一个需要注意的点是，由于MediaCodec输出的编码时间有点问题（具体原因未明，主要表现是输出的时间有增有减），因此这里使用的是getPresentationTimeUs方法来计算帧时间，防止后面使用MediaMuxer混合出现卡死的情况

值得注意的是，据博主了解与尝试，由于Android碎片化的缘故，音频使用硬件编码质量并不十分稳定，而音频软件编码的耗时与稳定性都相对不错，因此建议大家学习一下使用如FFMPEG的工具来软件编码

编码后，输出的AAC帧如下所示：

```java
public class AacFrame {

    public byte[] data;
    public long presentationTimeUs;

    public AacFrame(byte[] data, long presentationTimeUs) {
        this.data = data;
        this.presentationTimeUs = presentationTimeUs;
    }
}
```

# 视频录制

视频录制部分主要是把摄像头回调回来的YUV数据经过处理后，使用硬件编码成H264格式输出
相关代码如下：
```java
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

这里大部分内容与学习笔记（二）中所提及的一样，主要注意的就是YUV格式的转换以及画面内容旋转的问题，在此不再赘述
还有当我们要把音频与视频打包合成MP4的时候，一些MediaCodec.BufferInfo的flag信息以及视频时间戳也是必不可少的

编码后的H264帧如下所示：

```java
public class H264Frame {

    public byte[] data;
    public int flags;
    public long presentationTimeUs;

    public H264Frame(byte[] data, int flags, long presentationTimeUs) {
        this.data = data;
        this.flags = flags;
        this.presentationTimeUs = presentationTimeUs;
    }
}
```

# 音视频打包混合

在分别编码出aac音频以及h264视频数据后，我们需要把他们打包混合成MP4文件
相关代码如下：

```java
public class MediaMuxerWrapper {

    private static final String TAG = "MediaMuxerWrapper";

    private BlockingQueue<H264Frame> videoQueue;
    private BlockingQueue<AacFrame> audioQueue;
    private MediaMuxer mediaMuxer;

    private CyclicBarrier startCyclicBarrier;
    private CyclicBarrier stopCyclicBarrier;
    private int videoIndex;
    private boolean videoWork = false;
    private Thread videoWorkThread;
    private int audioIndex;
    private boolean audioWork = false;
    private Thread audioWorkThread;

    private Callback callback;

    public MediaMuxerWrapper(BlockingQueue<H264Frame> videoQueue, BlockingQueue<AacFrame> audioQueue, String outputFile) {
        if (videoQueue == null) {
            throw new IllegalArgumentException("videoQueue can not be null !");
        }
        this.videoQueue = videoQueue;
        if (audioQueue == null) {
            throw new IllegalArgumentException("audioQueue can not be null !");
        }
        this.audioQueue = audioQueue;
        try {
            mediaMuxer = new MediaMuxer(outputFile, MediaMuxer.OutputFormat.MUXER_OUTPUT_MPEG_4);
        } catch (IOException e) {
            throw new IllegalArgumentException(e.toString());
        }

        startCyclicBarrier = new CyclicBarrier(2, new Runnable() {
            @Override
            public void run() {
                mediaMuxer.start();
                Log.i(TAG, "mux both start");
            }
        });
        stopCyclicBarrier = new CyclicBarrier(2, new Runnable() {
            @Override
            public void run() {
                mediaMuxer.stop();
                mediaMuxer.release();
                Log.i(TAG, "mux both finish");
                if (callback != null) {
                    callback.onMuxFinish();
                }
            }
        });
    }

    public void setCallback(Callback callback) {
        this.callback = callback;
    }

    public void startMuxVideo(MediaFormat mediaFormat) {
        if (videoWork) {
            return;
        }
        videoWork = true;
        videoIndex = mediaMuxer.addTrack(mediaFormat);
        videoWorkThread = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    startCyclicBarrier.await();
                    while (videoWork || !videoQueue.isEmpty()) {
                        H264Frame frame = videoQueue.poll(5, TimeUnit.SECONDS);
                        if (frame == null) {
                            Log.i(TAG, "video take data null !");
                            continue;
                        }
                        Log.i(TAG, "video take data, data left " + videoQueue.size());
                        ByteBuffer buffer = ByteBuffer.allocate(frame.data.length);
                        buffer.put(frame.data);
                        MediaCodec.BufferInfo info = new MediaCodec.BufferInfo();
                        info.offset = 0;
                        info.size = frame.data.length;
                        info.presentationTimeUs = frame.presentationTimeUs;
                        info.flags = frame.flags;
                        Log.i(TAG,
                                "video info flags " + info.flags + " time " + info.presentationTimeUs);
                        if ((info.flags & MediaCodec.BUFFER_FLAG_KEY_FRAME) != 0) {
                            Log.i(TAG, "video key frame !");
                        }
                        mediaMuxer.writeSampleData(videoIndex, buffer, info);
                        Log.i(TAG, "video write sample data finish");
                    }
                } catch (InterruptedException | BrokenBarrierException e) {
                    // ignore
                    Log.i(TAG, "mux video interrupted ! " + e.toString());
                }
                Log.i(TAG, "mux video finish, await");
                try {
                    stopCyclicBarrier.await();
                } catch (InterruptedException | BrokenBarrierException e) {
                    // ignore
                }
            }
        });
        videoWorkThread.start();
    }

    public void stopVideo() {
        if (!videoWork) {
            return;
        }
        videoWork = false;
    }

    public void startMuxAudio(MediaFormat mediaFormat) {
        if (audioWork) {
            return;
        }
        audioWork = true;
        audioIndex = mediaMuxer.addTrack(mediaFormat);
        audioWorkThread = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    startCyclicBarrier.await();
                    while (audioWork || !audioQueue.isEmpty()) {
                        AacFrame frame = audioQueue.poll(5, TimeUnit.SECONDS);
                        if (frame == null) {
                            Log.i(TAG, "audio take data null !");
                            continue;
                        }
                        Log.i(TAG, "audio take data, data left " + audioQueue.size());
                        ByteBuffer buffer = ByteBuffer.allocate(frame.data.length);
                        buffer.put(frame.data);
                        MediaCodec.BufferInfo info = new MediaCodec.BufferInfo();
                        info.offset = 0;
                        info.size = frame.data.length;
                        info.presentationTimeUs = frame.presentationTimeUs;
                        info.flags = 0;
                        Log.i(TAG,
                                "audio info flags " + info.flags + " time " + info.presentationTimeUs);
                        if ((info.flags & MediaCodec.BUFFER_FLAG_KEY_FRAME) != 0) {
                            Log.i(TAG, "audio key frame !");
                        }
                        mediaMuxer.writeSampleData(audioIndex, buffer, info);
                        Log.i(TAG, "audio write sample data finish");
                    }
                } catch (InterruptedException | BrokenBarrierException e) {
                    // ignore
                    Log.i(TAG, "mux audio interrupted !" + e.toString());
                }
                Log.i(TAG, "mux audio finish, await");
                try {
                    stopCyclicBarrier.await();
                } catch (InterruptedException | BrokenBarrierException e) {
                    // ignore
                }
            }
        });
        audioWorkThread.start();
    }

    public void stopAudio() {
        if (!audioWork) {
            return;
        }
        audioWork = false;
    }

    public interface Callback {

        void onMuxFinish();
    }

}
```

这里我们使用的是官方的MediaMuxer接口，官方文档如下：
https://developer.android.com/reference/android/media/MediaMuxer

这里我们通过两个BlockingQueue阻塞队列来持续提供数据，分别使用两个线程来添加音频帧与视频帧数据
由于音频与视频编码速度存在差异（实践下经常是音频编码比视频快很多），这里我们使用CyclicBarrier来控制混合流程的开始与结束