---
title: Android生成渠道包总结
tags: [android,应用,编译]
categories: [android]
date: 2017-07-20 20:46:54
description: 什么是渠道包、Gradle构筑渠道包的方法、使用apktool反编译apk加入渠道信息、META-INF添加渠道信息
---
最近在工作上了解了一些与渠道包相关的信息，在此进行一下总结。
# 什么是渠道包
每当发新版本时，我们编写的Android客户端应用会被分发到各个应用市场，比如豌豆荚，360手机助手等。为了统计应用这些市场的效果（活跃数，下单数等），我们需要有一种唯一标识来区分它们。渠道号就是我们用来区分不同市场的唯一标识，比方说，发布到豌豆荚市场的应用的渠道号是“wandoujia”，而发布到360手机助手的应用的渠道号则为“qihu360”。带有渠道号的包即是渠道包，当我们的应用进行打点汇报等操作时，往往会把渠道包中的渠道信息一同上传，以便后台接下来计算不同渠道的效果。

# Gradle构筑渠道包的方法
在Android的Gradle中，它为我们提供了Flavor属性用于构建不同渠道的渠道包，使用方法如下：File -&gt; Project Structure -&gt; Flavors 选项卡
![gradle_build_channel_pic1](1.png)
设置Flavors过后，我们的build.gradle文件将会添加上对应的代码（当然我们也可以手动直接编写代码）：
![productFlavors_pic2](2.png)

Gradle提供的Flavors允许我们设置多种多样的属性，而我们可以通过Gradle生成的BuildConfig类来读取当前渠道包的相关信息：
```java
BuildConfig.FLAVOR
```

这种打渠道包的方法的优点在于可以对渠道包进行多种属性的定制，然而缺点在于每打一个渠道包都要重新执行一次编译过程，当渠道的数量较多或者工程编译过程较长时，会耗费相当多的时间，而且当渠道包多到一定程度的时候，配置渠道包的冗长的脚本也会让人抓狂。
# 使用apktool反编译apk加入渠道信息
apktool是用于编译以及反编译apk的工具，我们可以通过使用apktool来反编译我们打出的普通apk文件，为其中添加渠道信息后再重新编译。具体操作流程如下：
首先我们在AndroidManifest中添加元数据的渠道信息：
```html
<meta-data
            android:name="channel"
            android:value="BASE_CHANNEL" />
```

接着执行我们的gradle任务，在此，我们把任务分为几个步骤一一列出（下列代码写在Module的build.gradle文件中）首先，通过apktool反编译apk：

```java
task decompileApk(type: Exec, dependsOn: 'assembleRelease') {
    workingDir 'build/outputs/apk'
    commandLine 'java', '-jar', 'apktool_2.2.3.jar', 'd', '-f', 'app-release.apk', '-o', 'base-apk'
}
```

反编译后我们把文件保存在base-apk目录下，其中可以看到我们要加入渠道信息的AndroidManifest.xml文件：
```html
<?xml version="1.0" encoding="utf-8" standalone="no"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android" package="com.example.superxlcr.buildchanneltest" platformBuildVersionCode="26" platformBuildVersionName="8.0.0">
    <meta-data android:name="android.support.VERSION" android:value="26.0.0-alpha1"/>
    <application android:allowBackup="true" android:icon="@mipmap/ic_launcher" android:label="@string/app_name" android:supportsRtl="true" android:theme="@style/AppTheme">
        <activity android:name="com.example.superxlcr.buildchanneltest.MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>
        <meta-data android:name="channel" android:value="BASE_CHANNEL"/>
    </application>
</manifest>

```

接下来我们读取channel.txt文件的渠道信息，并返回一个列表：
```java
def ArrayList<String> getChannelList() {
    ArrayList<String> list = new ArrayList<String>();
    file('build/outputs/apk/channel.txt').eachLine { channel ->
        list.add(channel);
    }
    return list;
}
```

然后我们继续执行任务，替换xml中的‘BASE_CHANNEL’字符串为我们的渠道信息：
```java
task addChannelInfo(dependsOn: decompileApk) << {
    final String BASE_CHANNEL_NAME = 'BASE_CHANNEL';
    ArrayList<String> channelList = getChannelList();
    for (String channel : channelList) {
        String channelDir = "build/outputs/apk/${channel}";
        copy {
            from 'build/outputs/apk/base-apk'
            into channelDir
        }
        file("${channelDir}/AndroidManifest_new.xml").createNewFile();
        file("${channelDir}/AndroidManifest_new.xml").withWriter('UTF-8') { writer ->
            file("${channelDir}/AndroidManifest.xml").withReader('UTF-8') { reader ->
                reader.eachLine { line ->
                    writer.append(line.replaceAll(BASE_CHANNEL_NAME, channel));
                    writer.append('\r\n');
                }
            }
        }
        file("${channelDir}/AndroidManifest.xml").delete();
        file("${channelDir}/AndroidManifest_new.xml").renameTo(file("${channelDir}/AndroidManifest.xml"));
    }
}
```

最后我们再通过apktool回编译出我们的apk文件，此时apk文件由于进行了修改，因此需要进行重新签名验证：
```java
task buildChannelApk(dependsOn: addChannelInfo) << {
    getChannelList().each { channel ->
        exec {
            workingDir 'build/outputs/apk'
            commandLine 'java', '-jar', 'apktool_2.2.3.jar', 'b', channel, '-o', "${channel}-unsigned.apk"
        }
        exec {
            workingDir 'build/outputs/apk'
            commandLine 'jarsigner', '-sigalg', 'MD5withRSA',
                    '-digestalg', 'SHA1',
                    '-keystore', 'your_keystore_path',
                    '-storepass', 'your_keystore_password',
                    '-signedjar', "${channel}.apk", "${channel}-unsigned.apk", 'your_alias'
        }
        file("build/outputs/apk/${channel}-unsigned.apk").delete();
        file("build/outputs/apk/${channel}").deleteDir();
        file('build/outputs/apk/base-apk').deleteDir();
    }
}
```

最后我们可以得到的以渠道名命名的apk包

当我们需要获取渠道信息时，调用以下函数即可：
```java
    private String getChannel(Context context) {
        try {
            PackageManager pm = context.getPackageManager();
            ApplicationInfo appInfo = pm.getApplicationInfo(context.getPackageName(), PackageManager.GET_META_DATA);
            return appInfo.metaData.getString("channel");
        } catch (PackageManager.NameNotFoundException e) {
            e.printStackTrace();
        }
        return "";
    }
```

这种使用apktool工具来添加渠道信息的方法虽然没有Gradle自带的编写渠道包方法定制功能强，但会相对比较快，因为编译过程较短，主要需要重新签名，而且配置渠道文件也较为简单，仅为txt文本。
# META-INF添加渠道信息
使用apktool工具虽然比Gradle提供的打渠道包快，但因为修改了文件还是要执行签名工作，有没有什么方法呢能直接修改apk的渠道号，而不需要再重新签名呢？答案是有的，当我们解压apk文件时，我们会发现一个META-INF目录

该目录主要存放于检验apk文件相关的签名文件与校验文件等。对于apk的签名方法，往META-INF中添加空白文件并不会影响apk的校验值，因此我们可以以我们的渠道名新建一个空白文件添加进去即可。gradle代码如下，自动构建apk目录下channel.txt文件中的渠道包：
```java
task addChannelInfoByAnt(dependsOn: 'assembleRelease') << {
    def dir = 'build/outputs/apk'
    getChannelList().each { channel ->
        copy {
            from "${dir}/app-release.apk"
            into dir
            rename {
                "${channel}.apk"
            }
        }
        file("${dir}/${channel}").mkdir();
        file("${dir}/${channel}/META-INF").mkdir();
        file("${dir}/${channel}/META-INF/channel_${channel}").createNewFile();
        ant.zip(basedir: "${dir}/${channel}", includes: "META-INF/channel_${channel}", keepcompression: true, update: true, destfile: "${dir}/${channel}.apk");
        file("${dir}/${channel}").deleteDir();
    }
}
```
这里解释一下ant.zip这句代码：这句代码使用ant工具创建了一个zip类型的task执行压缩任务，其中includes参数表示要包含的文件，默认值为xx，即basedir下所有文件
keepcompression参数表示对已压缩的文件保持压缩状态而非重新压缩
update参数表示如果destfile目标文件已经出现，进行更新而非重写覆盖更详细的资料可以参考官方的文档
http://ant.apache.org/manual/Tasks/zip.html


python代码如下：
```
import zipfile
zipped = zipfile.ZipFile(your_apk, 'a', zipfile.ZIP_DEFLATED) 
empty_channel_file = "META-INF/channel_{channel}".format(channel=your_channel)
zipped.write(your_empty_file, empty_channel_file)
```

上面代码渠道信息我们添加了channel_的前缀。虽说apk可以通过zip的方法打开，但是貌似不能随便使用zip来进行解压与并重新压缩，本人试过多次Java的ZipEntry等方式来解压压缩apk，均以失败告终。目前发现可以更新apk压缩包的方法仅以上两种。（看到ant官方zip类型task相关的资料描述，猜测与zip压缩时间戳相关，重新解压压缩会影响zip文件的时间戳相关参数）
当我们需要在Java中取得渠道信息时，调用以下代码即可：
```java
public static String getChannel(Context context) {
        ApplicationInfo appinfo = context.getApplicationInfo();
        String sourceDir = appinfo.sourceDir;
        String ret = "";
        ZipFile zipfile = null;
        try {
            zipfile = new ZipFile(sourceDir);
            Enumeration<?> entries = zipfile.entries();
            while (entries.hasMoreElements()) {
                ZipEntry entry = ((ZipEntry) entries.nextElement());
                String entryName = entry.getName();
                if (entryName.startsWith("channel")) {
                    ret = entryName;
                    break;
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            if (zipfile != null) {
                try {
                    zipfile.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }

        String[] split = ret.split("_");
        if (split != null && split.length >= 2) {
            return ret.substring(split[0].length() + 1);

        } else {
            return "";
        }
    }
```

相较于其他两种添加渠道信息的方法，往META-INF中添加空白文件由于不用签名，属于最快、最方便的一种方法了。
 