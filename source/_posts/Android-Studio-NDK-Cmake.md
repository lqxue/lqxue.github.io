---
title: Android-Studio-NDK-Cmake
date: 2017-3-10
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

### 在模块根目录添加`CMakeLists.txt`

```java
cmake_minimum_required(VERSION 3.4.1)

# 使用安卓log日志
find_library( # Sets the name of the path variable.
              log-lib
              log )

# 设置引用第三方so的路径  ${CMAKE_SOURCE_DIR}代表和CMakeLists.txt文件同级目录
set(distribution_DIR ${CMAKE_SOURCE_DIR}/libs)
# 导入第三方头文件需要在这指明,比如在jni中添加第三方的头文件
include_directories(src/main/jni)

# 添加和src同级目中libs的so库,其中fmod是so库的名字
add_library( fmod
             SHARED
             IMPORTED )
set_target_properties( fmod
                       PROPERTIES IMPORTED_LOCATION
                       ${distribution_DIR}/${ANDROID_ABI}/libfmod.so )

# 添加和src同级目中libs的so库,其中fmodL是so库的名字
add_library( fmodL
             SHARED
             IMPORTED )
set_target_properties( fmodL
                       PROPERTIES IMPORTED_LOCATION
                       ${distribution_DIR}/${ANDROID_ABI}/libfmodL.so )
add_library( print
             SHARED
        src/main/jni/testJni.c)

# 将所有的so关联,在这关联后还要将下面所有so在Java代码中load一遍
#target_link_libraries( print fmod fmodL
#       ${log-lib} )
target_link_libraries( print
                       ${log-lib} )

```

### `build.gradle`配置

```java
apply plugin: 'com.android.application'

android {
    compileSdkVersion 29
    buildToolsVersion "29.0.2"
    defaultConfig {
        applicationId "com.jniprojectcmake"
        minSdkVersion 21
        targetSdkVersion 29
        versionCode 1
        versionName "1.0"

        ndk{
            //设置只生成armeabi-v7a 平台的so库,不写默认生成所有平台
//            abiFilters 'armeabi-v7a'
        }

    }
    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }
    externalNativeBuild {
        cmake {
            path "CMakeLists.txt"
        }
    }
}

dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])
}

```

### 创建jni文件夹

在src/main文件夹下创建jni文件夹

### 编写工具类调用本地方法

指明需要加载的so名字:print.so,

声明本地方法print

```java
package com.jniprojectcmake;

public class Util {
    static {
        System.loadLibrary("print");
    }

    public  static native String print();
}
```

### 执行`javah`生成头文件

```java
lqxdeMacBook-Pro:java lqx$ pwd
/Users/lqx/Documents/jniprojectcmake/app/src/main/java
lqxdeMacBook-Pro:java lqx$ javah com.jniprojectcmake.Util
```

生成`com_jniprojectcmake_Util.h`头文件

将`com_jniprojectcmake_Util.h`头文件剪切到jni目录中

### 编写testJni.c文件

其中方法名一定是Java_包名_类名_方法

引入上面生成的头文件   复制其中的方法,在`testJni.c`中实现方法


```java
#include "com_jniprojectcmake_Util.h"

JNIEXPORT jstring JNICALL Java_com_jniprojectcmake_Util_print
        (JNIEnv *env, jclass thiz){
    return (*env)->NewStringUTF(env,"hello world");
}
```

### 最后生成`.so`

运行项目,项目如果不报错

![](https://raw.githubusercontent.com/lqxue/picture_list/master/image/20200229223913.png)



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



[代码github地址](https://github.com/lqxue/JniProjectCmake.git)
