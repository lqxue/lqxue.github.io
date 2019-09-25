---
title: Bitmap占用内存大小及加载解析
date: 2017-07-10
tags:
    - Bitmap
categories: Android
---


[文章转自](https://blog.csdn.net/smileiam/article/details/68946182)

## 问题

在讲解图片占用内存前，我们先问自己几个问题：

- 我们在对手机进行屏幕适时，常想可不可以只切一套图适配所有的手机呢？
- 一张图片加载到手机中，占用内存到底有多少？
- 图片占用内存跟哪些东西有关？跟手机有关系么？同一张图片放在不同的dpi文件夹下内存占用会变化么？
- 如果是网络图片，加载到手机中，占用内存跟手机屏幕有关系么？

带着这些问题我们来一层层解析。我们先看看加载本地资源，不同手机所占内存情况：

## 一、加载本地资源，不同手机占内存情况

我们如果加载app内图片，想知道它占用多少内存，可先将此资源转成bitmap进行查看。

### 1. 从资源中获取bitmap

```java
Bitmap bmp = BitmapFactory.decodeResource(getResources(), R.mipmap.testxh);
```

获取到bitmap，我们还需要知道此bitmap在内存占多少空间，具体方法如下。

### 2. 获取图片大小

```java
public int getBitmapSize(Bitmap bitmap){
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT){     //API 19
        return bitmap.getAllocationByteCount();
    } else if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.HONEYCOMB_MR1){//API 12
        return bitmap.getByteCount();
    } else {
        return bitmap.getRowBytes() * bitmap.getHeight(); //earlier version
    }
}
```
接下来就来测试，不同的手机、同一张图片放在不同的密度文件夹下，占用内存情况。

### 3. 同一图片在不同屏幕的手机、不同的屏幕密度文件夹下占用内存大小

1. 经测试同一张图片分别放在不同的mipmap文件夹（mipmap-hdpi, mipmap-xhdpi, mipmap-xxhdpi）下或是drawable文件夹（drawable-hdpi, drawable-xhdpi, drawable-xxhdpi）下，相同的dpi下的文件夹下加载出来的图片，bitmap占用内存大小一样；

2. 对于同一张图片，放在不同手机、不同的屏幕密度文件夹下占用内存情况又是如何呢，这里我们以一张大小为1024*731 = 748544B, 大小为485.11K 的图片为例，下面是测试手机占用的内存情况。

![image.png](https://upload-images.jianshu.io/upload_images/3846387-c52069b0d2e0b6be.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

对于不同的手机屏幕密度的手机占用内存大小
从上表可以看出不同屏幕密度的手机加载图片，如果图片放在与自己屏幕密度相同的文件夹下，占用的内存都是2994176B，与图片本身大小748544B存在一个**4倍关系**，因为图片采用的ARGB-888色彩格式，每个像素点占用4个字节。

从上述测试可以得出，bitmap占用内存大小，与手机的屏幕密度、图片所放文件夹密度、图片的色彩格式有关。

这里总结一下获取Bitmap图片大小的代码：手机在加载图片时，会先查找自己本密度的文夹下是否存在资源，不存在则会向上查找，再向下查找，并对图片进行相应倍数的缩放：

- 如果在与自己屏幕密度相同的文件夹下存在此资源，会原样显示出来，占用内存正好是: 图片的分辨率*色彩格式占用字节数；

- 若自己屏幕密度相同的文件夹下不存在此文件，而在大于自己屏幕密度的文件夹下存在此资源，会进行缩小相应的倍数的平方；

- 若在大于自己屏幕密度的文件夹下没找到此资源，则会向小于自己屏幕密度的文件夹下查找，如果存在，则会进行放大相应的倍数的平方，这两种情况图片占用内存为:
**占用内存=图片宽度 X 图片高度/((资源文件夹密度/手机屏幕密度)^2) * 色彩格式每一个像素占用字节数**

### 4. 图片占用内存与图片的色彩格式的关系
我们在计算bitmap大小时，是通过计算getRowBytes * bitmap.getHeight()得来的，后面的乘数就是图片的高度，而第一个乘数getRowBytes是什么呢？我们根进Bitmap代码查看getRowBytes函数：

```java
/**
 * 返回位图像素的行的字节数，由位图存储的像素值有关，它会根据Color类进行打包
 */
public final int getRowBytes() {
    if (mRecycled) {
        Log.w(TAG, "Called getRowBytes() on a recycle()'d bitmap! This is undefined behavior!");
    }
    return nativeRowBytes(mNativePtr);
}
```

该方法最终调用的是Bitmap中的native方法：

```java
private static native int nativeRowBytes(long nativeBitmap);
```

我们再查看对应的Bitmap.cpp里的nativeRowBytes方法

```java
static jint Bitmap_rowBytes(JNIEnv* env, jobject, jlong bitmapHandle) {
     SkBitmap* bitmap = reinterpret_cast<SkBitmap*>(bitmapHandle)
     return static_cast<jint>(bitmap->rowBytes());
}
```

我们可以看到这里的bitmap形式是以SkBitmap对象展现的，这个Bitmap就和图片展示的色彩格式有关,我们再看看SkBitmap里是怎么计算rowBytes的：

```java
size_t SkBitmap::ComputeRowBytes(Config c, int width) {
    return SkColorTypeMinRowBytes(SkBitmapConfigToColorType(c), width);
}

SkImageInfo.h

static int SkColorTypeBytesPerPixel(SkColorType ct) {
   static const uint8_t gSize[] = {
    0,  // Unknown
    1,  // Alpha_8
    2,  // RGB_565
    2,  // ARGB_4444
    4,  // RGBA_8888
    4,  // BGRA_8888
    1,  // kIndex_8
  };
  SK_COMPILE_ASSERT(SK_ARRAY_COUNT(gSize) == (size_t)(kLastEnum_SkColorType + 1),
                size_mismatch_with_SkColorType_enum);

   SkASSERT((size_t)ct < SK_ARRAY_COUNT(gSize));
   return gSize[ct];
}

static inline size_t SkColorTypeMinRowBytes(SkColorType ct, int width) {
    return width * SkColorTypeBytesPerPixel(ct);
}
```

可以看到，图片的宽乘以了一个SkColorTypeBytesPerPixel(ct)变量，对于不同色彩格式，每个像素占用的字节数就是在SkColorTypeBytesPerPixel中定义的。这就是为什么上面得出的bitmap大小，在自己屏幕密度的文件夹下图片占用的内存大小都被乘以了**4**,因为bitmap加载默认采用的是RGBA_8888编码格式。

### 5. 图片占用内存与手机屏幕密度、图片所在文件夹密度的关系

那么手机怎么加载图片时，为什么同样的图片在不同的屏幕分辨率的手机上、不同的屏幕密度文件夹下占用内存会相差这么大呢？

在加载资源图片时，我们一般会借助于BitmapFactory的decodeResource方法，此方法的源代码如下：

```java
 /**
 * @param res   包含图片资源的Resources对象，一般通过getResources()即可获取
 * @param id    资源文件id, 如R.mipmap.ic_laucher
 * @param opts  可为空，控制采样或图片是否需要完全解码还是只需要获取图片大小
 * @return      解码的bitmap
 */
public static Bitmap decodeResource(Resources res, int id, Options opts) {
    Bitmap bm = null;
    InputStream is = null;

    try {
        final TypedValue value = new TypedValue();
        //1.读取资源返回数据流格式，最终会调用AssetManager的openNonAsset方法进行读取资源
        is = res.openRawResource(id, value);
        //2. 根据数据流格式进行解码，在直接加载res资源时，一般opts为空
        bm = decodeResourceStream(res, value, is, null, opts);
    } catch (Exception e) {

    } finally {
        try {
            if (is != null) is.close();
        } catch (IOException e) {
            // Ignore
        }
    }

    if (bm == null && opts != null && opts.inBitmap != null) {
        throw new IllegalArgumentException("Problem decoding into existing bitmap");
    }

    return bm;
}
 ```

我们再来看看BitmapFactory的decodeResourceStream方法

```java
/**
 * 根据输入的数据流确码成一个新的bitmap, 数据流是从资源处获取，在这里可以根据规则对图片进行一些缩放操作
 */
public static Bitmap decodeResourceStream(Resources res, TypedValue value,
        InputStream is, Rect pad, Options opts) {

    if (opts == null) {//如果没有设置Options，系统会新创建一个Options对象
        opts = new Options();
    }
    //若没有设置opts，inDensity就是初始值0,它代表图片资源密度
    if (opts.inDensity == 0 && value != null) {
        final int density = value.density;
        if (density == TypedValue.DENSITY_DEFAULT) { //如果density等于0,则采用默认值160
            opts.inDensity = DisplayMetrics.DENSITY_DEFAULT;
        } else if (density != TypedValue.DENSITY_NONE) {//如果没有设置资源密度，则图片不会被缩放
            opts.inDensity = density;//这里density的值对应的就是资源密度值
        }
    }
    //此时inTargetDensity默认也为0
    if (opts.inTargetDensity == 0 && res != null) {
        //将手机的屏幕密度值赋值给最终图片显示的密度
        opts.inTargetDensity = res.getDisplayMetrics().densityDpi;
    }

    return decodeStream(is, pad, opts);
}
```

可以看到这里调用了native decodeStream方法：

```java
static jobject doDecode(JNIEnv* env, SkStreamRewindable* stream, jobject padding, jobject options) {

......
    if (env->GetBooleanField(options, gOptions_scaledFieldID)) {
        //资源本身的密度
        const int density = env->GetIntField(options, gOptions_densityFieldID);
        //最终加载的图片的密度
        const int targetDensity = env->GetIntField(options, gOptions_targetDensityFieldID);
        //手机的屏幕密度
        const int screenDensity = env->GetIntField(options, gOptions_screenDensityFieldID);
        //如果资源密度不为0，手机屏幕密度也不为0, 资源的密度与屏幕密度不相等时，图片缩放比例=屏幕密度/资源密度，如对于三星手机屏幕密度为640,如果图片放在文件夹为xhdpi 320下，则scale=2,会对图片长宽均放大2倍
        if (density != 0 && targetDensity != 0 && density != screenDensity) {
            scale = (float) targetDensity / density;
        }
    }
}

const bool willScale = scale != 1.0f;//判断是否需要缩放
......
SkBitmap decodingBitmap;
if (!decoder->decode(stream, &decodingBitmap, prefColorType,decodeMode)) {
   return nullObjectReturn("decoder->decode returned false");
}
//这里这个deodingBitmap就是解码出来的bitmap，大小是图片原始的大小
int scaledWidth = decodingBitmap.width();
int scaledHeight = decodingBitmap.height();
if (willScale && decodeMode != SkImageDecoder::kDecodeBounds_Mode) {
    scaledWidth = int(scaledWidth * scale + 0.5f);//这里+0.5是保证在图片缩小时，可能会出小数，这里加0.5是为了让除后的数向上取整
    scaledHeight = int(scaledHeight * scale + 0.5f);
}
if (willScale) {
    const float sx = scaledWidth / float(decodingBitmap.width());
    const float sy = scaledHeight / float(decodingBitmap.height());

    // 设置解码图片的colorType
    SkColorType colorType = colorTypeForScaledOutput(decodingBitmap.colorType());
     //设置图片的宽高
    outputBitmap->setInfo(SkImageInfo::Make(scaledWidth, scaledHeight,
            colorType, decodingBitmap.alphaType()));
    if (!outputBitmap->allocPixels(outputAllocator, NULL)) {
        return nullObjectReturn("allocation failed for scaled bitmap");
    }

    if (outputAllocator != &javaAllocator) {
        outputBitmap->eraseColor(0);
    }

    SkPaint paint;
    paint.setFilterLevel(SkPaint::kLow_FilterLevel);

    SkCanvas canvas(*outputBitmap);
    canvas.scale(sx, sy);//根据缩放比画出图像
    canvas.drawBitmap(decodingBitmap, 0.0f, 0.0f, &paint);//将图片画到画布上
}
......
}
```

**inDensity，inTargetDensity，inScreenDensity, inScaled三者关系**

通过追查代码，我们可以看到图片资源通过数据流解码时，会根据inDensity，inTargetDensity，inScreenDensity三个值和是否被缩放标识inScaled

- inDensity：图片本身的像素密度（其实就是图片资源所在的哪个密度文件夹下，如在xxhdpi下就是480，如果在asstes、手机内存／sd卡下，默认是160）；
- inTargetDensity：图片最终在bitmap里的像素密度，如果没有赋值，会将inTargetDensity设置成inScreenDensity；
- inScreenDensity：手机本身的屏幕密度，如我们测试的三星手机dpi=640, 如果inDensity与inTargetDensity不相等时，就需要对图片进行缩放，inScaled = inTargetDensity／inDensity。

我们上面研究了加载应用程序的图片占用内存大小与手机屏幕密码和图片所放的密度文件夹、图片的编码格式有关，那如果加载的是网络图片或是本地图片，在不同的手机上占用内存又是否一样呢？

## 二、加载sd卡下的资源或是网络图片解析

手机无论是加载sd卡图片，assets路径下还是网络图片，都需要先把图片读成数据流格式，再调用相应的decodeStream方法，将数据流转成bitmap形式，在调用decodeStream如果不设置Options的话，通过以上三款手机打印出图片所占内存大小均为：2994176B，也就是跟手机的屏幕密度没有关系。

那如果设置Options中的参数，图片占用的内存会不会与手机的屏幕密度有关系呢？我在测试中发现单独手动设置图片密度inDensity或是inTargetDensity，并不起作用，图片占用内存一直都是图片本身大小。

为什么没起作用呢，这需要我们从资源加载的源头看起。

### 1. 根据手机本地图片路径获取Bitmap
我们先来看一下BitmapFactory的decodeFile函数：

```java
//读取手机本地的图片资源
public static Bitmap decodeFile(String pathName, Options opts) {
        Bitmap bm = null;
        InputStream stream = null;
        try {
            stream = new FileInputStream(pathName);
            //调用decodeStream将数据流转成bitmap
            bm = decodeStream(stream, null, opts);
        } catch (Exception e) {
            /*  do nothing.
                If the exception happened on open, bm will be null.
            */
            Log.e("BitmapFactory", "Unable to decode stream: " + e);
        } finally {
            if (stream != null) {
                try {
                    stream.close();
                } catch (IOException e) {
                    // do nothing here
                }
            }
        }
        return bm;
    }
```

### 2. 根据网络地址获取图片Bitmap

```java
/**
 * 从网络中获取图片，先获取数据流，再转成Bitmap
 * @return
 */
public Bitmap getBitmapByPicUrl(String picurl) throws IOException {
    InputStream inputStream = null;
    URL url = new URL(picurl);                    //服务器地址
    if (url != null) {
        //打开连接
        HttpURLConnection httpURLConnection = (HttpURLConnection)url.openConnection();
        httpURLConnection.setConnectTimeout(3000);//设置网络连接超时的时间为3秒
        httpURLConnection.setRequestMethod("GET");        //设置请求方法为GET
        httpURLConnection.setDoInput(true);                //打开输入流
        int responseCode = httpURLConnection.getResponseCode();    // 获取服务器响应值
        if (responseCode == HttpURLConnection.HTTP_OK) {        //正常连接
            inputStream = httpURLConnection.getInputStream();        //获取输入流
        }
    }
    return BitmapFactory.decodeStream(inputStream);
}
```

可以看到通过路径加载图片，最终还是会调用BitmapFactory里的decodeStream方法，我们再来看看decodeStream方法。

### 3. 将数据流转成Bitmap

```java
/**
 *根据输入的数据流确码成一个新的bitmap
 *
 * @param is           从源数据获取的输入数居流，用于解码成bitmap
 * @param outPadding   如果不为空，返回bitmap的边距，这个会加入到图片所占内存大小里
 * @param opts         可以为空; 用来控制图片的采样率和图片是否需要完全解码，还是只需要获取图片大小
 * @return             解码后的图片
 */
public static Bitmap decodeStream(InputStream is, Rect outPadding, Options opts) {
    if (is == null) {
        return null;
    }

    Bitmap bm = null;

    Trace.traceBegin(Trace.TRACE_TAG_GRAPHICS, "decodeBitmap");
    try {//如果数据流来自资源，则直接调用native方法
        if (is instanceof AssetManager.AssetInputStream) {
            final long asset = ((AssetManager.AssetInputStream) is).getNativeAsset();
            bm = nativeDecodeAsset(asset, outPadding, opts);
        } else {
            bm = decodeStreamInternal(is, outPadding, opts);
        }

        if (bm == null && opts != null && opts.inBitmap != null) {
            throw new IllegalArgumentException("Problem decoding into existing bitmap");
        }

        setDensityFromOptions(bm, opts);
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_GRAPHICS);
    }

    return bm;
}
```

如果数据流来自于资源，则调用BitmapFactory的nativeDecodeAsset，

```java
private static native Bitmap nativeDecodeAsset(long nativeAsset, Rect padding, Options opts);
```

否则调用decodeStreamInternal方法：

```java
 /**
 * is不得为空，会根据流的需要提供一个缓冲区
 */
private static Bitmap decodeStreamInternal(InputStream is, Rect outPadding, Options opts) {
    // ASSERT(is != null);
    byte [] tempStorage = null;
    if (opts != null) tempStorage = opts.inTempStorage;
    //如果Options没有提供inTempStorage参数会默认提供一个16M的缓冲区
    if (tempStorage == null) tempStorage = new byte[DECODE_BUFFER_SIZE];
    return nativeDecodeStream(is, tempStorage, outPadding, opts);
}
```

此方法会调用native的nativeDecodeStream方法：

### 4. native层的数据流解析
1. nativeDecodeStream／nativeDecodeAsset
通过追踪上述两种nativeDecodeStream方法和nativeDecodeAsset方法，它们最终都会调用nativeDecodeStreamScaled或是nativeDecodeAssetScaled方法，它们会添加两个参数，一个是false,一个是1.0f，这两个参数具体代表什么呢？

```java
//解码Asset资源的数据流
static jobject nativeDecodeAsset(JNIEnv* env, jobject clazz, jint native_asset,
        jobject padding, jobject options) {
    return nativeDecodeAssetScaled(env, clazz, native_asset, padding, options, false, 1.0f);
}
//解码纯数据流
static jobject nativeDecodeStream(JNIEnv* env, jobject clazz, jobject is, jbyteArray storage,
        jobject padding, jobject options) {
    return nativeDecodeStreamScaled(env, clazz, is, storage, padding, options, false, 1.0f);
}
```

2. nativeDecodeStreamScaled／nativeDecodeAssetScaled<br/>
nativeDecodeAssetScaled或是nativeDecodeStreamScaled方法中最后两个参数，分别是applyScale，sclae，一个是是否申请缩放，一个是缩放比例，也就是从这种数据流加载的图片，默认都不会进缩放。我们注意到，这两个函数最终都会走到doDecode方法里，我们直接看nativeDecodeStreamScaled方法，发现此方法只是对输入流进行了转换，转成SkStream类型。

```java
static jobject nativeDecodeStreamScaled(JNIEnv* env, jobject clazz, jobject is, jbyteArray storage,
        jobject padding, jobject options, jboolean applyScale, jfloat scale) {
    jobject bitmap = NULL;
    SkStream* stream = CreateJavaInputStreamAdaptor(env, is, storage, 0);
    if (stream) {
        // for now we don't allow purgeable with java inputstreams
        bitmap = doDecode(env, stream, padding, options, false, false, applyScale, scale);
        stream->unref();
    }
    return bitmap;
}
```
3. doDecode
我们来看最终的doDecode函数：

```java
static jobject doDecode(JNIEnv* env, SkStream* stream, jobject padding,
        jobject options, bool allowPurgeable, bool forcePurgeable = false,
        bool applyScale = false, float scale = 1.0f) {
    int sampleSize = 1;
    SkImageDecoder::Mode mode = SkImageDecoder::kDecodePixels_Mode;
    SkBitmap::Config prefConfig = SkBitmap::kARGB_8888_Config;//直接采用ARGB_8888的色彩格式
    bool doDither = true;
    bool isMutable = false;
    bool willScale = applyScale && scale != 1.0f;//上面传的参数applyScale为false，所以willScale为false
    bool isPurgeable = !willScale &&
            (forcePurgeable || (allowPurgeable && optionsPurgeable(env, options)));
    bool preferQualityOverSpeed = false;
    jobject javaBitmap = NULL;
    if (options != NULL) {
        sampleSize = env->GetIntField(options, gOptions_sampleSizeFieldID);//获取采样率
        if (optionsJustBounds(env, options)) {//是否只加载图片边界，而不解码
            mode = SkImageDecoder::kDecodeBounds_Mode;
        }
       //省略初始化代码
    }
    //省略一堆代码
    SkImageDecoder* decoder = SkImageDecoder::Factory(stream);
    int scaledWidth = decoded->width();//获取解码图片的宽度
    int scaledHeight = decoded->height();//获取解码后图片的调节度
    if (willScale && mode != SkImageDecoder::kDecodeBounds_Mode) {//由于willScale为false，这里不会运行
        scaledWidth = int(scaledWidth * scale + 0.5f);
        scaledHeight = int(scaledHeight * scale + 0.5f);
    }
    // update options (if any)
    if (options != NULL) {
        env->SetIntField(options, gOptions_widthFieldID, scaledWidth);
        env->SetIntField(options, gOptions_heightFieldID, scaledHeight);
        env->SetObjectField(options, gOptions_mimeFieldID,
                getMimeTypeString(env, decoder->getFormat()));
    }
    if (mode == SkImageDecoder::kDecodeBounds_Mode) {//如果只获取图片大小，这里不会返回bitmap
        return NULL;
    }

    if (padding) {//如果设置了padding，则会把边矩算进去
        if (peeker.fPatchIsValid) {
            GraphicsJNI::set_jrect(env, padding,
                    peeker.fPatch->paddingLeft, peeker.fPatch->paddingTop,
                    peeker.fPatch->paddingRight, peeker.fPatch->paddingBottom);
        } else {
            GraphicsJNI::set_jrect(env, padding, -1, -1, -1, -1);
        }
    }
    SkPixelRef* pr;
    if (isPurgeable) {
        pr = installPixelRef(bitmap, stream, sampleSize, doDither);
    } else {
        pr = bitmap->pixelRef();
    }


    return GraphicsJNI::createBitmap(env, bitmap, javaAllocator.getStorageObj(),
            isMutable, ninePatchChunk);//创建bitmap
}
```
通过上面分析直至native中的decode函数，我们发现options里的参数只提取了sampleSize、optionsJustBounds，但是没有见到inDensity，inTargetDensity，inScreenDensity等参数的提取。如果我在加载流前，设置ops.inDensity和ops.inTargetDensity参数如下，图片占用内存大小会缩小到原来的1/4

```java
BitmapFactory.Options ops = new BitmapFactory.Options();
            int targetDensity = getResources().getDisplayMetrics().densityDpi;
            ops.inDensity = 240;
            ops.inTargetDensity = 480;
            Bitmap assetsbmp = BitmapFactory.decodeStream(stream, null, ops);
```

但是如果只设置inDensity或是inTargetDensity参数，是完全不起作用，感觉是因为只设置了一个参数，另一个参数默认为0, 前面咱们判断过，只要有一个参数为0, 就不会计算缩放比。所以默认还是显示原来图片尺寸大小，只有两个参数均设置，都不为0, 才会去计算缩放比。

通过上面的分析，我们可以回答最开始的问题了。

## 结论：

**1.** 在对手机进行屏幕适时，可以只切一套图适配所有的手机。

- 但是如果只切一套小图，那在高屏幕密度手机上，会对图片进行放大，这样图片占用的内存往往比切相应图片放在高密度文件夹下，占用的内存还要大。

- 那如果只切一套大图放在高幕文件夹下，在小屏幕密度手机上，会缩小显示，按道理是行得通的。但系统在对图片进行缩放时，会进行大量计算，会对手机的性能有一定的影响。同时如果图片缩放比较狠，可能导致图片出现抖动或是毛边。

- 所以最好切出不同比便的图片放在不同幕度的文件夹下，对于性能要求不大高的图片，可以只切一套大图；

**2.** 一张图片**占用内存=图片长 * 图片宽 ／ （资源图片文件密度/手机屏幕密度）^2 * 每一象素占用字节数**，所以图片占用内存跟图片本身大小、手机屏幕密度、图片所在的文件夹密度，图片编码的色彩格式有关；

**3.** 对于网络图片，在不同屏幕密度的手机上加载出来，占用内存是一样的。

**4.** 对于网络或是assets/手机本地图片加载，如果想通过设置Options里的
inDensity或是inTargetDensity参数来调整图片的缩放比，必须两个参数均设置才能起作用，只设置一个，不会起作用。

**5.** drawable和mipmap文件夹存放图片的区别，首先图片放在drawable-xhdpi和mipmap-xhdpi下，两者占用的内存是一样的，
Mipmaps早在Android2.2+就可以用了，但是直到4.3 google才强烈建议使用。把图片放到mipmaps可以提高系统渲染图片的速度，提高图片质量，减少GPU压力。其他并没有什么区别。