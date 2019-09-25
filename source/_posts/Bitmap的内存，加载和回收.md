---
title: Bitmap的内存，加载和回收
date: 2017-07-11
tags:
    - Bitmap
categories: Android
---


## 概述

如何高效地加载Bitmap?其实核心思想很简单，那就是采用BitmapFactory.Options加载所需尺寸的图片。有时候我们用ImageView加载图片，图片的原始尺寸远远大于ImageView。这个时候把图片完全加载进来没有必要，因为ImageView也显示不出来原始的图片。

我们可以使用BitmapFactory.Options对图片进行预加载，然后对图片进行压缩，将缩小后的图片放在ImageView中展示。这样提高了Bitmap加载的性能，一定程度上避免了OOM。

## Bitmap加载图片

Bitmap的加载离不开BitmapFactory类，关于Bitmap官方介绍：

`Creates Bitmap objects from various sources, including files, streams, and byte-arrays.`

BitmapFactory类提供了四类方法用来加载Bitmap：

1. decodeFile()，从文件系统加载。
2. decodeResource()，资源文件中加载。
3. decodeStream()，从输入流加载。
4. decodeByteArray()，从字节数组中加载。

注意：查看源码可以发现，decodeFile()和decodeResource()间接调用decodeStream()。

## Bitmap的内存位置

在Android3.0之前：Bitmap的像素数据存放在Native内存，而Bitmap对象本身则存放在Dalvik Heap中。

从Android3.0开始：Bitmap的内存就全部在Dalvik Heap里了 。

## Bitmap的内存回收

在Android3.0之前，需要使用Bitmap.recycle()进行Bitmap的内存回收。

从Android3.0开始，不需要手动回收Bitmap了。

## Bitmap的内存复用

从Android3.0开始，在Bitmap中引入了一个新的字段BitmapFactory.Options.inBitmap，设置此字段为true后，解码方法会尝试复用一张存在的Bitmap。这意味着Bitmap的内存被复用，避免了内存的回收及申请过程，显然性能表现更佳。

Android4.4(API 19)之前只有格式为jpg、png，同等宽高（要求苛刻），inSampleSize为1的Bitmap才可以复用。

从Android4.4(API 19)开始被复用的Bitmap的内存大于需要新申请内存的Bitmap的内存就可以了。

## 使用缓存

### LruCache+DiskLruCache

出于对性能和app的考虑，我们肯定是想着第一次从网络中加载到图片之后，能够将图片缓存在内存和sd卡中，这样，我们就不用频繁的去网络中加载图片，可以很好地控制内存问题。

一般都会考虑使用LruCache+DiskLruCache，LruCache作为Bitmap在内存中的存放容器，在sd卡则使用DiskLruCache来统一管理磁盘上的图片缓存。

### SoftReference+inBitmap

之前提到，可以采用LruCache作为存放Bitmap的容器，而在LruCache中有一个方法值得留意，那就是entryRemoved()，按照文档给出的说法，在LruCache容器满了需要淘汰存放其中的对象腾出空间的时候会调用此方法。

注意：这里只是对象被淘汰出LruCache容器，但并不意味着对象的内存会立即被Dalvik虚拟机回收掉。
此时可以在此方法中将Bitmap使用SoftReference包裹起来，并用事先准备好的一个HashSet容器来存放这些即将被回收的Bitmap，有人会问，这样存放有什么意义？

之前我们提到将inmutable设置为true，Bitmap的内存可以被复用，当然肯定要满足之前所说的条件。

解码方法对图片进行decode的时候会检查内存中是否有可复用的Bitmap，避免我们频繁地去SD卡上加载图片而造成系统性能的下降，毕竟从直接从内存中复用要比在SD卡上进行IO操作的效率要高很多。

## Bitmap的像素格式

1. ALPHA_8：颜色信息只由透明度组成，占8位。
2. ARGB_4444：颜色信息由透明度与R（Red），G（Green），B（Blue）四部分组成，每个部分都占4位，总共占16位。
3. ARGB_8888：颜色信息由透明度与R（Red），G（Green），B（Blue）四部分组成，每个部分都占8位，总共占32位。是Bitmap默认的颜色配置信息，也是最占空间的一种配置。
4. RGB_565：颜色信息由R（Red），G（Green），B（Blue）三部分组成，R占5位，G占6位，B占5位，总共占16位。

通常我们优化Bitmap时，当需要做性能优化或者防止OOM，我们通常会使用RGB_565，因为ALPHA_8只有透明度，显示一般图片没有意义，Bitmap.Config.ARGB_4444显示图片不清楚，Bitmap.Config.ARGB_8888占用内存最多。

## Bitmap的内存计算

Bitmap类中有一个方法getByteCount()：

```java
/**
 * Returns the minimum number of bytes that can be used to store this bitmap's pixels.
 *
 * <p>As of {@link android.os.Build.VERSION_CODES#KITKAT}, the result of this method can
 * no longer be used to determine memory usage of a bitmap. See {@link
 * #getAllocationByteCount()}.</p>
 */
public final int getByteCount() {
    // int result permits bitmaps up to 46,340 x 46,340
    return getRowBytes() * getHeight();
}
```

还有一个方法getAllocationByteCount()：

```java
/**
 * Returns the size of the allocated memory used to store this bitmap's pixels.
 *
 * <p>This can be larger than the result of {@link #getByteCount()} if a bitmap is reused to
 * decode other bitmaps of smaller size, or by manual reconfiguration. See {@link
 * #reconfigure(int, int, Config)}, {@link #setWidth(int)}, {@link #setHeight(int)}, {@link
 * #setConfig(Bitmap.Config)}, and {@link BitmapFactory.Options#inBitmap
 * BitmapFactory.Options.inBitmap}. If a bitmap is not modified in this way, this value will be
 * the same as that returned by {@link #getByteCount()}.</p>
 *
 * <p>This value will not change over the lifetime of a Bitmap.</p>
 *
 * @see #reconfigure(int, int, Config)
 */
public final int getAllocationByteCount() {
    if (mBuffer == null) {
        // native backed bitmaps don't support reconfiguration,
        // so alloc size is always content size
        return getByteCount();
    }
    return mBuffer.length;
}
```

通过方法注释我们可以了解到，**getByteCount()代表存储Bitmap的色素需要的最少内存，而getAllocationByteCount()代表在内存中为Bitmap分配的内存大小**。

其实getByteCount()方法是在API12加入的，代表存储Bitmap的色素需要的最少内存。**从API19开始getAllocationByteCount()方法代替了getByteCount()。**

一般情况下getByteCount()和getAllocationByteCount()是相等的。但是**Bitmap内存如果复用之后，两者就不一样了。**

通过复用Bitmap来解码图片，如果被复用的Bitmap的内存比待分配内存的Bitmap大，那么getByteCount()表示新解码图片占用内存的大小（并非实际内存大小，实际大小是复用的那个Bitmap的大小），**getAllocationByteCount()表示被复用Bitmap真实占用的内存大小。**

那getByteCount()和getAllocationByteCount()值是怎么计算出来的呢？

## 例子

下面就来举个例子计算理论上Bitmap加载一张图片时，所占内存的大小，和getByteCount()的结果比较一下。

## 假设

一张像素为522*686的PNG图片，把它放到drawable-xxhdpi目录下，在三星s6上加载，getByteCount()的结果是2547360B。

## 推导

### 第一步

默认的像素格式是ARGB_8888，之前已经说到了ARGB_8888格式下的一个像素点占用32位内存即4个字节。

所以结果是：`int res = 522*686*4`，**1432368**。

显然和答案不一样啊。

### 第二步

假设中说把图片放到drawable-xxhdpi目录下，在三星s6上加载。并不是随口一说的，它们也是影响Bitmap所占内存大小的重要因素。

我们读取的是drawable目录下面的图片，用的是decodeResource方法，该方法本质上就两步：

读取原始资源，这个调用了Resource.openRawResource方法，这个方法调用完成之后会对TypedValue进行赋值，其中包含了原始资源的density等信息。

调用decodeResourceStream对原始资源进行解码和适配。这个过程实际上就是原始资源的density到屏幕density的一个映射。

原始资源的density其实取决于资源存放的目录（比如xxhdpi对应的是480），而屏幕density的值是和设备的硬件有关的，三星s6的值为640。加载时，原始的资源会自动进行缩放。

所以结果是：`int res = (522 * 640 / 480) * (686 * 640 / 480) * 4` ，**2546432**。

### 第三步

好像还是差那么一点，其实系统是进行了精度处理。

所以最终结果是：`int res = (522 * 640 / 480f + 0.5) * (686 * 640 / 480f + 0.5) * 4`，**2547360**。

### inScaled

上面说的缩放和一个参数inScaled有关：

```java
public static class Options {
    /**
     * Create a default Options object, which if left unchanged will give
     * the same result from the decoder as if null were passed.
     */
    public Options() {
        inDither = false;
        inScaled = true;
        inPremultiplied = true;
    }
    ……
}
```

如果inScaled设置为true，就缩放。设置为false，则不进行缩放。默认的值从上面代码可以看到，就是true。

缩放的比例为inTargetDensity / inDensity。

但是缩放也只针对资源文件有效，对于其他来源的图片不起效果，我们可以从源码中参数上的解释得知：

    /**
     * The pixel density to use for the bitmap.  This will always result
     * in the returned bitmap having a density set for it (see
     * {@link Bitmap#setDensity(int) Bitmap.setDensity(int)}).  In addition,
     * if {@link #inScaled} is set (which it is by default} and this
     * density does not match {@link #inTargetDensity}, then the bitmap
     * will be scaled to the target density before being returned.
     *
     * <p>If this is 0,
     * {@link BitmapFactory#decodeResource(Resources, int)},
     * {@link BitmapFactory#decodeResource(Resources, int, android.graphics.BitmapFactory.Options)},
     * and {@link BitmapFactory#decodeResourceStream}
     * will fill in the density associated with the resource.  The other
     * functions will leave it as-is and no density will be applied.
     *
     * @see #inTargetDensity
     * @see #inScreenDensity
     * @see #inScaled
     * @see Bitmap#setDensity(int)
     * @see android.util.DisplayMetrics#densityDpi
     */
    public int inDensity;

其中有一句`The other functions will leave it as-is and no density will be applied`就是这个意思。

以上所说inDensity和inTargetDensity其实是DPI(dots per inch)，[关于DPI的概念请移步 全面理解Android中的Px，DPI，DIP，Density，Sp等概念。](https://blog.csdn.net/sted_zxz/article/details/78273633)

## 结论

**Bitmap加载资源文件**在内存当中占用的大小取决于以下三点：

- 像素格式，前面我们已经提到，如果是ARGB8888那么就是一个像素4个字节，如果是RGB565那就是2个字节。
- 原始文件存放的资源目录（是hdpi还是xxhdpi）
- 目标屏幕的DPI（同等条件下，红米在资源方面消耗的内存肯定是要小于三星S6的）
Bitmap加载其他来源的图片，就和像素格式有关。

## 减少Bitmap内存占用

### 合理选择Bitmap的像素格式

不需要透明度的情况下，我们通常使用RGB_565。

### 使用采样

inSampleSize的值必须大于1时才会有效果，且采样率同时作用于宽和高。当inSampleSize=1时，采样后的图片为图片的原始大小。

当inSampleSize=n时，采样后的图片的宽高均为原始图片宽高的1/n，这时像素为原始图片的1/(n*n)，占用内存也为原始图片的1/(n*n)。

**inSampleSize的取值应该总为2的整数倍**，否则会向下取整，取一个最接近2的整数倍，比如inSampleSize=3时，系统会取inSampleSize=2。

假设一张`1024*1024`，模式为ARGB_8888的图片，inSampleSize=2，原始占用内存大小是4MB，采样后的图片占用内存大小就是`(1024/2) * (1024/2 )* 4 = 1MB`。

### inSampleSize

下面我们来介绍inSampleSize这个参数，当这个参数为1时，采样后的图片大小和原来一样；当这个参数为2时，采样后的图片宽高均为原来的1/2，大小也就成了原来的1/4。也就是说，采样后的大小等于原始大小除以采样率的平方。

官方文档规定，inSampleSize的值应为2的非负整数次幂（1，2，4，… ），否则会被系统向下取整并找到一个最接近的值。
通过设置inSampleSize我们就能够将图片缩放到一个合理的大小，那么该如何设置inSampleSize的值呢？

在讲解这个之前，我们先来考虑以下情况：我们的ImageView的大小为100 * 100，要显示的图片大小为300 * 400，此时我们应该将inSampleSize设为多少呢？

首先我们通过计算可以得到图片宽是ImageView的3倍，而图片高是ImageView的4倍。那么我们应该将图片宽高缩小为原来的4倍吗？假如我们把图片宽高都变为原来的1/4，那么现在图片大小为75 * 100，ImageView大小为100 * 100，图片要显示在ImageView中需要进行拉伸，而拉伸的话可能会导致图片失真。所以我们应该把图片宽高变为原来的1/3，以保证它不小于ImageView的大小，这样尽管多占用一些内存，但不会造成图片质量的下降，这还是很有必要的。

通过以上分析，我们知道了在设置inSampleSize时应该注意使得**缩放后的图片大小不小于相应的ImageView大小。**

### 计算inSampleSize的步骤

1. 获取图片的原始宽高，通过将Options的inJustDecodeBounds属性设为true后调用decodeResource方法，可以实现不真正加载图片而只是获取图片的尺寸信息

```java
public static Bitmap decodeSampledBitmapFromResource(Resources res, int resId,
                                                     int reqWidth, int reqHeight) {

    final BitmapFactory.Options options = new BitmapFactory.Options();
    options.inJustDecodeBounds = true;
    BitmapFactory.decodeResource(res, resId, options);

    options.inSampleSize = calculateInSampleSize(options, reqWidth, reqHeight);

    options.inJustDecodeBounds = false;
    return BitmapFactory.decodeResource(res, resId, options);
}
```

2. 根据原始宽高计算出inSampleSize

```java
public static int calculateInSampleSize(
        BitmapFactory.Options options, int reqWidth, int reqHeight) {
    // Raw height and width of image
    final int height = options.outHeight;
    final int width = options.outWidth;
    int inSampleSize = 1;

    if (height > reqHeight || width > reqWidth) {

        final int halfHeight = height / 2;
        final int halfWidth = width / 2;
        while ((halfHeight / inSampleSize) >= reqHeight
                && (halfWidth / inSampleSize) >= reqWidth) {
            inSampleSize *= 2;
        }
    }
    return inSampleSize;
}
```

## 图片的质量压缩

上述用inSampleSize压缩是尺寸压缩，Android中还有一种压缩方式叫质量压缩。质量压缩是在保持像素的前提下改变图片的位深及透明度等，来达到压缩图片的目的，经过它压缩的图片文件大小(kb)会有改变，但是导入成Bitmap后占得内存是不变的，宽高也不会改变。因为要保持像素不变，所以它就无法无限压缩，到达一个值之后就不会继续变小了。显然这个方法并不适用与缩略图，其实也不适用于想通过压缩图片减少内存的适用，仅仅适用于想在保证图片质量的同时减少文件大小的情况而已。

## 使用矩阵

我们之前使用inSampleSize对图片进行采样，采样之后内存是小了，可是图的尺寸也小了，我们要用Canvas绘制原始大小的图片该怎么办？就可以使用矩阵：

```java
Matrix matrix = new Matrix();
matrix.preScale(2, 2, 0, 0);
canvas.drawBitmap(bitmap, matrix, paint);
```

这样，绘制出来的图就是放大以后的效果了，不过占用的内存却仍然是我们采样出来的大小。

如果我要把图片放到ImageView当中呢？一样可以，请看：

```java
Matrix matrix = new Matrix();
matrix.postScale(2, 2, 0, 0);
imageView.setImageMatrix(matrix);
imageView.setScaleType(ScaleType.MATRIX);
imageView.setImageBitmap(bitmap);
```

## 参考：

[1.Android坑档案：你的Bitmap究竟占多大内存？](https://zhuanlan.zhihu.com/p/20732309)
[2.Android性能优化：谈谈Bitmap的内存管理与优化](http://www.mamicode.com/info-detail-516558.html)
[3.Android 之Bitmap](https://www.jianshu.com/p/98c88f9ceafa)
[4.Android性能优化（五）之细说Bitmap](https://www.jianshu.com/p/e49ec7d053b3)
[5.softReference+LruCache优化Android缓存](https://www.jianshu.com/p/7eaa037e03c7)
[6.玩转Android Bitmap](https://www.jianshu.com/p/3950665e93e6)