---
title: Android-Glide使用
date: 2016-07-10
tags:
    - 图片加载
categories: Android
---


> 图片加载很是重要,我也对比过别的库,觉得还是Glide好用,我只是简单的分享下我开发用到的相关使用方法

[Glide](https://github.com/bumptech/glide)

如果想深入研究下可以参考[Glide最全解析](https://blog.csdn.net/sinyu890807/column/info/15318)

## Glide的配置

配置很简单，只要在Module的Gradle添加依赖即可

```java
compile 'com.github.bumptech.glide:glide:3.7.0'
compile 'com.android.support:support-v4:25.3.0'
```

<!-- more -->

当然，如果涉及到网络加载图片，记得添加网络权限

```java
<uses-permission android:name="android.permission.INTERNET" />
```

## Glide的使用

### 初始化

Glide支持Activity和Fragment的绑定

```java
Glide.with(Context context);
Glide.with(Activity activity);
Glide.with(FragmentActivity activity);
Glide.with(Fragment fragment);
```

将Activity/Fragment作为with()参数的好处是:图片加载会和Activity/Fragment的生命周期保持一致

### 加载资源

Glide支持网络资源、assets资源、Resources资源、File资源、Uri资源、字节数组

```java
Glide.with(this).load("http://pic9/258/a2.jpg").into(iv);
Glide.with(this).load("file:///xxx.jpg").into(iv);
Glide.with(this).load(R.mipmap.ic_launcher).into(iv);
Glide.with(this).load(file).into(iv);
Glide.with(this).load(uri).into(iv);
Glide.with(this).load(byte[]).into(iv);
```

### 加载gif图片

- 加载静态gif图片(静态就是gif相当于一张图片)

```java
Glide.with(this).load(imageUrl).asBitmap().into(iv);
```

- 加载动态gif图片(gif是动的)

```java
Glide.with(this).load(imageUrl).asGif().into(iv);
```

- 显示本地视频

Glide 还能显示视频！只要他们是存储在手机上的。假设你通过让用户选择一个视频后得到了一个文件路径：

```java
String filePath = "/storage/emulated/0/Pictures/example_video.mp4";
Glide .with( context )
    .load( Uri.fromFile(new File( filePath)))
    .into( iv);
```

这里需要注意的是，这仅仅对本地视频起作用。如果没有存储在该设备上的视频（如一个网络 URL 的视频），它是不工作的!

### 设置加载中和加载失败的图片

- 设置加载中图片

```java
.placeholder(R.drawable.placeholder)
```

- 设置加载失败图片

```java
.error(R.drawable.error)
```

- 设置缩略图支持

```java
//先加载缩略图 然后在加载全图
Glide.with(this)
    .load(imageUrl)
    .thumbnail(0.1f)
    .into(iv);
```

### 设置加载动画

- 默认是淡入淡出动画

```java
Glide.with(this)
    .load("http://nm/photo/1f/1f7a.jpg")
    .crossFade(int duration)//去减慢（或加快）动画
    .into(iv);
```

- 使用 crossFade()

```java
Glide.with(this)
    .load("http://nm/photo/1f/1f7a.jpg")
    .crossFade()//动画默认的持续时间是 300毫秒
    .into(iv);
```

- 添加自定义动画

```java
Glide.with(this)
    .load("http://nm/photo/1f/1f7a.jpg")
    .animate(R.anim.fade_in)
    .into(iv);
```

- 去除动画

```java
Glide.with(this)
    .load("http://nm/photo/1f/1f7a.jpg")
    .dontAnimate()
    .into(iv);
```

### 缩放图像

- CenterCrop 即缩放图像至填充到ImageView内,裁剪额外的部分。ImageView会完全填充，但图像可能不会显示不全。

```java
Glide.with(this)
    .load(url)
    .centerCrop ()
    .into(iv);
```

- fitCenter()   图片会按照imageview长宽中最小的边界作为依据,按比例缩放图像。该图像将会完全显示，但可能不会填满整个 ImageView。

```java
Glide.with(this)
    .load(url)
    .fitCenter()
    .into(iv);
```

### 设置监听回调

```java
Glide.with(this)
    .load(imageUrl)
    .listener(RequestListener listener)
    .into(iv);
```

### 设置加载尺寸

- 指定尺寸(图片大小在xml中不能写死,是wrap_content才可以指定尺寸)

```java
Glide.with(this)
    .load("http://nm/photo/1f/1f7a.jpg")
    .override(600,600)
    .into(iv);
```

### 设置缓存策略

- 设置跳过内存缓存

```java
Glide.with(this)
    .load("http://nm/photo/1f/1f7a.jpg")
    .skipMemoryCache(true)
    .into(iv);
```

- 设置缓存策略

```java
Glide.with(this)
    .load("http://nm/photo/1f/1f7a.jpg")
    .diskCacheStrategy(DiskCacheStrategy.ALL)
    .into(iv);
DiskCacheStrategy.ALL //缓存源资源和转换后的资源
DiskCacheStrategy.NONE//不做任何磁盘缓存
DiskCacheStrategy.RESULT //缓存转换后的资源
DiskCacheStrategy.SOURCE缓存源资源
```

- 清理磁盘缓存

```java
Glide.get(this).clearDiskCache();//在子线程中进行
```

- 清理内存缓存

```java
Glide.get(this).clearMemory();//可以在主线程
```

- 设置磁盘缓存目录和图片效果(默认Bitmap格式是RGB_565)

1. 在AndroidManifest中application节点下:

```java
<!--glide缓存目录设置-->
<meta-data
    android:name="包名.widget.GlideModuleConfig"
    android:value="GlideModule" />
```

2. 创建类GlideModuleConfig

```java
public class GlideModuleConfig implements GlideModule {
    @Override
    public void applyOptions(Context context, GlideBuilder builder) {
        //内部存储/Android/data/包名/cache/glide-images
        builder.setDiskCache(new ExternalCacheDiskCacheFactory(context, "glide-images", 2 * 1024 * 1024));
        //将默认的RGB_565效果转换到ARGB_8888
        builder.setDecodeFormat(DecodeFormat.PREFER_ARGB_8888);
    }

    @Override
    public void registerComponents(Context context, Glide glide) {
        //不做处理
    }
}
```

### BitmapTransformation

Glide在Github上还有一个库，可以处理图片效果，比如裁剪、圆角、高斯模糊等等

- 引入依赖库

```java
compile 'jp.wasabeef:glide-transformations:2.0.1'
```

- 实现高斯模糊

```java
//radius取值1-25,值越大图片越模糊
Glide.with(context)
    .load(url)
    .bitmapTransform(new BlurTransformation(context, radius))
    .into(iv);
```

- 原图基础上变换设置圆形图

```java
Glide.with(context)
    .load(url)
    .bitmapTransform(new CropCircleTransformation(this))
    .into(iv);
```

- 原图基础上变换成圆图 +毛玻璃（高斯模糊）

```java
Glide.with(this)
    .load(url)
    .bitmapTransform(new BlurTransformation(this, 25), new CropCircleTransformation(this))
    .into(iv);
```

- 原图处理成圆角

```java
//如果是四周已经是圆角则RoundedCornersTransformation.CornerType.ALL
Glide.with(this)
    .load(url)
    .bitmapTransform(new RoundedCornersTransformation(this, 30, 0, RoundedCornersTransformation.CornerType.BOTTOM))
    .into(iv);
```

### ViewTarget

用于配合View集成使用

```java
// 写一个自定义View
public class MyLayout extends LinearLayout {
    
    // 定义一个ViewTarget, 注意泛型
    private ViewTarget<MyLayout, GlideDrawable> viewTarget; 

    public MyLayout(Context context, AttributeSet attrs) {
        super(context, attrs);
        
        // 创建并重写代码，其实和SimpleTarget是一样的
        viewTarget = new ViewTarget<MyLayout, GlideDrawable>(this) {
            @Override
            public void onResourceReady(GlideDrawable resource, GlideAnimation glideAnimation) {
                MyLayout myLayout = getView();
                myLayout.setImageAsBackground(resource);
            }
        };
    }

    // 重要的是这个方法，使用Glide的时候需要调用这个方法来得到一个Target
    public ViewTarget<MyLayout, GlideDrawable> getTarget() {
        return viewTarget;
    }

    public void setImageAsBackground(GlideDrawable resource) {
        setBackground(resource);
    }
}
```

使用

```java
Glide.with(this)
    .load(url)
    .into(myLayout.getTarget());
 ```