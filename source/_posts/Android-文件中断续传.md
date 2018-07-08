---
title: Android 文件中断续传
tags: [android,应用]
categories: [android]
date: 2017-05-29 11:44:40
description: 功能原理、代码
---
最近尝试了Android 的文件中断续传功能，在此写下一篇博客进行记录。


# 功能原理
实现文件中断续传功能主要依赖的原理有两个：

1. 通过RandomAccessFile文件定位功能在特定位置写入文件
2. 通过HTTP 1.1协议的Range头域来获取下载文件的特定部分




# 代码

```java
public class FileContinueDownload {

    private static final String SP_NAME = "FileContinueDownload";
    private static final int BUFFER_SIZE = 512;

    private Context context;
    private String fileUrl;
    private String fileName;
    private String filePath;
    private FileContinueDownloadListener listener;

    private SharedPreferences sharedPreferences;
    private RandomAccessFile randomAccessFile;
    private long startPosition;

    private volatile boolean work;

    /**
     * 
     * @param context 上下文
     * @param fileUrl url
     * @param fileName 文件名
     * @param filePath 文件存储路径
     * @param listener 监听器
     */
    public FileContinueDownload(Context context, String fileUrl, String fileName, String filePath, FileContinueDownloadListener listener) {
        this.context = context.getApplicationContext();
        this.fileUrl = fileUrl;
        this.fileName = fileName;
        this.filePath = filePath;
        this.listener = listener;
        init();
    }

    /**
     * 开始下载文件
     */
    public void start() {
        if (work) {
            return;
        }
        work = true;
        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    // 开启网络连接
                    URL url = new URL(fileUrl);
                    HttpURLConnection httpURLConnection = (HttpURLConnection) url.openConnection();
                    httpURLConnection.setConnectTimeout(3000);
                    httpURLConnection.setRequestMethod("GET");
                    httpURLConnection.setRequestProperty("Range", "bytes=" + startPosition + "-");
                    int responseCode = httpURLConnection.getResponseCode();
                    if (responseCode == 200 || responseCode == 206) {
                        int fileLength = httpURLConnection.getContentLength();
                        InputStream inputStream = httpURLConnection.getInputStream();
                        byte buffer[] = new byte[BUFFER_SIZE];
                        int len;
                        // 下载文件
                        while ((len = inputStream.read(buffer)) != -1) {
                            randomAccessFile.write(buffer, 0, len);
                            // 更新进度
                            listener.onUpdate((int) randomAccessFile.getFilePointer(), fileLength);
                            // 更新sharedPreferences
                            SharedPreferences.Editor editor = sharedPreferences.edit();
                            startPosition = randomAccessFile.getFilePointer();
                            editor.putLong(fileName, startPosition);
                            editor.apply();
                            // 判断是否暂停
                            if (!work) {
                                break;
                            }
                        }
                        if (work) { // 工作状态退出，代表已下载完毕
                            SharedPreferences.Editor editor = sharedPreferences.edit();
                            editor.remove(fileName);
                            editor.apply();
                            listener.onFinish();
                        }
                    }
                } catch (Exception e) {
                    listener.onError(e);
                }
            }
        }).start();
    }

    /**
     * 暂停下载文件
     */
    public void pause() {
        work = false;
    }

    private void init() {
        try {
            // 获取sharedPreferences
            sharedPreferences = context.getSharedPreferences(SP_NAME, Context.MODE_PRIVATE);
            if (!sharedPreferences.contains(fileName)) {
                // 不存在进度，创建文件
                File file = new File(filePath, fileName);
                if (!file.exists()) {
                    file.createNewFile();
                }
                randomAccessFile = new RandomAccessFile(file, "rwd");
                SharedPreferences.Editor editor = sharedPreferences.edit();
                editor.putLong(fileName, 0);
                editor.apply();
                startPosition = 0;
            } else {
                // 存在进度，获取进度
                File file = new File(filePath, fileName);
                randomAccessFile = new RandomAccessFile(file, "rwd");
                startPosition = sharedPreferences.getLong(fileName, 0);
                randomAccessFile.seek(startPosition);
            }
        } catch (Exception e) {
            listener.onError(e);
            return;
        }
    }

}

interface FileContinueDownloadListener {

    void onUpdate(int nowProgress, int totalProgress);

    void onError(Exception e);

    void onFinish();
}
```


FileContinueDownload为文件中断续传帮助类，通过Android的SharedPreferences来记录文件的下载进度，使用RandomAccessFile的seek方法来找到合适的位置进行文件写入，通过HTTP 1.1的Range头域来获取剩余的部分文件，最终结果或者错误通过监听器进行返回。



测试的例子如下：

```java
public class MainActivity extends AppCompatActivity {

    private static MyHandler handler;

    private static final String URL = "https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1495972879917&di=327cc1f368b192793d82a2320e8b7599&imgtype=0&src=http%3A%2F%2Fi2.hdslb.com%2Fbfs%2Farchive%2Fb622e10be9e9a63ca2922348541ec86b55c0fc53.jpg";
    private static final String FILE_NAME = "file_continue_download_test.jpg";
    private static final String FILE_PATH = Environment.getExternalStorageDirectory().getPath();

    private static final int HANDLER_START = 0;
    private static final int HANDLER_FINISH = 1;

    private TextView nameTV;
    private ProgressBar progressBar;
    private Button startBtn;
    private Button pauseBtn;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        initView();
        handler = new MyHandler(this);

        FileContinueDownloadListener listener = new FileContinueDownloadListener() {
            @Override
            public void onUpdate(int nowProgress, int totalProgress) {
                Message message = handler.obtainMessage(HANDLER_START, nowProgress, totalProgress);
                handler.sendMessage(message);
            }

            @Override
            public void onError(Exception e) {
                Log.e("MyLog", Log.getStackTraceString(e));
            }

            @Override
            public void onFinish() {
                Message message = handler.obtainMessage(HANDLER_FINISH);
                handler.sendMessage(message);
            }
        };
        final FileContinueDownload download = new FileContinueDownload(this, URL, FILE_NAME, FILE_PATH, listener);

        startBtn.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                download.start();
                startBtn.setEnabled(false);
                pauseBtn.setEnabled(true);
            }
        });
        pauseBtn.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                download.pause();
                startBtn.setEnabled(true);
                pauseBtn.setEnabled(false);
            }
        });
    }

    private void initView() {
        nameTV = (TextView) findViewById(R.id.textView);
        nameTV.setText(FILE_NAME);
        progressBar = (ProgressBar) findViewById(R.id.progressBar);
        progressBar.setMax(1);
        progressBar.setProgress(0);
        startBtn = (Button) findViewById(R.id.start);
        pauseBtn = (Button) findViewById(R.id.pause);
        startBtn.setEnabled(true);
        pauseBtn.setEnabled(false);
    }

    static class MyHandler extends Handler {

        private SoftReference<MainActivity> reference;

        MyHandler(MainActivity mainActivity) {
            reference = new SoftReference<MainActivity>(mainActivity);
        }

        @Override
        public void handleMessage(Message msg) {
            MainActivity activity = reference.get();
            if (activity != null && msg.what == HANDLER_START) {
                int nowProgress = msg.arg1;
                int totalProgress = msg.arg2;
                activity.progressBar.setMax(totalProgress);
                activity.progressBar.setProgress(nowProgress);
            } else if (activity != null && msg.what == HANDLER_FINISH) {
                Toast.makeText(activity, "download finish!", Toast.LENGTH_SHORT).show();
                activity.startBtn.setEnabled(false);
                activity.pauseBtn.setEnabled(false);
            }
        }
    }
}
```


xml文件就不贴出来了，里面是一个TextView、ProgressBar以及两个Button。


效果如下所示：


下图分别为初始界面、开始下载、暂停下载


![界面_pic1](1.jpg)![开始下载_pic2](2.jpg)![暂停_pic3](3.jpg)


下载到一半的file_continue_download_test.jpg文件：
![下载一半的文件_pic4](4.jpg)



继续下载以及下载完成的文件：
![继续下载_pic5](5.jpg)![下载完成的文件_pic6](6.jpg)

