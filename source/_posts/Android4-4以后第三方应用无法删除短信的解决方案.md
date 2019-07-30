---
title: Android4.4以后第三方应用无法删除短信的解决方案
date: 2019-07-30 13:22:44
tags:
    - 短信
categories: Android
---


## 概述

最近在做短信删除功能,发现在红米6上Android8.0上无法删除短信

## 分析

在google查阅后得知：Android为了防止第三方软件拦截短信和偷发短信吸费，在android4.4之后，只有默认的短信应用才有权限操作短信数据库。

Android4.4短信机制的改变：
[Getting Your SMS Apps Ready for KitKat](https://android-developers.googleblog.com/2013/10/getting-your-sms-apps-ready-for-kitkat.html)

<!-- more -->

### 4.4 之前：

- 新接收短信广播 `SMS_RECEIVED_ACTION` 为有序广播。任意应用可接到该广播并中止其继续传播。中止后优先级低的短信应用和系统短信服务将不知道新短信到达，从而不写进数据库。这样就做到了拦截（其实很多恶意应用也这么干）。
- 任意应用都可以操作短信数据库，包括新建（含伪造收件箱和发件箱短信）、修改（含篡改历史短信）、删除。
- 任意应用都可以发送短信和彩信，但默认不写进短信数据库，除非应用手动存入，否则用户是看不到的（配合拦截就可以安静地吸费了）。

### 4.4 及之后：

- 设立默认短信应用机制，成为默认短信后的应用将全面接管（替代）系统短信服务。与设置默认浏览器类似，成为默认短信应用需要向用户申请。
- 新接收短信广播 `SMS_RECEIVED_ACTION` 更改为无序广播，增加只有默认短信应用能够接收的广播 `SMS_DELIVER_ACTION`和`WAP_PUSH_DELIVER_ACTION` 。二者的不同在于，当默认短信应用收到 `SMS_DELIVER_ACTION` 时它要负责将其存入数据库。任意应用仍然可以接收到 `SMS_RECEIVED_ACTION` 广播但不能将其中止。因此所有的应用和系统短信服务都可以接收到新短信，没有应用能够再用中止广播的方式拦截短信。
- 只有默认短信应用可以操作短信数据库，包括新建（含伪造收件箱和发件箱短信）、修改（含篡改历史短信）和删除。其它应用只能读取短信数据库。默认短信应用需要在发送短信、收到新短信之后手动写入系统短信数据库，否则其它应用将读取不到该条短信。默认短信应用可以通过控制不写入数据库的方式拦截短信。
- 任意应用仍然都可以发送短信，但默认短信应用以外的应用发短信的接口底层改为调用系统短信服务，而不再直接调用驱动通信，因此其所发短信会被系统短信服务自动转存数据库。此外，只有默认短信应用可以发送彩信。

### 简单来说，第三方非默认短信应用：

- 可以收短信、发短信并接收短信回执，但是不能删除短信
- 可以查询短信数据库，但是不能新增、删除、修改短信数据库
- 无法拦截短信

但是！极少数国产手机厂商会修改这个机制，实际测试中发现小米就修改了这个机制！小米4，android6.0系统，miui稳定版8.2，运行非默认应用，居然还是可以删除短信。但是别的小米手机又是不行的。非常奇怪。

## 如何解决

### 提示用户设置自己的app为default SMS app

为了使我们的应用出现在系统设置的 `Default SMS app` 中，我们需要在Manifest中做一些声明，获取对应的权限：

- 声明一个 `broadcast receiver` 控件，对`SMS_DELIVER_ACTION`广播进行监听，当然这个`receiver`也要声明`BROADCAST_SMS`权限。
- 声明一个  `broadcast receiver` 控件，对`WAP_PUSH_DELIVER_ACTION`广播进行监听，当然这个`receiver`也要声明`BROADCAST_WAP_PUSH`权限。
- 在短信发送界面，需要监听 `ACTION_SENDTO`，同时配置上`sms:, smsto:, mms:, and mmsto`这四个概要，这样别的应用如果想发送短信，你的这个activity就能知道。
- 需要有一个`service`，能够监听`ACTION_RESPONSE_VIA_MESSAGE`，同时也要配置上`sms:, smsto:, mms:, and mmsto` 这四个概要，并且要声明`SEND_RESPOND_VIA_MESSAGE`权限。这样用户就能在来电的时候，用你的应用来发送拒绝短信。

```java
<manifest>
    ...
    <application>
        <!-- BroadcastReceiver that listens for incoming SMS messages -->
        <receiver android:name=".SmsReceiver"
                android:permission="android.permission.BROADCAST_SMS">
            <intent-filter>
                <action android:name="android.provider.Telephony.SMS_DELIVER" />
            </intent-filter>
        </receiver>

        <!-- BroadcastReceiver that listens for incoming MMS messages -->
        <receiver android:name=".MmsReceiver"
            android:permission="android.permission.BROADCAST_WAP_PUSH">
            <intent-filter>
                <action android:name="android.provider.Telephony.WAP_PUSH_DELIVER" />
                <data android:mimeType="application/vnd.wap.mms-message" />
            </intent-filter>
        </receiver>

        <!-- Activity that allows the user to send new SMS/MMS messages -->
        <activity android:name=".ComposeSmsActivity" >
            <intent-filter>
                <action android:name="android.intent.action.SEND" />
                <action android:name="android.intent.action.SENDTO" />
                <category android:name="android.intent.category.DEFAULT" />
                <category android:name="android.intent.category.BROWSABLE" />
                <data android:scheme="sms" />
                <data android:scheme="smsto" />
                <data android:scheme="mms" />
                <data android:scheme="mmsto" />
            </intent-filter>
        </activity>

        <!-- Service that delivers messages from the phone "quick response" -->
        <service android:name=".HeadlessSmsSendService"
                 android:permission="android.permission.SEND_RESPOND_VIA_MESSAGE"
                 android:exported="true" >
            <intent-filter>
                <action android:name="android.intent.action.RESPOND_VIA_MESSAGE" />
                <category android:name="android.intent.category.DEFAULT" />
                <data android:scheme="sms" />
                <data android:scheme="smsto" />
                <data android:scheme="mms" />
                <data android:scheme="mmsto" />
            </intent-filter>
        </service>
    </application>
</manifest>
```

通过 `Telephony.Sms.getDefaultSmsPackage()`方法来判断自己的应用是否为`Default SMS app`。如果不是，可以通过`startActivity()` 方法启动类似如下的`Dialog`。

```java
public class ComposeSmsActivity extends Activity {

    @Override
    protected void onResume() {
        super.onResume();

        final String myPackageName = getPackageName();
        if (!Telephony.Sms.getDefaultSmsPackage(this).equals(myPackageName)) {
            // App is not default.
            // Show the "not currently set as the default SMS app" interface
            View viewGroup = findViewById(R.id.not_default_app);
            viewGroup.setVisibility(View.VISIBLE);

            // Set up a button that allows the user to change the default SMS app
            Button button = (Button) findViewById(R.id.change_default_app);
            button.setOnClickListener(new View.OnClickListener() {
                public void onClick(View v) {
                    Intent intent =
                            new Intent(Telephony.Sms.Intents.ACTION_CHANGE_DEFAULT);
                    intent.putExtra(Telephony.Sms.Intents.EXTRA_PACKAGE_NAME,
                            myPackageName);
                    startActivity(intent);
                }
            });
        } else {
            // App is the default.
            // Hide the "not currently set as the default SMS app" interface
            View viewGroup = findViewById(R.id.not_default_app);
            viewGroup.setVisibility(View.GONE);
        }
    }
}
```

![](https://dev.tencent.com/u/liqingxue/p/picture_list/git/raw/master/image/2019-7-30-11-41-10.jpg)
