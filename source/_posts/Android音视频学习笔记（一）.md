---
title: Android音视频学习笔记（一）
tags: [android,音视频]
categories: [android]
date: 2019-05-18 15:20:46
description: 学习思路、AudioRecord录制PCM，输出WAV格式，AudioTrack播放PCM
---

最近学习了一些关于Android音视频的知识，在此写博客记录一下

# 学习思路

关于Android音视频的学习思路参考的是这篇博客：https://www.cnblogs.com/renhui/p/7452572.html

初步学习主要分为四项任务：

1. AudioRecord录制PCM，输出WAV格式，AudioTrack播放PCM
2. Camera采集视频YUV数据，输出H264格式
3. MediaExtractor和MediaMuxer解析和封装mp4文件
4. MediaCodec编译AAC与H264格式，封装mp4

# AudioRecord录制PCM，输出WAV格式，AudioTrack播放PCM

这个任务分为三个部分：

1. 使用系统AudioRecord API录制PCM
2. 通过为PCM格式文件添加文件头，封装为WAV格式
3. 使用系统AudioTrack API播放PCM

## 使用系统AudioRecord API录制PCM

官方API文档：https://developer.android.com/reference/android/media/AudioRecord

话不多说，封装的PcmRecorder代码如下
```java
/**
 * Created by superxlcr on 2019/3/17.
 */

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

说明一下构造函数中需要的几个参数，顺便回顾下一些音频相关的参数知识：

1. sampleRate：采样率，表示每秒的采样点数，根据Android官方文档，目前44.1K是保证在所有设备上都能运行的采样率（详见AudioRecord构造函数doc）
2. channel：声道，Android支持单声道（AudioFormat#CHANNEL_IN_MONO表示）或双声道（AudioFormat#CHANNEL_IN_STEREO表示）
3. bitWidth：位宽，表示每个采样点的存储大小，一般为8bit（AudioFormat#ENCODING_PCM_8BIT表示）或16bit（AudioFormat#ENCODING_PCM_16BIT表示）
4. fileOutputPath：输出文件路径
5. queue：存放PCM数据的阻塞队列，用于配合进行Aac编码

需要值得注意的是，当我们需要计算录音时长时，通过记录一个起始录音时间点，再通过当前录音时间点相减的方式并不可靠（大致原因是AudioRecord#start调用后其实并没有马上开始录音）
比较好的方法时通过 **采样率的公式来计算录音时长**，具体代码如下：
```java
/**
 *	采样率 = 采样数 / 时间 = （数据总大小 / 位宽 / 声道数） / 时间
**/
time = data.length / bitWidth / channel / sampleRate;
```

## 通过为PCM格式文件添加头数据，封装为WAV格式

在录制完PCM文件后，我们可以通过为其添加文件头，使其变成可以被播放器播放的WAV格式

文件头介绍：https://baike.baidu.com/item/%E6%96%87%E4%BB%B6%E5%A4%B4
WAV格式文件头详细介绍：http://soundfile.sapp.org/doc/WaveFormat/
大小端介绍：https://baike.baidu.com/item/%E5%A4%A7%E5%B0%8F%E7%AB%AF%E6%A8%A1%E5%BC%8F/6750542?fromtitle=%E5%A4%A7%E7%AB%AF%E5%B0%8F%E7%AB%AF&fromid=15925891&fr=aladdin

具体代码如下：
```java
/**
 * Created by superxlcr on 2019/3/17.
 */

public class WavEncoder {

    private static final int HEADER_SIZE = 44;

    private int sampleRate;
    private int channel;
    private int bitWidth;

    WavEncoder(int sampleRate, int channel, int bitWidth) {
        this.sampleRate = sampleRate;
        this.channel = (channel == 1 ? 1 : 2);
        this.bitWidth = bitWidth;
    }

    void encode(File pcmFile, File wavFile) {
        try {
            FileInputStream fis = new FileInputStream(pcmFile);
            FileOutputStream fos = new FileOutputStream(wavFile);
            fos.write(buildHeader((int) fis.getChannel().size()), 0, HEADER_SIZE);
            byte data[] = new byte[1024];
            int len;
            while ((len = fis.read(data)) != -1) {
                fos.write(data, 0, len);
            }
            fis.close();
            fos.flush();
            fos.close();
        } catch (IOException e) {
            throw new IllegalStateException("encode with exception : " + e.toString());
        }
    }

    private byte[] buildHeader(int audioSize) {
        byte header[] = new byte[HEADER_SIZE];
        int i = 0;
        // chunk id : big-endian
        header[i++] = 'R';
        header[i++] = 'I';
        header[i++] = 'F';
        header[i++] = 'F';
        // chunk size : little-endian
        int dataSize = audioSize + 36;
        header[i++] = (byte) (dataSize & 0xff);
        header[i++] = (byte) ((dataSize >>> 8) & 0xff);
        header[i++] = (byte) ((dataSize >>> 16) & 0xff);
        header[i++] = (byte) ((dataSize >>> 24) & 0xff);
        // format : big-endian
        header[i++] = 'W';
        header[i++] = 'A';
        header[i++] = 'V';
        header[i++] = 'E';
        // sub chunk 1 id : big-endian
        header[i++] = 'f';
        header[i++] = 'm';
        header[i++] = 't';
        header[i++] = ' ';
        // sub chunk 1 size : little-endian
        // 16 for pcm
        header[i++] = 16;
        header[i++] = 0;
        header[i++] = 0;
        header[i++] = 0;
        // audio format : little-endian
        // 1 for linear quantization pcm
        header[i++] = 1;
        header[i++] = 0;
        // num channels : little-endian
        // just mono or stereo
        header[i++] = (byte) channel;
        header[i++] = 0;
        // sample rate : little-endian
        header[i++] = (byte) (sampleRate & 0xff);
        header[i++] = (byte) ((sampleRate >>> 8) & 0xff);
        header[i++] = (byte) ((sampleRate >>> 16) & 0xff);
        header[i++] = (byte) ((sampleRate >>> 24) & 0xff);
        // byte rate : little-endian
        int byteRate = sampleRate * channel * bitWidth / 8;
        header[i++] = (byte) (byteRate & 0xff);
        header[i++] = (byte) ((byteRate >>> 8) & 0xff);
        header[i++] = (byte) ((byteRate >>> 16) & 0xff);
        header[i++] = (byte) ((byteRate >>> 24) & 0xff);
        // block align : little-endian
        header[i++] = (byte) (channel * bitWidth / 8);
        header[i++] = 0;
        // bit per sample : little-endian
        header[i++] = (byte) bitWidth;
        header[i++] = 0;
        // sub chunk 2 id : big-endian
        header[i++] = 'd';
        header[i++] = 'a';
        header[i++] = 't';
        header[i++] = 'a';
        // sub chunk 2 size : little-endian
        header[i++] = (byte) (audioSize & 0xff);
        header[i++] = (byte) ((audioSize >>> 8) & 0xff);
        header[i++] = (byte) ((audioSize >>> 16) & 0xff);
        header[i++] = (byte) ((audioSize >>> 24) & 0xff);
        if (i != 44) {
            throw new IllegalStateException("the header size is not " + HEADER_SIZE + " ?!");
        }
        return header;
    }

}

```

值得注意的是，在WAV文件头中，字符串相关的参数均为大端模式，而数据类型的均为小端模式

## 使用系统AudioTrack API播放PCM

AudioTrack官方文档：https://developer.android.com/reference/android/media/AudioTrack

相关代码如下：
```java
public class MainActivity extends AppCompatActivity implements View.OnClickListener {

    private static final String TAG = "MainActivity";

    private static final String RECORD_FILE_DIR_PATH = Environment.getExternalStorageDirectory().getAbsolutePath() + File.separator + "recodeDir" + File.separator;
    private static final String RECORD_FILE_PATH = RECORD_FILE_DIR_PATH + "record";
    private static final String RECORD_FILE_WAV_PATH = RECORD_FILE_DIR_PATH + "record_wav.wav";
    private static final String PLAY_FILE_PATH = RECORD_FILE_DIR_PATH + "play";
    private static final String RECORD_FILE_AAC_PATH = RECORD_FILE_DIR_PATH + "record_aac.aac";

    private PcmRecorder pcmRecorder;
    private AacEncoder aacEncoder;
    private WavEncoder wavEncoder;
    private BlockingQueue<byte[]> queue = new ArrayBlockingQueue<>(100);
    private AudioTrack audioTrack;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        findViewById(R.id.start_btn).setOnClickListener(this);
        findViewById(R.id.stop_btn).setOnClickListener(this);
        findViewById(R.id.play_btn).setOnClickListener(this);

        new File(RECORD_FILE_DIR_PATH).mkdirs();

        pcmRecorder = new PcmRecorder(44100, 2, 16, RECORD_FILE_PATH, queue);
        aacEncoder = new AacEncoder(queue, RECORD_FILE_AAC_PATH, pcmRecorder.getSampleRate(),
                pcmRecorder.getChannel(), pcmRecorder.getBitWidth(), null);
        wavEncoder = new WavEncoder(pcmRecorder.getSampleRate(), pcmRecorder.getChannel(),
                pcmRecorder.getBitWidth());
    }

    @SuppressLint("StaticFieldLeak")
    @Override
    public void onClick(View v) {
        switch (v.getId()) {
            case R.id.start_btn:
                pcmRecorder.start();
                aacEncoder.start();
                break;
            case R.id.stop_btn:
                pcmRecorder.stop();
                aacEncoder.stop();
                wavEncoder.encode(new File(RECORD_FILE_PATH), new File(RECORD_FILE_WAV_PATH));
                break;
            case R.id.play_btn:
                if (audioTrack != null) {
                    Toast.makeText(this, "playing...", Toast.LENGTH_SHORT).show();
                    return;
                }
                new AsyncTask<Void, Void, Void>() {
                    @Override
                    protected Void doInBackground(Void... voids) {
                        try {
                            File playFile = new File(RECORD_FILE_PATH);
                            FileInputStream fis = new FileInputStream(playFile);
                            int bufferSize = AudioTrack.getMinBufferSize(
                                    pcmRecorder.getSampleRate(),
                                    pcmRecorder.getChannel() == 1 ? AudioFormat.CHANNEL_OUT_MONO : AudioFormat.CHANNEL_OUT_STEREO,
                                    pcmRecorder.getBitWidth() == 8 ? AudioFormat.ENCODING_PCM_8BIT : AudioFormat.ENCODING_PCM_16BIT);
                            audioTrack = new AudioTrack(AudioManager.STREAM_MUSIC,
                                    pcmRecorder.getSampleRate(),
                                    pcmRecorder.getChannel() == 1 ? AudioFormat.CHANNEL_OUT_MONO : AudioFormat.CHANNEL_OUT_STEREO,
                                    pcmRecorder.getBitWidth() == 8 ? AudioFormat.ENCODING_PCM_8BIT : AudioFormat.ENCODING_PCM_16BIT,
                                    bufferSize, AudioTrack.MODE_STREAM);
                            audioTrack.play();
                            byte buffer[] = new byte[bufferSize];
                            int len;
                            while ((len = fis.read(buffer, 0, buffer.length)) != -1) {
                                audioTrack.write(buffer, 0, len);
                            }
                            fis.close();
                            audioTrack.play();
                        } catch (IOException e) {
                            throw new RuntimeException("ioException : " + e.toString());
                        }
                        Log.i(TAG, "write finish !");
                        return null;
                    }

                    @Override
                    protected void onPostExecute(Void aVoid) {
                        new Handler().postDelayed(new Runnable() {
                            @Override
                            public void run() {
                                Log.i(TAG, "audio track stop");
                                audioTrack.stop();
                                audioTrack = null;
                            }
                        }, 5000);
                    }
                }.execute();
                break;
        }
    }
}
```

值得注意的是，AudioTrack支持两种播放模式，这里用的是AudioTrack#MODE_STREAM模式

- MODE_STREAM：在这种模式下，通过write一次次把音频数据写到AudioTrack中。这和平时通过write系统调用往文件中写数据类似，但这种工作方式每次都需要把数据从用户提供的Buffer中拷贝到AudioTrack内部的Buffer中，这在一定程度上会使引入延时。为解决这一问题，AudioTrack就引入了第二种模式。
- MODE_STATIC：这种模式下，在play之前只需要把所有数据通过一次write调用传递到AudioTrack中的内部缓冲区，后续就不必再传递数据了。这种模式适用于像铃声这种内存占用量较小，延时要求较高的文件。但它也有一个缺点，就是一次write的数据不能太多，否则系统无法分配足够的内存来存储全部数据。

xml文件相对简单，这里不贴出来了，测试时注意申请相关权限
```xml
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
<uses-permission android:name="android.permission.RECORD_AUDIO" />
```