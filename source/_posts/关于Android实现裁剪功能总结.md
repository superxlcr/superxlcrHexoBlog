---
title: 关于Android实现裁剪功能总结
tags: [android,应用]
categories: [android]
date: 2017-04-17 17:42:07
description: 使用com.android.camera.action.crop、com.android.camera.action.crop的缺点、其他的图片裁剪办法
---
最近在进行毕业设计的时候，需要实现一个图片裁剪的功能。在此，对Android系统如何实现图片裁剪功能进行一个小结。

# 使用com.android.camera.action.crop
com.android.camera.action.crop是Android系统提供的一个Intent，我们可以利用该Intent打开一个裁剪用的Activity，然后通过onActivityResult返回或者把图片保存到外部的方式获取裁剪的结果。

com.android.camera.action.crop所使用的参数如下表所示：


| 附加选项 | 数据类型 | 描述 | 
| - | - | - |
| crop | String | 发送裁剪信号，“true”表示启用裁剪 | 
| aspectX | int | X方向比例 | 
| aspectY | int | Y方向比例 | 
| outputX | int | 裁剪区的宽 | 
| outputY | int | 裁剪区的高 | 
| scale | boolean | 是否保留比例 | 
| return-data | boolean | 是否将裁剪数据保留在Bitmap中返回 | 
| data | Parcelable | 需要裁剪的Bitmap数据 | 
| circleCrop | boolean | 是否圆形裁剪区域 | 
| MediaStore.EXTRA_OUTPUT | URI | 裁剪数据输出位置 | 


下面为一段裁剪图片的例子：

```java
    private void cropPhoto(Uri uri) {
        Intent intent = new Intent("com.android.camera.action.CROP");
        intent.setDataAndType(uri, "image/*");
        intent.putExtra("scale", true);
        // 裁剪比例
        intent.putExtra("aspectX", xxx);
        intent.putExtra("aspectY", xxx);
        // 裁剪宽高
        intent.putExtra("outputX", xxx);
        intent.putExtra("outputY", xxx);
        // 文件输出位置
        intent.putExtra(MediaStore.EXTRA_OUTPUT, imageUri);
        startActivityForResult(intent, CROP_PHOTO);
    }
```




```java
Intent photoPickerIntent = new Intent(Intent.ACTION_PICK, android.provider.MediaStore.Images.Media.EXTERNAL_CONTENT_URI);
photoPickerIntent.setType("image/*");
photoPickerIntent.putExtra("crop", "true");
photoPickerIntent.putExtra(MediaStore.EXTRA_OUTPUT, xxx);
photoPickerIntent.putExtra("outputFormat", Bitmap.CompressFormat.JPEG.toString());
startActivityForResult(photoPickerIntent, REQ_CODE_PICK_IMAGE);
```





在例子中我们设置了MediaStore.EXTRA_OUTPUT参数，因此裁剪的结果将会保存在我们参数的URI中。



# com.android.camera.action.crop的缺点
虽然com.android.camera.action.crop这个API看起来很方便，然而它还是有非常明显的缺点的。
首先，它不是一个官方公开的API接口，在Android官方网站上我们找不到和该接口有关的资料，同时Google可以在不发表任何通知的情况下更改或取消该接口。虽然它一般情况下在大部分的设备上都可以使用，但并不能保证100%不会出现问题或导致APP crash。
其次，在使用“return-data”参数获取返回值时，获得的图片在大小尺寸上会有较大的问题。当我们裁剪图片的尺寸大小在300像素以上时，APP有极大可能会crash，更甚者会死机直到你拆掉手机电池重新安上才解决。因此，当我们使用这个接口时，应尽量避免通过onActivityResult返回裁剪结果。





# 其他的图片裁剪办法
由于com.android.camera.action.crop这个API并非官方的正式API，再加上在通过onActivityResult获取的图片尺寸有一定的问题，因此我们需要使用一些别的方法来实现图片裁剪功能。在此，我们可以使用一些第三方库来实现图片裁剪的功能。
第三方图片裁剪库github：https://github.com/lvillani/android-cropimage
使用例子如下：

```java
private void doCrop(File croppedResult){
        CropImageIntentBuilder builder = new CropImageIntentBuilder(600,600, croppedResult);
        // don't forget this, the error handling within the library is just ignoring if you do
        builder.setSourceImage(mImageCaptureUri);
        Intent  intent = builder.getIntent(getApplicationContext());
        // do not use return data for big images
        intent.putExtra("return-data", false);
        // start an activity and then get the result back in onActivtyResult
        startActivityForResult(intent, CROP_FROM_CAMERA);
    }
```


对于图片裁剪问题更详细的讨论请查看StackOverflow：
http://stackoverflow.com/questions/12758425/how-to-set-the-output-image-use-com-android-camera-action-crop

