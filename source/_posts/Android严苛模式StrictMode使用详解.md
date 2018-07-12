---
title: Android严苛模式StrictMode使用详解
tags: [android,内存,应用]
categories: [android]
date: 2017-07-15 16:48:01
description: （转载）StrictMode具体能检测什么、工作原理、常见用法、查看报告结果、ThreadPolicy 详解、VMPolicy 详解、其他操作、注意事项
---


StrictMode类是Android 2.3 （API 9）引入的一个工具类，可以用来帮助开发者发现代码中的一些不规范的问题，以达到提升应用响应能力的目的。举个例子来说，如果开发者在UI线程中进行了网络操作或者文件系统的操作，而这些缓慢的操作会严重影响应用的响应能力，甚至出现ANR对话框。为了在开发中发现这些容易忽略的问题，我们使用StrictMode，系统检测出主线程违例的情况并做出相应的反应，最终帮助开发者优化和改善代码逻辑。
官网文档：http://developer.android.com/reference/android/os/StrictMode.html

# StrictMode具体能检测什么
严苛模式主要检测两大问题，一个是线程策略，即ThreadPolicy，另一个是VM策略，即VmPolicy。
## ThreadPolicy线程策略检测
- 自定义的耗时调用 使用detectCustomSlowCalls()开启
- 磁盘读取操作 使用detectDiskReads()开启
- 磁盘写入操作 使用detectDiskWrites()开启
- 网络操作 使用detectNetwork()开启

## VmPolicy虚拟机策略检测
- Activity泄露 使用detectActivityLeaks()开启
- 未关闭的Closable对象泄露 使用detectLeakedClosableObjects()开启
- 泄露的Sqlite对象 使用detectLeakedSqlLiteObjects()开启
- 检测实例数量 使用setClassInstanceLimit()开启

# 工作原理
其实StrictMode实现原理也比较简单，以IO操作为例，主要是通过在open，read，write，close时进行监控。libcore.io.BlockGuardOs文件就是监控的地方。以open为例，如下进行监控。

```java
@Override
public FileDescriptor open(String path, int flags, int mode) throws ErrnoException {
  BlockGuard.getThreadPolicy().onReadFromDisk();
    if ((mode & O_ACCMODE) != O_RDONLY) {
      BlockGuard.getThreadPolicy().onWriteToDisk();
    }
    return os.open(path, flags, mode);
}
```

其中onReadFromDisk()方法的实现，代码位于StrictMode.Java中。

```java
public void onReadFromDisk() {
    if ((mPolicyMask & DETECT_DISK_READ) == 0) {
      return;
    }
    if (tooManyViolationsThisLoop()) {
      return;
    }
    BlockGuard.BlockGuardPolicyException e = new StrictModeDiskReadViolation(mPolicyMask);
    e.fillInStackTrace();
    startHandlingViolationException(e);
}
```

# 常见用法
严格模式的开启可以放在Application或者Activity以及其他组件的onCreate方法。为了更好地分析应用中的问题，建议放在Application的onCreate方法中。
其中，我们只需要在app的开发版本下使用 StrictMode，线上版本避免使用 StrictMode，这里定义了一个布尔值变量DEV_MODE来进行控制。

```java
private boolean DEV_MODE = true;
 public void onCreate() {
     if (DEV_MODE) {
         StrictMode.setThreadPolicy(new StrictMode.ThreadPolicy.Builder()
                 .detectCustomSlowCalls() //API等级11，使用StrictMode.noteSlowCode
                 .detectDiskReads()
                 .detectDiskWrites()
                 .detectNetwork()   // or .detectAll() for all detectable problems
                 .penaltyDialog() //弹出违规提示对话框
                 .penaltyLog() //在Logcat 中打印违规异常信息
                 .penaltyFlashScreen() //API等级11
                 .build());
         StrictMode.setVmPolicy(new StrictMode.VmPolicy.Builder()
                 .detectLeakedSqlLiteObjects()
                 .detectLeakedClosableObjects() //API等级11
                 .penaltyLog()
                 .penaltyDeath()
                 .build());
     }
     super.onCreate();
 }
```
其中Android3.0引入的方法包括detectCustomSlowCalls()和noteSlowCode()，它们都是用来检测应用中执行缓慢代码的或者潜在的缓慢代码。
# 查看报告结果
严格模式有很多种报告违例的形式，但是想要分析具体违例情况，还是需要查看日志，终端下过滤StrictMode就能得到违例的具体stacktrace信息。

```
adb logcat | grep StrictMode
```
![日志_pic1](1.jpg)
当然也可以选择弹窗形式来简明提醒开发者
![弹窗警告_pic2](2.jpg)
# ThreadPolicy 详解
StrictMode.ThreadPolicy.Builder 主要方法如下
## detectNetwork()
用于检查UI线程中是否有网络请求操作

检测UI线程中网络请求案例：
```java
public class MainActivity extends AppCompatActivity implements View.OnClickListener {

    Button btnTest;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        StrictMode.setThreadPolicy(new StrictMode.ThreadPolicy.Builder()
                .detectNetwork()
                .penaltyLog()
                .build());
        btnTest = (Button) findViewById(R.id.btn_test);
        btnTest.setOnClickListener(this);
    }

    @Override
    public void onClick(View v) {
        int id = v.getId();
        switch (id) {
            case R.id.btn_test:
                postNetwork();
            break;
        }
    }

    /**
     * 网络连接的操作
     */
    private void postNetwork() {
        try {
            URL url = new URL("http://www.wooyun.org");
            HttpURLConnection conn = (HttpURLConnection) url.openConnection();
            conn.connect();
            BufferedReader reader = new BufferedReader(new InputStreamReader(
                    conn.getInputStream()));
            String lines = null;
            StringBuffer sb = new StringBuffer();
            while ((lines = reader.readLine()) != null) {
                sb.append(lines);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

运行后，触发的警告如下
![日志_pic3](3.jpg)
## detectDiskReads()和detectDiskWrites()
是磁盘读写检查


```java
磁盘读写检查案例：
public 

    Button btnTest;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        StrictMode.setThreadPolicy(new StrictMode.ThreadPolicy.Builder()
                .detectDiskWrites()
                .detectDiskReads()
                .penaltyLog()
                .build());
        btnTest = (Button) findViewById(R.id.btn_test);
        btnTest.setOnClickListener(this);
    }

    @Override
    public void onClick(View v) {
        int id = v.getId();
        switch (id) {
            case R.id.btn_test:
                writeToExternalStorage();
                break;
        }
    }

    /**
     * 文件系统的操作
     */
    public void writeToExternalStorage() {
        File externalStorage = Environment.getExternalStorageDirectory();
        File mbFile = new File(externalStorage, "castiel.txt");
        try {
            OutputStream output = new FileOutputStream(mbFile, true);
            output.write("www.wooyun.org".getBytes());
            output.flush();
            output.close();
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

运行后，触发的警告如下 
![日志_pic4](4.jpg)
## noteSlowCall
针对执行比较耗时的检查
StrictMode从 API 11开始允许开发者自定义一些耗时调用违例，这种自定义适用于自定义的任务执行类中，比如我们有一个进行任务处理的类，为TaskExecutor。


```java
public class TaskExecutor {
    public void execute(Runnable task) {
        task.run();
    }
}
```

先需要跟踪每个任务的耗时情况，如果大于500毫秒需要提示给开发者，noteSlowCall就可以实现这个功能，如下修改代码

```java
public class TaskExecutor {

    private static long SLOW_CALL_THRESHOLD = 500;
    public void executeTask(Runnable task) {
        long startTime = SystemClock.uptimeMillis();
        task.run();
        long cost = SystemClock.uptimeMillis() - startTime;
        if (cost > SLOW_CALL_THRESHOLD) {
            StrictMode.noteSlowCall("slowCall cost=" + cost);
        }
    }
}
```

执行一个耗时2000毫秒的任务

```java
TaskExecutor executor = new TaskExecutor();
executor.executeTask(new Runnable() {
  @Override
    public void run() {
        try {
          Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
});
```
得到的违例日志，注意其中~duration=20 ms并非耗时任务的执行时间，而我们的自定义信息msg=slowCall cost=2000才包含了真正的耗时。
## penaltyDeath()
当触发违规条件时，直接Crash掉当前应用程序。
## penaltyDeathOnNetwork()
当触发网络违规时，Crash掉当前应用程序。
## penaltyDialog()
触发违规时，显示对违规信息对话框。
## penaltyFlashScreen()
会造成屏幕闪烁，不过一般的设备可能没有这个功能。
## penaltyDropBox()
将违规信息记录到 dropbox 系统日志目录中（/data/system/dropbox），你可以通过如下命令进行插件：


```
adb shell dumpsys dropbox dataappstrictmode  --print
```

## permitCustomSlowCalls()、permitDiskReads ()、permitDiskWrites()、permitNetwork()
如果你想关闭某一项检测，可以使用对应的permit方法。

# VMPolicy 详解
StrictMode.VmPolicy.Builder 主要方法如下
## detectActivityLeaks()
用户检查 Activity 的内存泄露情况

内存泄露检查案例：
```java
public class MainActivity extends AppCompatActivity implements View.OnClickListener {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        StrictMode.setVmPolicy(new StrictMode.VmPolicy.Builder()
                .detectActivityLeaks()
                .penaltyLog()
                .build()
        );

        new Thread() {
            @Override
            public void run() {
                while (true) {

                    SystemClock.sleep(1000);
                }
            }
        }.start();

    }
}
```

我们反复旋转屏幕就会输出提示信息（重点在 instances=2; limit=1 这一行） 
![日志_pic5](5.jpg)
这时因为，我们在Activity中创建了一个Thread匿名内部类，而匿名内部类隐式持有外部类的引用。而每次旋转屏幕是，android会新创建一个Activity，而原来的Activity实例又被我们启动的匿名内部类线程持有，所以不会释放，从日志上看，当先系统中该Activty有4个实例，而限制是只能创建1各实例。我们不断翻转屏幕，instances 的个数还会持续增加。
## detectLeakedClosableObjects()
用于资源没有正确关闭时提醒


```java
// 资源引用没有关闭检查案例
public class MainActivity extends AppCompatActivity implements View.OnClickListener {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        StrictMode.setVmPolicy(new StrictMode.VmPolicy.Builder()
                .detectLeakedClosableObjects()
                .penaltyLog()
                .build()
        );

        File newxmlfile = new File(Environment.getExternalStorageDirectory(), "castiel.txt");
        try {
            newxmlfile.createNewFile();
            FileWriter fw = new FileWriter(newxmlfile);
            fw.write("猴子搬来的救兵WooYun");
            //fw.close(); 我们在这里特意没有关闭 fw
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

运行后触发警告如下 
![日志_pic7](6.jpg)
## detectLeakedSqlLiteObjects()和detectLeakedClosableObjects()
用法类似，只不过是用来检查 SQLiteCursor 或者 其他 SQLite 对象是否被正确关闭
## detectLeakedRegistrationObjects()
用来检查 BroadcastReceiver 或者ServiceConnection 注册类对象是否被正确释放
## setClassInstanceLimit()
设置某个类的同时处于内存中的实例上限，可以协助检查内存泄露


```java
// 检测内存泄露案例
public class MainActivity extends AppCompatActivity implements View.OnClickListener {

    private static }
    private static List<CastielClass> classList;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        classList = new ArrayList<CastielClass>();

        StrictMode.setVmPolicy(new StrictMode.VmPolicy.Builder()
                .setClassInstanceLimit(CastielClass.class, 2)
                .penaltyLog()
                .build());

        classList.add(new CastielClass());
        classList.add(new CastielClass());
        classList.add(new CastielClass());
        classList.add(new CastielClass());
        classList.add(new CastielClass());
        classList.add(new CastielClass());
        classList.add(new CastielClass());
        classList.add(new CastielClass());

    }
}
```


# 其他操作
除了通过日志查看之外，我们也可以在开发者选项中开启严格模式，开启之后，如果主线程中有执行时间长的操作，屏幕则会闪烁，这是一个更加直接的方法。 
![开发者模式_pic9](7.jpg)
# 注意事项
- 只在开发阶段启用StrictMode，发布应用或者release版本一定要禁用它。
- 严格模式无法监控JNI中的磁盘IO和网络请求。
- 应用中并非需要解决全部的违例情况，比如有些IO操作必须在主线程中进行。
