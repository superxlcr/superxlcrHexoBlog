---
title: Android Native进程保活工程解析
tags: [android,cpp]
categories: [android]
date: 2017-05-09 12:29:44
description: Android Native进程保活工程解析
---
最近学习了一个关于Android Native进程保活的工程，在此进行一下总结。


工程github如下：https://github.com/Coolerfall/Android-AppDaemon


其总体的保活工作流程如图所示：
![Native进程保活流程_pic1](1.png)


其工作流程如下：
首先，用户调用Daemon的静态方法run，传入上下文、Service组件类名、以及检查等待的时间间隔：


```java
	/**
	 * Run daemon process.
	 *
	 * @param context            context
	 * @param daemonServiceClazz the name of daemon service class
	 * @param interval           the interval to check
	 */
	public static void run(final Context context, final Class<?> daemonServiceClazz,
	                       final int interval)
```


工程通过android.os.Build.CPU_ABI获取设备的CPU架构，选择对应的二进制文件，通过AssetManager访问文件后，复制到指定位置，使用chmod指令改变文件权限为0755：



```java
		Runtime.getRuntime().exec("chmod " + mode + " " + abspath).waitFor();
```


asset文件目录如下：

![asset_dir_pic2](2.png)



通过shell执行二进制文件，传入包名、类名与时间间隔：


```java
		String cmd = context.getDir(BIN_DIR_NAME, Context.MODE_PRIVATE)
			.getAbsolutePath() + File.separator + DAEMON_BIN_NAME;

		/* create the command string */
		StringBuilder cmdBuilder = new StringBuilder();
		cmdBuilder.append(cmd);
		cmdBuilder.append(" -p ");
		cmdBuilder.append(context.getPackageName());
		cmdBuilder.append(" -s ");
		cmdBuilder.append(daemonClazzName.getName());
		cmdBuilder.append(" -t ");
		cmdBuilder.append(interval);

		try {
			Runtime.getRuntime().exec(cmdBuilder.toString()).waitFor();
		} catch (IOException | InterruptedException e) {
			Log.e(TAG, "start daemon error: " + e.getMessage());
		}
```


在二进制文件中，首先检查了传入的参数是否正确：



```cpp
	if (argc < 7)
	{
		LOGE(LOG_TAG, "usage: %s -p package-name -s "
		 "daemon-service-name -t interval-time", argv[0]);
		return;
	}

	for (i = 0; i < argc; i ++)
	{
		if (!strcmp("-p", argv[i]))
		{
			package_name = argv[i + 1];
			LOGD(LOG_TAG, "package name: %s", package_name);
		}

		if (!strcmp("-s", argv[i]))
		{
			service_name = argv[i + 1];
			LOGD(LOG_TAG, "service name: %s", service_name);
		}

		if (!strcmp("-t", argv[i]))
		{
			interval = atoi(argv[i + 1]);
			LOGD(LOG_TAG, "interval: %d", interval);
		}
	}

	/* package name and service name should not be null */
	if (package_name == NULL || service_name == NULL)
	{
		LOGE(LOG_TAG, "package name or service name is null");
		return;
	}
```


保证传入参数个数正确，包名类名不为空后，开始创建守护进程：



```cpp
	if ((pid = fork()) < 0)
	{
		exit(EXIT_SUCCESS);
	}
	else if (pid == 0)
	{
		...

		/* become session leader */
		setsid();
		/* change work directory */
		chdir("/");

		for (i = 0; i < MAXFILE; i ++)
		{
			close(i);
		}

		...
	}
	else
	{
		/* parent process */
		exit(EXIT_SUCCESS);
	}
```


创建守护进程的流程如下：（此处的方法感觉有点欠缺，具体的可以去查看博客：http://blog.chinaunix.net/uid-25365622-id-3055635.html）
1. 调用fork函数，创建子进程
2. 使进程在后台运行，关闭父进程
3. 调用setsid函数，脱离控制终端，登录会话和进程组（创建新会话）
4. 调用chdir函数，改变当前工作目录至根目录
5. 调用close函数，关闭从父进程继承的文件描述符，一般而言由于没有了输入终端，因此可以关闭标准输入、输出、错误文件（即0、1和2三个文件）

当工程成功创建守护进程后，开始调用signal函数处理信号：

```cpp
// 标识select等待循环是否运行的flag
volatile int sig_running = 1;

// 传入signal的函数，参数为int类型
/* signal term handler */
static void sigterm_handler(int signo)
{
	LOGD(LOG_TAG, "handle signal: %d ", signo);
	sig_running = 0;
}
		
// 处理SIGTERM信号，当进程被杀死时会收到该信号
signal(SIGTERM, sigterm_handler);
```


接着，守护进程处理以前开启的守护进程：

```cpp
		int pid_list[100];
		// find_pid_by_name为工程定义的，通过进程的cmdline文件查找进程的函数
		int total_num = find_pid_by_name(argv[0], pid_list);
		LOGD(LOG_TAG, "total num %d", total_num);
		for (i = 0; i < total_num; i ++)
		{
			int retval = 0;
			int daemon_pid = pid_list[i];
			if (daemon_pid > 1 && daemon_pid != getpid())
			{
				retval = kill(daemon_pid, SIGTERM);
				if (!retval)
				{
					LOGD(LOG_TAG, "kill daemon process success: %d", daemon_pid);
				}
				else
				{
					LOGD(LOG_TAG, "kill daemon process %d fail: %s", daemon_pid, strerror(errno));
					exit(EXIT_SUCCESS);
				}
			}
		}
```


最后，利用循环轮询的方式来唤醒服务组件：

```cpp
		while(sig_running)
		{
			interval = interval < SLEEP_INTERVAL ? SLEEP_INTERVAL : interval;
			select_sleep(interval, 0);

			LOGD(LOG_TAG, "check the service once, interval: %d", interval);

			/* start service */
			start_service(package_name, service_name);
		}
		
/**
 * Use `select` to sleep with specidied second and microsecond.
 */
void select_sleep(long sec, long msec)
{
	struct timeval timeout;

	timeout.tv_sec = sec;
	timeout.tv_usec = msec * 1000;

	select(0, NULL, NULL, NULL, &timeout);
}
```


唤醒组件的代码如下：

```cpp
/* start daemon service */
static void start_service(char *package_name, char *service_name)
{
	/* get the sdk version */
	int version = get_version();

	pid_t pid;

	if ((pid = fork()) < 0)
	{
		exit(EXIT_SUCCESS);
	}
	else if (pid == 0)
	{
		if (package_name == NULL || service_name == NULL)
		{
			LOGE(LOG_TAG, "package name or service name is null");
			return;
		}

		char *p_name = str_stitching(package_name, "/");
		char *s_name = str_stitching(p_name, service_name);
		LOGD(LOG_TAG, "service: %s", s_name);

		if (version >= 17 || version == 0)
		{
			int ret = execlp("am", "am", "startservice",
						"--user", "0", "-n", s_name, (char *) NULL);
			LOGD(LOG_TAG, "result %d", ret);
		}
		else
		{
			execlp("am", "am", "startservice", "-n", s_name, (char *) NULL);
		}

		LOGD(LOG_TAG , "exit start-service child process");
		exit(EXIT_SUCCESS);
	}
	else
	{
		waitpid(pid, NULL, 0);
	}
}
```


在上述代码中，守护进程fork了一个子进程，然后在子进程中通过am的startservice命令，加上我们传入的包名与类名来启动对应的服务组件，启动完成后退出子进程。


因此，为了让服务可被外部调用，该工程要求我们设置Service组件的exported属性：

```html
        <service
            android:name="com.coolerfall.service.DaemonService"
            android:process=":daemon"
            android:exported="true" >
        </service>
```


总的来说，个人认为该工程还有以下待改进的地方：

1. 该工程使用的是轮询的方法来定期给Service发送intent，资源消耗较大
2. Android 5.0以上的系统在清理进程时，会把c进程一同清理掉
3. 其守护方式是单向的，如果c进程挂掉了，将无法守护Service组件
4. 根据源码，其每次开启守护进程时都会清除旧的守护进程，无法同时守护两个Service组件



