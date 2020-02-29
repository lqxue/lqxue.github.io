---
title: Android-Studio-NDK-初试
date: 2017-3-9
tags:
  - NDK
categories: Android
---

> 本文主要介绍在AndroidStudio中如何搭建JNI开发环境，并通过输出helloword的方式了解进行JNI开发的步骤。

## NDK

NDK 是 Native Developmentit的缩写，是Google在Android开发中提供的一套用于快速创建native工程的一个工具。使用这个工具可以很方便的编写和调试JNI的代码。

大致意思：NDK是一个工具，可以让你实现你的应用程序使用本地代码的语言，如C和C++的部分。

## JNI

JNI:Java Native Interface它提供了若干的API实现了Java和其他语言的通信（主要是C&C++）。从Java1.1开始，JNI标准成为java平台的一部分，它允许Java代码和其他语言写的代码进行交互。

<!-- more -->

## AndroidStudio 配置

### NDK下载与配置

![](https://raw.githubusercontent.com/lqxue/picture_list/master/image/20200229170527.jpg)

### `build.gradle`配置

![](https://raw.githubusercontent.com/lqxue/picture_list/master/image/20200229171008.png)

```java
android {
    compileSdkVersion 29
    defaultConfig {
        applicationId "com.jni"
        minSdkVersion 19
        targetSdkVersion 29
        versionCode 1
        versionName "1.0"


        ndk{
            //设置只生成armeabi-v7a 平台的so库
            //abiFilters 'armeabi-v7a'
        }

        sourceSets.main {
            jni.srcDirs=["src/main/jni"]
        }
    }

    externalNativeBuild{
        //告诉Gradle在src/main/jni/Android.mk文件中找到根ndk构建脚本
        ndkBuild{
            path "src/main/jni/Android.mk"
        }
    }

    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
}
```

### MainActivity

![](https://raw.githubusercontent.com/lqxue/picture_list/master/image/20200229152443.png)


### 创建jni文件夹

![](https://raw.githubusercontent.com/lqxue/picture_list/master/image/20200229171256.png)


### 编写testJni.c文件

其中方法名一定是Java_包名_类名_方法

```
#include "jni.h"
jstring Java_com_testJni_MainActivity_print(JNIEnv *env, jobject thiz){
    return (*env)->NewStringUTF(env,"hello world");
}
```

### 在jni文件夹新建文件命名为`Android.mk`

Android.mk文件内容

```
LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)
LOCAL_MODULE := print
LOCAL_SRC_FILES := testJni.c
include $(BUILD_SHARED_LIBRARY)
```

### 最后生成`.so`

运行项目,项目如果不报错

![](https://raw.githubusercontent.com/lqxue/picture_list/master/image/20200229171803.jpg)



##  Android系统目前支持的CPU架构：

ARMv5，ARMv7 (从2010年起)
x86 (从2011年起)
MIPS (从2012年起)
ARMv8，MIPS64和x86_64 (从2014年起)

每一个CPU架构对应一个ABI
CPU架构           ABI
ARMv5   --->    armeabi
ARMv7   --->    armeabi-v7a
x86     --->    x86
MIPS    --->    mips
ARMv8   --->    arm64-v8a
MIPS64  --->    mips64
x86_64  --->    x86_64

armeabi：默认选项，将创建以基于ARM* v5TE 的设备为目标的库。 具有这种目标
的浮点运算使用软件浮点运算。 使用此ABI（二进制接口）创建的二进制代码将可以
在所有 ARM*设备上运行。所以armeabi通用性很强。但是速度慢

armeabi-v7a：创建支持基于ARM* v7 的设备的库，并将使用硬件FPU指令。
armeabi-v7a是针对有浮点运算或高级扩展功能的arm v7 cpu。

mips：MIPS是世界上很流行的一种RISC处理器。MIPS的意思是“无内部互锁流水级
的微处理器”(Microprocessor without interlocked piped stages)，其机
制是尽量利用软件办法避免流水线中的数据相关问题。

x86：支持基于硬件的浮点运算的IA-32 指令集。x86是可以兼容armeabi平台运行
的，无论是armeabi-v7a还是armeabi，同时带来的也是性能上的损耗，另外需要
指出的是，打包出的x86的so，总会比armeabi平台的体积更小。

总结
如果项目只包含了 armeabi，那么在所有Android设备都可以运行；
如果项目只包含了 armeabi-v7a，除armeabi架构的设备外都可以运行；
如果项目只包含了 x86，那么armeabi架构和armeabi-v7a的Android设备是无法
运行的；
如果同时包含了 armeabi，armeabi-v7a和x86，所有设备都可以运行，程序在运
行的时候去加载不同平台对应的so，这是较为完美的一种解决方案，同时也会导致
包变大。



[代码github地址](https://github.com/lqxue/JniProject)
