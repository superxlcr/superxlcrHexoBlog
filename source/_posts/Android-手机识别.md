---
title: Android 手机识别
tags: [android,应用]
categories: [android]
date: 2017-05-03 12:18:55
description: 使用DeviceId、使用MAC地址、使用UUID、使用Serial Number、使用ANDROID_ID
---
最近博主学习了如何通过某些技术来识别不同的Android手机，在此进行一番总结。
# 使用DeviceId
deviceId为Android提供的用于手机识别的识别码，其获取代码如下：

```java
        TelephonyManager telephonyManager = (TelephonyManager) getSystemService(TELEPHONY_SERVICE);
        Log.d("MyLog", telephonyManager.getDeviceId());
```
打印Log如下：
![device_id_log_pic1](1.png)


getDeviceId方法的文档如下：
![getDeviceId_method_pic2](2.png)
该方法会根据手机的类型为GSM（Global System for Mobile Communication，全球移动通信系统）或CDMA（Code Division Multiple Access，码分多址），返回对应的IMEI（International Mobile Equipment Identity，国际移动设备身份码）、MEID（Mobile Equipment Identifier，移动设备识别码）或ESN码（Electronic Serial Number，电子序列号）。


但该方法获取的deviceId也有一定的缺陷：

1. 非手机的Android设备没有deviceId
2. 获取deviceId需要READ_PHONE_STATE权限
3. 在少数手机上该方法存在bug




# 使用MAC地址
MAC地址即 Media Access Control或者Medium Access Control 地址，意译为媒体访问控制，或称为物理地址、硬件地址，用来定义网络设备的位置。一般而言，每个主机都会有其自己的MAC地址，因此我们可以通过MAC地址来识别一台Android手机。
以下为获取MAC地址例子：
方法一，使用WifiManager，这种方法需要 ACESS_WIFI_STATE 权限：

```java
        String macAddress = null;
        WifiManager wifiManager = (WifiManager)getApplicationContext().getSystemService(Context.WIFI_SERVICE);
        WifiInfo info = (null == wifiManager ? null : wifiManager.getConnectionInfo());
        if (null != info) {
            macAddress = info.getMacAddress();
        }
        Log.d("MyLog", macAddress);
```


值得注意的是，据说在Android 6.0以上版本中，该方法会不再适用，获取的MAC都会变为 02:00:00:00:00:00 默认值。


方法二，使用cmd命令直接访问网卡MAC地址：

```java
        String macSerial = "";
        try {
            Process pp = Runtime.getRuntime().exec(
                    "cat /sys/class/net/wlan0/address");
            InputStreamReader ir = new InputStreamReader(pp.getInputStream());
            LineNumberReader input = new LineNumberReader(ir);

            String line;
            while ((line = input.readLine()) != null) {
                macSerial += line.trim();
            }
            input.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
        Log.d("MyLog", macSerial);
```


这种方法不需要任何权限即可获取MAC地址，不过当手机未连接wifi时，无法获取MAC地址。


通过MAC地址来识别我们的手机具有一定的可行性，但这种方法仍存在一定的缺陷：

1. 不是所有设备都具有网卡或者蓝牙
2. 如果wifi没有打开过，我们便无法获取其MAC地址
3. 蓝牙设备只有在打开的时候才能获取MAC地址

# 使用UUID

UUID含义是通用唯一识别码 (Universally Unique Identifier)，是指在一台机器上生成的数字，它保证对在同一时空中的所有机器都是唯一的。UUID由以下几部分组合而成：


1. 当前日期和时间
2. 时钟序列
3. 全局唯一的IEEE机器识别号（如果有网卡，从网卡获得，没有网卡以其他方式获得）

获取UUID的例子如下：


```java
        UUID uuid = UUID.randomUUID();
        Log.d("MyLog", uuid.toString());
```


打印的log如下：
![uuid_log_pic3](3.png)



使用uuid的缺陷如下：

1. uuid生成的结果较长
2. uuid需要在用户本地保存，因此有可能被篡改

# 使用Serial Number

Serial Number是用于辨识设备的序列号，其获取的例子如下：

```java
Log.d("MyLog", android.os.Build.SERIAL);
```


打印log如下：
![serial_number_log_pic4](4.png)



该方法的缺陷如下：

1. Serial Number在没有IMEI码设备（非手机设备）上必须提供，但手机设备上可能没有
2. 在Android 版本2.3以前可能没有Serial Number

因此，Serial Number可以结合DeviceId来使用



# 使用ANDROID_ID
在设备首次启动时，系统会随机生成一个64位的数字，并把这个数字以16进制字符串的形式保存下来，这个16进制的字符串就是ANDROID_ID，当设备被恢复出厂设置后该值可能会被重置

获取ANDROID_ID的方法如下：

```java
        String ANDROID_ID = Settings.System.getString(getContentResolver(), Settings.Secure.ANDROID_ID);
        Log.d("MyLog", ANDROID_ID);
```


打印Log如下：
![android_id_log_pic5](5.png)



使用ANDROID_ID时，我们需要注意一下问题：

1. 在Android 2.2版本中可能存在问题
2. 可能有设备产生相同ANDROID_ID
3. 由于厂商定制系统可能导致bug，返回null

综上所述，Android 手机识别的几种方法可以用下图进行总结：

![Android 手机识别_pic6](6.png)

