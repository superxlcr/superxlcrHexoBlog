---
title: Android 加壳尝试（一）
tags: [android,应用]
categories: [android]
date: 2017-09-11 21:00:15
description: 实现效果、原理简介、存在问题
---
最近看了一篇Android加壳相关的文章：http://blog.csdn.net/jiangwei0910410003/article/details/48415225
尝试根据文章的步骤来实现Android加壳的功能，在发现文章实现的效果不大理想后，本人进行了一定的调整与改进



# 实现效果

实现效果如下：
![_pic1](1.png)



reinforceTest是我们的加壳Android工程，我们把需要加壳的apk放置在其workspace目录下
接着在reinforceTest工程下运行gradle的task：buildReinforceApk
![_pic2](2.png)

在workspace目录下，我们可以找到加壳后的output.apk
![_pic3](3.png)

加壳后的apk可正常运行：
![_pic4](4.png)

通过dex2jar以及jd-gui，我们可以看到dex中的壳代码，但是看不到源apk的代码：
![_pic5](5.png)




# 原理简介

加壳的原理大致如下图所示：
![_pic6](6.png)





壳apk（即reinforceTest工程）的主要作用用是提供壳classes.dex，用于在启动app时解析出源classes.dex，并引导程序执行classes.dex的代码
引导程序执行真正classes.dex文件的步骤如下：
![_pic7](7.png)



引导的Application如下：

```java
public class ReinforceApplication extends Application {

    private static final String TAG = ReinforceApplication.class.getSimpleName();

    private static final String ACTIVITY_THREAD = "android.app.ActivityThread";
    private static final String LOADED_APK = "android.app.LoadedApk";

    private static final String APPLICATION_CLASS_NAME = "APPLICATION_CLASS_NAME";

    private String mDexFileName;
    private String mOdexPath;
    private String mLibPath;

    @Override
    protected void attachBaseContext(Context base) {
        super.attachBaseContext(base);
        initDexEnvironment();
        decryptDex();
        replaceDexLoader();
    }

    @Override
    public void onCreate() {
        // TODO provider onCreate?
        String appClassName = null;
        // 获取Application名字
        try {
            ApplicationInfo ai = this.getPackageManager()
                    .getApplicationInfo(this.getPackageName(),
                            PackageManager.GET_META_DATA);
            Bundle bundle = ai.metaData;
            if (bundle != null && bundle.containsKey(APPLICATION_CLASS_NAME)) {
                appClassName = bundle.getString(APPLICATION_CLASS_NAME);
            } else {
                return;
            }
        } catch (PackageManager.NameNotFoundException e) {
            Log.e(TAG, Log.getStackTraceString(e));
        }
        Log.i(TAG, appClassName);

        Object sCurrentActivityThread = RefInvoke.invokeStaticMethod(
                ACTIVITY_THREAD, "currentActivityThread",
                new Class[]{}, new Object[]{});
        Object mBoundApplication = RefInvoke.getFieldObject(
                ACTIVITY_THREAD, "mBoundApplication", sCurrentActivityThread);
        Object info = RefInvoke.getFieldObject(
                ACTIVITY_THREAD + "$AppBindData", "info", mBoundApplication);
        // 把当前进程的mApplication 设置成null
        RefInvoke.setFieldObject(LOADED_APK, "mApplication", info, null);
        // 删除oldApplication
        Object oldApplication = RefInvoke.getFieldObject(
                ACTIVITY_THREAD, "mInitialApplication", sCurrentActivityThread);
        ArrayList<Application> mAllApplications = (ArrayList<Application>) RefInvoke
                .getFieldObject(ACTIVITY_THREAD, "mAllApplications", sCurrentActivityThread);
        mAllApplications.remove(oldApplication);

        ApplicationInfo appInfoInLoadedApk = (ApplicationInfo) RefInvoke
                .getFieldObject(LOADED_APK, "mApplicationInfo", info);
        ApplicationInfo appInfoInAppBindData = (ApplicationInfo) RefInvoke
                .getFieldObject(ACTIVITY_THREAD + "$AppBindData", "appInfo", mBoundApplication);
        appInfoInLoadedApk.className = appClassName;
        appInfoInAppBindData.className = appClassName;
        // 执行 makeApplication（false,null），此功能需要把当前进程的mApplication 设置成null
        Application app = (Application) RefInvoke.invokeMethod(
                LOADED_APK, "makeApplication", info,
                new Class[]{boolean.class, Instrumentation.class},
                new Object[]{false, null});
        RefInvoke.setFieldObject(ACTIVITY_THREAD, "mInitialApplication", sCurrentActivityThread,
                app);

        ArrayMap mProviderMap = (ArrayMap) RefInvoke
                .getFieldObject(ACTIVITY_THREAD, "mProviderMap", sCurrentActivityThread);
        Iterator it = mProviderMap.values().iterator();
        while (it.hasNext()) {
            Object providerClientRecord = it.next();
            Object localProvider = RefInvoke
                    .getFieldObject(ACTIVITY_THREAD + "$ProviderClientRecord", "mLocalProvider",
                            providerClientRecord);
            RefInvoke.setFieldObject("android.content.ContentProvider", "mContext", localProvider,
                    app);
        }

        Log.i(TAG, "app:" + app);
        app.onCreate();
    }

    private void initDexEnvironment() {
        mDexFileName = getApplicationInfo().dataDir + "/real.dex";
        mOdexPath = getApplicationInfo().dataDir + "/odex";
        File odexDir = new File(mOdexPath);
        if (!odexDir.exists()) {
            odexDir.mkdir();
        }
        mLibPath = getApplicationInfo().nativeLibraryDir;
    }

    private void decryptDex() {
        byte[] dex = readDexFromApk();
        if (dex != null) {
            byte[] realDexBytes = decryption(dex);
            if (realDexBytes != null) {
                try {
                    File realDex = new File(mDexFileName);
                    if (realDex.exists()) {
                        realDex.delete();
                    }
                    realDex.createNewFile();
                    FileOutputStream fos = new FileOutputStream(realDex);
                    fos.write(realDexBytes);
                    fos.flush();
                    fos.close();
                } catch (IOException e) {
                    Log.e(TAG, Log.getStackTraceString(e));
                }
            }
        }
    }

    private byte[] readDexFromApk() {
        File sourceApk = new File(getPackageCodePath());
        try {
            ZipInputStream zis = new ZipInputStream(new FileInputStream(sourceApk));
            ByteArrayOutputStream baos = new ByteArrayOutputStream();
            ZipEntry entry;
            while ((entry = zis.getNextEntry()) != null) {
                if (entry.getName().equals("classes.dex")) {
                    byte[] bytes = new byte[1024];
                    int len;
                    while ((len = zis.read(bytes)) != -1) {
                        baos.write(bytes, 0, len);
                        baos.flush();
                    }
                    return baos.toByteArray();
                }
            }
            zis.close();
            return null;
        } catch (IOException e) {
            Log.e(TAG, Log.getStackTraceString(e));
            return null;
        }
    }

    private byte[] decryption(byte[] dex) {
        int totalLen = dex.length;
        byte[] realDexLenBytes = new byte[4];
        System.arraycopy(dex, totalLen - 4, realDexLenBytes, 0, 4);
        ByteArrayInputStream bais = new ByteArrayInputStream(realDexLenBytes);
        DataInputStream ins = new DataInputStream(bais);
        int realDexLen;
        try {
            realDexLen = ins.readInt();
        } catch (IOException e) {
            Log.e(TAG, Log.getStackTraceString(e));
            return null;
        }
        byte[] realDexBytes = new byte[realDexLen];
        System.arraycopy(dex, totalLen - 4 - realDexLen, realDexBytes, 0, realDexLen);
        return decrypt(realDexBytes);
    }

    private byte[] decrypt(byte[] bytes) {
        // TODO
        byte[] result = new byte[bytes.length];
        for (int i = 0; i < bytes.length; i++) {
            result[i] = (byte) (bytes[i] ^ 0x4598);
        }
        return result;
    }

    private void replaceDexLoader() {
        Object sCurrentActivityThread = RefInvoke
                .invokeStaticMethod(ACTIVITY_THREAD, "currentActivityThread", null, null);
        String packageName = getPackageName();
        ArrayMap mPackages = (ArrayMap) RefInvoke
                .getFieldObject(ACTIVITY_THREAD, "mPackages", sCurrentActivityThread);
        WeakReference weakReference = (WeakReference) mPackages.get(packageName);
        Object loadedApk = weakReference.get();
        ClassLoader mClassLoader = (ClassLoader) RefInvoke
                .getFieldObject(LOADED_APK, "mClassLoader", loadedApk);
        DexClassLoader dexClassLoader = new DexClassLoader(mDexFileName, mOdexPath, mLibPath,
                mClassLoader);
        RefInvoke.setFieldObject(LOADED_APK, "mClassLoader", loadedApk, dexClassLoader);
    }
}
```





reinforceTest工程具体的加壳步骤如下所示：

```java
task buildReinforceApk(dependsOn: 'assembleDebug') << {
    // 清理目录
    cleanDir();
    // 解压apk
    decodeApk();
    // 修改Manifest文件
    modifyManifest();
    // 加壳
    reinforce();
    // 重新打包apk并签名
    rebuildAndSign();
}
```


这里的Gradle Task依赖了assembleDebug Task，用于获取最新的壳apk


清理目录就不多说了，解压apk使用的是apktool工具，把源apk与壳apk反编译出来：

```java
private void decodeApk() {
    // 复制解壳apk
    copy {
        from 'build/outputs/apk/app-debug.apk'
        into WORKSPACE
        rename {
            REIN_FORCE_APK
        }
    }
    // 解压解壳apk
    exec {
        workingDir WORKSPACE
        commandLine 'java', '-jar', TOOLS_DIR + APK_TOOL, 'd', '-s', REIN_FORCE_APK, '-o', REIN_FORCE_DIR
    }
    // 解压源apk
    exec {
        workingDir WORKSPACE
        commandLine 'java', '-jar', TOOLS_DIR + APK_TOOL, 'd', '-s', SRC_APK, '-o', SRC_DIR
    }
}
```


接着是修改源apk的AndroidManifest.xml文件，为啥要修改呢？首先因为源apk上有我们需要的资源文件，所以我们肯定是把加壳后的dex放入源apk中，而不是放入壳apk中。由上面的引导步骤图我们得知，我们解壳时需要先执行壳apk的Application。如果加壳后的dex放入源apk中，我们的解壳Application由于没有在源apk的AndroidManifest中注册，因此无法率先执行。所以，我们需要修改源apk的AndroidManifest，把Application的name改为壳apk的Application，同时添加一项meta-data，用于记录源apk的Application的名字，让我们在待会替换回真正的Application时知道它的名字：

```java
private void modifyManifest() {
    // 声明命名空间
    def android = new Namespace('http://schemas.android.com/apk/res/android', 'android')

    // 获取源apk application name
    def parser = new XmlParser()
    def srcManifest = parser.parse("${WORKSPACE}${SRC_DIR}/AndroidManifest.xml")
    def srcApp = srcManifest.application[0].attribute(android.name)

    // 获取壳apk application name
    def reinforceManifest = new XmlParser().parse("${WORKSPACE}${REIN_FORCE_DIR}/AndroidManifest.xml")
    def reinforceApp = reinforceManifest.application[0].attribute(android.name)

    // 合成新Manifest
    // 新建meta-data节点，记录源apk application Name
    parser.createNode(
            srcManifest.application[0],
            new QName('http://schemas.android.com/apk/res/android', 'meta-data'),
            [
                    (android.name):'APPLICATION_CLASS_NAME',
                    (android.value):srcApp
            ]
    )

    // 修改application节点，改为壳apk application Name
    srcManifest.application[0].attributes().put(android.name, reinforceApp)
    println srcManifest.application[0].attribute(android.name)

    // 写入文件
    Writer writer = new FileWriter("${WORKSPACE}${SRC_DIR}/AndroidManifest.xml")
    writer.write(XmlUtil.serialize(srcManifest))
    writer.flush()
}
```


接着就是加壳的步骤了，这里主要调用了用Java写的加壳工具：

```java
private void reinforce() {
    // 加壳
    OutputStream os = new ByteArrayOutputStream();
    exec {
        workingDir TOOLS_DIR
        // 参数为 源dex 壳dex 输出dex
        commandLine 'java', '-cp', '.', 'DexTools', "${WORKSPACE}${SRC_DIR}/classes.dex", "${WORKSPACE}${REIN_FORCE_DIR}/classes.dex", WORKSPACE + OUTPUT_DEX
        standardOutput = os;
    }
    println os.toString()
    // 输出dex替换源dex
    file("${WORKSPACE}${SRC_DIR}/classes.dex").delete();
    copy {
        from WORKSPACE + OUTPUT_DEX
        into WORKSPACE + SRC_DIR
    }
}
```


工具Java代码如下：

```java
public class DexTools {

	public static void main(String[] args) {
		if (args.length != 3) {
			System.out.println("plz enter srcDex , reinforceDex and outputDex");
			System.exit(0);
		}
		try {
			// 源dex
			File srcDex = new File(args[0]);
			// 壳dex
			File reinForceDex = new File(args[1]);

			// 对源dex进行加密
			byte[] encryptSrcApkBytes = encrypt(readFileBytes(srcDex));
			byte[] reinForceDexBytes = readFileBytes(reinForceDex);

			// 新dex长度，4字节用于存放源apk长度
			int totalLen = encryptSrcApkBytes.length + reinForceDexBytes.length
					+ 4;
			byte[] newDex = new byte[totalLen];

			// 先拷贝壳dex
			System.arraycopy(reinForceDexBytes, 0, newDex, 0,
					reinForceDexBytes.length);
			// 再拷贝源dex
			System.arraycopy(encryptSrcApkBytes, 0, newDex,
					reinForceDexBytes.length, encryptSrcApkBytes.length);
			// 写上源dex长度
			System.arraycopy(int2byte(encryptSrcApkBytes.length), 0, newDex,
					totalLen - 4, 4);

			// 修改dex文件长度
			fixHeaderFileSize(newDex);
			// 修改dex签名
			fixHeaderSignature(newDex);
			// 修改dex校验和
			fixHeaderCheckSum(newDex);

			File outputDex = new File(args[2]);
			if (!outputDex.exists()) {
				outputDex.createNewFile();
			}
			FileOutputStream fos = new FileOutputStream(outputDex);
			fos.write(newDex);
			fos.flush();
			fos.close();
		} catch (Exception e) {
			e.printStackTrace();
		}
	}

	private static byte[] readFileBytes(File file) {
		if (file.canRead()) {
			try {
				FileInputStream fis = new FileInputStream(file);
				ByteArrayOutputStream baos = new ByteArrayOutputStream();
				int len;
				byte[] bytes = new byte[1024];
				while ((len = fis.read(bytes)) != -1) {
					baos.write(bytes, 0, len);
				}
				fis.close();
				return baos.toByteArray();
			} catch (IOException e) {
				e.printStackTrace();
			}
		}
		return null;
	}

	private static byte[] encrypt(byte[] bytes) {
		// TODO
		byte[] result = new byte[bytes.length];
		for (int i = 0; i < bytes.length; i++) {
			result[i] = (byte) (bytes[i] ^ 0x4598);
		}
		return result;
	}

	private static byte[] int2byte(int number) {
		byte[] bytes = new byte[4];
		for (int i = 3; i >= 0; i--) {
			bytes[i] = (byte) (number % 256);
			number >>= 8;
		}
		return bytes;
	}

	private static void fixHeaderFileSize(byte[] dex) {
		byte[] newSize = int2byte(dex.length);
		// 修改（32-35）file_size
		System.arraycopy(changeBytesOrder(newSize), 0, dex, 32, 4);
	}

	private static void fixHeaderSignature(byte[] dex)
			throws NoSuchAlgorithmException {
		MessageDigest md = MessageDigest.getInstance("SHA-1");
		// 计算从32位到文件尾的sha-1值
		md.update(dex, 32, dex.length - 32);
		byte[] newSignature = md.digest();
		// 修改（12-31）signature
		System.arraycopy(newSignature, 0, dex, 12, 20);
	}

	private static void fixHeaderCheckSum(byte[] dex) {
		Adler32 adler32 = new Adler32();
		// 计算从12位到文件尾的校验和
		adler32.update(dex, 12, dex.length - 12);
		long checkSum = adler32.getValue();
		byte[] checkSumBytes = int2byte((int) checkSum);
		// 修改（8-11）checkSum
		System.arraycopy(changeBytesOrder(checkSumBytes), 0, dex, 8, 4);
	}

	private static byte[] changeBytesOrder(byte[] bytes) {
		int length = bytes.length;
		byte[] result = new byte[length];
		for (int i = 0; i < length; i++) {
			result[i] = bytes[length - 1 - i];
		}
		return result;
	}
}
```


具体的加壳原理可以参考文章顶部的链接，其中有提及，在此不做赘述
加壳后的dex如下所示：
![_pic8](8.png)



加壳完后把新的dex覆盖旧的源apk的dex，由于文件进行了改动，因此apk在重新打包后需要重新签名：

```java
private void rebuildAndSign() {
    // 打包apk
    exec {
        workingDir WORKSPACE
        commandLine 'java', '-jar', TOOLS_DIR + APK_TOOL, 'b', SRC_DIR
    }
    // 复制打包完的apk
    copy {
        from "${WORKSPACE}${SRC_DIR}/dist/${SRC_APK}"
        into WORKSPACE
        rename {
            OUTPUT_UNSIGNED_APK
        }
    }
    exec {
        workingDir WORKSPACE
        commandLine 'jarsigner', '-sigalg', 'MD5withRSA',
                '-digestalg', 'SHA1',
                '-keystore', rootDir.getAbsolutePath() + '/reinforceTestKey.jks',
                '-storepass', '123456',
                '-signedjar', OUTPUT_APK, OUTPUT_UNSIGNED_APK, 'Dummy'
    }
}
```


# 存在问题


目前这种加壳方式在测试过程中发现仍存在一些问题：

1. 不支持AppCompatActivity：当源apk使用AppCompatActivity时，会出现资源找不到的错误，具体原因未知，以后再做研究
2. ContentProvider不清楚是否支持：根据执行顺序，在APK中最先执行的几个方法应该为：Application.attachBaseContext -&gt; ContentProvider.onCreate -&gt; Application.onCreate。这里我们选择在Application.attachBaseContext以及Application.onCreate进行解壳以及引导程序执行的操作，不清楚是否会对ContentProvider造成影响，打算以后再做测试




github地址：https://github.com/superxlcr/reinforceTest
