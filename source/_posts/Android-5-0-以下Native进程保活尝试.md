---
title: Android 5.0 以下Native进程保活尝试
tags: [android,cpp,应用]
categories: [android]
date: 2017-05-26 17:33:53
description: Android 5.0 以下Native进程保活尝试
---
最近博主尝试了Android 5.0 以下版本的Native保活机制，感觉收获颇丰，在此写下一篇博客记录一下。
首先把整个保活流程通过图片的形式描述下：
![Native进程保活流程_pic1](1.png)


首先是AndroidManifest 中注册的控件：


```html
        <activity android:name=".MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />

                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
        <service android:name=".PersistService"
            android:process=":persist" />
        <receiver android:name=".WakeUpBroadcastReceiver"
            android:process=":wake_up"
            android:exported="true"/>
```


主要有三个控件：一个用于启动服务的Activity，一个在:persist 子进程中的运行任务的服务Service，以及一个在:wake_up 子进程中用于拉活Service的BroadcastReceiver



MainActivity过于简单在此就不做介绍了，PersistService的代码如下：


```java
public class PersistService extends Service {

    private static final String TAG = "MyLog";

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }

    @Override
    public void onCreate() {
        super.onCreate();
        // 运行守护Daemon
        Log.d(TAG, "run daemon");
        Daemon.run(this, WakeUpBroadcastReceiver.class);
        new Thread(new Runnable() {
            @Override
            public void run() {
                int i = 0;
                while (true) {
                    Log.d("TestLive", "number is " + i++);
                    try {
                        Thread.sleep(5000);
                    } catch (InterruptedException e) {}
                }
            }
        }).start();
    }

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        Log.d(TAG, "onStartCommand");
        return super.onStartCommand(intent, flags, startId);
    }
}
```


在PersistService的onCreate方法中，调用了Daemon类启动守护进程对PersistService类进行守护，并启动了新线程计数模拟工作。

Daemon类代码如下：


```java
    private static final String DIR_NAME = "bin";
    private static final String FILE_NAME = "serviceDaemon";

    private static AlarmManager alarmManager;
    public static PendingIntent wakeUpIntent;

    static {
        System.loadLibrary("daemon-lib");
    }

    /**
     * 运行守护进程
     * @param context 上下文
     * @param wakeUpClass 唤醒Service类
     */
    public static void run(final Context context, final Class wakeUpClass) {
        // 初始化闹钟与唤醒用的intent
        alarmManager = ((AlarmManager)context.getSystemService(Context.ALARM_SERVICE));
        Intent intent = new Intent(context, wakeUpClass);
        wakeUpIntent = PendingIntent.getBroadcast(context, 0, intent, 0);
        // 启动守护进程
        new Thread(new Runnable() {
            @Override
            public void run() {
                // 复制binary文件
                String binaryFilePath = Command.install(context, DIR_NAME, FILE_NAME);
                if (binaryFilePath != null) {
                    // 运行
                    start(context.getPackageName(), wakeUpClass.getSimpleName(), binaryFilePath);
                }
            }
        }).start();
    }

    private native static void start(String packageName, String serviceClassName, String binaryFilePath);
```


在run方法中，首先调用了Command工具类根据系统ABI获取在assets文件夹下相应的二进制文件，复制到特定地点后更改权限等待执行
然后启动子线程通过JNI调用native的start方法，传入包名、需要唤醒的类名以及二进制文件的路径：


```cpp
extern "C"
JNIEXPORT void JNICALL Java_com_app_superxlcr_mynativetest_Daemon_start
        (JNIEnv *env, jclass clazz, jstring jPackageName, jstring jWakeUpClassName,
         jstring jBinaryFilePath) {
    // 获取Java参数
    const char *packageName = env->GetStringUTFChars(jPackageName, 0);
    const char *wakeUpClassName = env->GetStringUTFChars(jWakeUpClassName, 0);
    const char *binaryFilePath = env->GetStringUTFChars(jBinaryFilePath, 0);
    LOGD(MY_NATIVE_TAG, "packageName is %s", packageName);
    LOGD(MY_NATIVE_TAG, "wakeUpCLassName is %s", wakeUpClassName);
    LOGD(MY_NATIVE_TAG, "binaryFilePath is %s", binaryFilePath);
    // 建立管道，pipe1用于父进程监听子进程，pipe2用于子进程监听父进程
    int pipe1Fd[2];
    int pipe2Fd[2];
    if (pipe(pipe1Fd) == -1) {
        LOGE(MY_NATIVE_TAG, "create pipe1 error");
    }
    if (pipe(pipe2Fd) == -1) {
        LOGE(MY_NATIVE_TAG, "create pipe2 error");
    }
    // 执行二进制文件
    pid_t result = fork();
    if (result > 0) { // 父进程
        // 关闭pipe1的写端
        close(pipe1Fd[1]);
        // 关闭pipe2的读端
        close(pipe2Fd[0]);
        // 监听子进程情况
        char readBuffer[100];
        int readResult = read(pipe1Fd[0], readBuffer, 100);
        LOGD(MY_NATIVE_TAG, "readResult is %d, errno is %d", readResult, errno);
        // 阻塞中断，子进程已退出
        LOGD(MY_NATIVE_TAG, "child process is dead");
        // 释放Java参数内存
        env->ReleaseStringUTFChars(jPackageName, packageName);
        env->ReleaseStringUTFChars(jWakeUpClassName, wakeUpClassName);
        env->ReleaseStringUTFChars(jBinaryFilePath, binaryFilePath);
        // 拉活处理，回调Java方法
        jmethodID methodID = env->GetStaticMethodID(clazz, "onDaemonDead", "()V");
        env->CallStaticVoidMethod(clazz, methodID, NULL);
        return;
    } else if (result == 0) { // 子进程
        // 管道描述符转为字符串
        char strP1r[10];
        char strP1w[10];
        char strP2r[10];
        char strP2w[10];
        sprintf(strP1r, "%d", pipe1Fd[0]);
        sprintf(strP1w, "%d", pipe1Fd[1]);
        sprintf(strP2r, "%d", pipe2Fd[0]);
        sprintf(strP2w, "%d", pipe2Fd[1]);
        // 执行二进制文件
        LOGD(MY_NATIVE_TAG, "execute binary file");
        LOGD(MY_NATIVE_TAG, "binary file argv ：%s %s %s %s %s %s %s %s %s %s %s %s %s",
             BINARY_FILE_NAME,
             PACKAGE_NAME, packageName,
             SERVICE_CLASS_NAME, wakeUpClassName,
             PIPE_1_READ, strP1r,
             PIPE_1_WRITE, strP1w,
             PIPE_2_READ, strP2r,
             PIPE_2_WRITE, strP2w);
        execlp(binaryFilePath,
               BINARY_FILE_NAME,
               PACKAGE_NAME, packageName,
               SERVICE_CLASS_NAME, wakeUpClassName,
               PIPE_1_READ, strP1r,
               PIPE_1_WRITE, strP1w,
               PIPE_2_READ, strP2r,
               PIPE_2_WRITE, strP2w,
               NULL);
        // 函数返回，执行失败
        LOGE(MY_NATIVE_TAG, "execute binary file fail, errno is %d", errno);
    } else {
        LOGE(MY_NATIVE_TAG, "fork fail!");
    }
    // 释放Java参数内存
    env->ReleaseStringUTFChars(jPackageName, packageName);
    env->ReleaseStringUTFChars(jWakeUpClassName, wakeUpClassName);
    env->ReleaseStringUTFChars(jBinaryFilePath, binaryFilePath);
}
```


在start方法中，我们首先获取了从Java层传过来的参数，然后通过pipe函数开启了两条匿名管道用于服务进程与其后代守护进程监听存活情况：


- pipe1：管道1，用于父进程监听子进程的存活情况，父进程关闭pipe1的写端，子进程关闭pipe1的读端
- pipe2：管道2，用于子进程监听父进程的存活情况，父进程关闭pipe2的读端，子进程关闭pipe2的写端

然后通过fork创建相应的子进程，在父进程中通过管道监听子进程的状况，在子进程中通过exec函数执行二进制文件重生为守护进程。


当父进程管道读取返回时，代表子进程已被销毁。此时，父进程通过回调Java层的onDeamonDead方法唤醒拉活进程并自杀等待重启：

```java
    public static void onDaemonDead() {
        Log.d(TAG, "call onDaemonDead!");
        // 闹钟启动拉活进程
        alarmManager.set(AlarmManager.ELAPSED_REALTIME, SystemClock.elapsedRealtime() + 5000, wakeUpIntent);
        // 自杀
        Process.killProcess(Process.myPid());
    }
```


二进制文件代码：


```cpp
int main(int argc, char *argv[]) {
    // 参数个数检查
    if (argc < 13) {
        LOGE(MY_NATIVE_TAG, "argc number is %d, expected 13!", argc);
        return 0;
    }
    char *processName = argv[0];
    char *packageName = NULL;
    char *wakeUpClassName = NULL;
    int pipe1Fd[2];
    int pipe2Fd[2];
    for (int i = 0; i < argc; i++) {
        if (!strcmp(PACKAGE_NAME, argv[i])) {
            packageName = argv[i + 1];
            LOGD(MY_NATIVE_TAG, "packageName is : %s", packageName);
        }
        if (!strcmp(SERVICE_CLASS_NAME, argv[i])) {
            wakeUpClassName = argv[i + 1];
            LOGD(MY_NATIVE_TAG, "wakeUpClassName is : %s", wakeUpClassName);
        }
        if (!strcmp(PIPE_1_READ, argv[i])) {
            pipe1Fd[0] = atoi(argv[i + 1]);
            LOGD(MY_NATIVE_TAG, "pipe1r is : %d", pipe1Fd[0]);
        }
        if (!strcmp(PIPE_1_WRITE, argv[i])) {
            pipe1Fd[1] = atoi(argv[i + 1]);
            LOGD(MY_NATIVE_TAG, "pipe1w is : %d", pipe1Fd[1]);
        }
        if (!strcmp(PIPE_2_READ, argv[i])) {
            pipe2Fd[0] = atoi(argv[i + 1]);
            LOGD(MY_NATIVE_TAG, "pipe2r is : %d", pipe2Fd[0]);
        }
        if (!strcmp(PIPE_2_WRITE, argv[i])) {
            pipe2Fd[1] = atoi(argv[i + 1]);
            LOGD(MY_NATIVE_TAG, "pipe2w is : %d", pipe2Fd[1]);
        }
    }

    // 成为守护进程
    int forkResult = fork();
    if (forkResult == 0) {
        // 成为会话组长
        setsid();
        // 改变工作目录
        chdir("/");

        // 清理僵尸进程
        cleanZombieProcess(processName);

        // 管道处理
        // 关闭pipe1的读端
        close(pipe1Fd[0]);
        // 关闭pipe2的写端
        close(pipe2Fd[1]);

        // 管道监听，监听父进程情况
        char readBuffer[100];
        int readResult = read(pipe2Fd[0], readBuffer, 100);
        LOGD(MY_NATIVE_TAG, "readResult is %d, errno is %d", readResult, errno);
        // 阻塞中断，父进程已退出
        LOGD(MY_NATIVE_TAG, "parent process is dead");
        // 拉活处理
        pid_t childPid = fork();
        if (childPid == 0) {
            // 唤醒广播拉活
            char *wakeUpName = new char[strlen(packageName) + strlen(wakeUpClassName) + 1];
            sprintf(wakeUpName, "%s/.%s", packageName, wakeUpClassName);
            LOGD(MY_NATIVE_TAG, "wakeUpName is %s", wakeUpName);
            int result = execlp("am", "am", "broadcast",
                                "--user", "0", "-n", wakeUpName, (char *) NULL);
            LOGD(MY_NATIVE_TAG, "execute am broadcast result is %d", result);
        } else if (childPid > 0) {
            waitpid(childPid, NULL, 0);
            LOGD(MY_NATIVE_TAG, "execute am broadcast over");
        } else {
            LOGE(MY_NATIVE_TAG, "fork fail!");
        }
        return 0;
    } else if (forkResult > 0) {
        return 0;
    } else {
        LOGE(MY_NATIVE_TAG, "fork fail!");
    }
}
```





在二进制文件中，首先检查了传入的参数，然后使该进程成为Linux的守护进程（没有关闭标准输入输出错误、关闭后执行am指令会出问题），并清理了以前剩下的僵尸守护进程，然后通过管道监听父进程的存活情况，当父进程死亡时，通过exec函数执行am指令发送广播唤醒拉活进程，并退出程序关闭守护进程。


负责拉活进程的广播接收器如下：

```java
public class WakeUpBroadcastReceiver extends BroadcastReceiver {

    private static final String TAG = "MyLog";

    @Override
    public void onReceive(Context context, Intent intent) {
        // 拉活处理
        Log.d(TAG, "Receiver Start Service");
        Intent serviceIntent = new Intent(context, PersistService.class);
        context.startService(serviceIntent);
    }
}
```


下面是在Android 4.3 CoolPad手机上测试的效果：
首次启动：
![首次启动_pic2](2.png)



在adb shell中通过force-stop关闭服务进程：
![force-stop_pic3](3.png)



守护进程检测到Service进程被杀死，进行拉活处理：
![子进程唤醒_pic4](4.png)



此次守护进程的pid为21561，我们尝试通过kill命令杀死守护进程

Service进程检测到守护进程被杀死，进行拉活处理：
![父进程唤醒_pic5](5.png)



通过测试发现，无论是Service进程死亡还是守护进程死亡，另一方都能通过唤醒拉活进程将其救活，真正实现了Native层的双进程守护功能。


然而，在Android 5.0以上版本该方法并没有效果……目前博主正在进行进一步的学习研究。


项目git地址：https://github.com/superxlcr/MyNativeTest

