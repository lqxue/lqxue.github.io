---
title: 随着RecycleView的滑动实现标题栏透明度渐变
date: 2019-08-15
tags:
	- View
	- RecyclerView
categories: Android
---


## 改变布局背景透明度

让一个布局的背景变色，关键是这行代码：

```java
ll_tool_bar.getBackground().setAlpha(int alpha);
```

<!-- more -->

## 布局特点

布局采用`FrameLayout`后显示变化的`view`在最上层

```java
<?xml version="1.0" encoding="utf-8"?>
<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <!--列表-->
    <android.support.v7.widget.RecyclerView
        android:layout_width="match_parent"
        android:layout_height="wrap_content" />

    <!--标题栏-->
    <FrameLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content">
        <!--scrollbarThumbVertical 设置垂直滚动的时候的指示器-->
        <!--scrollbarTrackVertical设置右边滚动条的背景-->
        <TextView
            android:layout_width="match_parent"
            android:layout_height="match_parent"/>
    </FrameLayout>
</FrameLayout>
```

## 初始化数据

初始化一些数据

```java
//设置初始值
int mDistance = 0;
//当距离在[0,255]变化时，透明度在[0,255之间变化]
int maxDistance = 255;
//先设置标题栏不显示
ll_tool_bar.setVisibility(View.GONE);
```

## 设置滚动监听

然后通过监听列表滚动来进行需求实现

```java
//设置列表滚动监听
rv_show_contact.addOnScrollListener(new RecyclerView.OnScrollListener() {
    @Override
    public void onScrollStateChanged(RecyclerView recyclerView, int newState) {
        super.onScrollStateChanged(recyclerView, newState);

    }

    @Override
    public void onScrolled(RecyclerView recyclerView, int dx, int dy) {
        super.onScrolled(recyclerView, dx, dy);
        mDistance += dy;
        float percent = mDistance * 1f / maxDistance;//百分比
        int alpha = (int) (percent * 255);
        if (alpha == 0) {
            ll_tool_bar.setVisibility(View.GONE);
        } else if (alpha >= 100) {
            ll_tool_bar.setVisibility(View.VISIBLE);
        }

        if (alpha >= 255) {
            alpha = 255;
        }

        //标题栏渐变
        // a:alpha透明度 r:红 g：绿 b蓝

        //没有透明效果
        // titlebar.setBackgroundColor(Color.rgb(57, 174, 255));

        //透明效果是由参数1决定的，透明范围[0,255]
        // titlebar.setBackgroundColor(Color.argb(alpha, 57, 174, 255));
        ll_tool_bar.getBackground().setAlpha(alpha);
        Log.d("---",alpha + "----------");
    }
});
```

进行一个上下滑动 打印alpha 

```java
08-15 17:12:41.175 27151-27151/com.yqd.smartcontact D/---: 9----------
08-15 17:12:41.192 27151-27151/com.yqd.smartcontact D/---: 19----------
08-15 17:12:41.209 27151-27151/com.yqd.smartcontact D/---: 30----------
08-15 17:12:41.224 27151-27151/com.yqd.smartcontact D/---: 41----------
08-15 17:12:41.240 27151-27151/com.yqd.smartcontact D/---: 51----------
08-15 17:12:41.256 27151-27151/com.yqd.smartcontact D/---: 61----------
08-15 17:12:41.272 27151-27151/com.yqd.smartcontact D/---: 71----------
08-15 17:12:41.293 27151-27151/com.yqd.smartcontact D/---: 80----------
08-15 17:12:41.306 27151-27151/com.yqd.smartcontact D/---: 87----------
08-15 17:12:41.323 27151-27151/com.yqd.smartcontact D/---: 94----------
08-15 17:12:41.340 27151-27151/com.yqd.smartcontact D/---: 101----------
08-15 17:12:41.358 27151-27151/com.yqd.smartcontact D/---: 107----------
08-15 17:12:41.372 27151-27151/com.yqd.smartcontact D/---: 114----------
08-15 17:12:41.389 27151-27151/com.yqd.smartcontact D/---: 120----------
08-15 17:12:41.405 27151-27151/com.yqd.smartcontact D/---: 129----------
08-15 17:12:41.422 27151-27151/com.yqd.smartcontact D/---: 133----------
08-15 17:12:41.442 27151-27151/com.yqd.smartcontact D/---: 138----------
08-15 17:12:41.455 27151-27151/com.yqd.smartcontact D/---: 141----------
08-15 17:12:41.472 27151-27151/com.yqd.smartcontact D/---: 146----------
08-15 17:12:41.489 27151-27151/com.yqd.smartcontact D/---: 150----------
08-15 17:12:41.505 27151-27151/com.yqd.smartcontact D/---: 154----------
08-15 17:12:41.523 27151-27151/com.yqd.smartcontact D/---: 159----------
08-15 17:12:41.539 27151-27151/com.yqd.smartcontact D/---: 162----------
08-15 17:12:41.555 27151-27151/com.yqd.smartcontact D/---: 166----------
08-15 17:12:41.572 27151-27151/com.yqd.smartcontact D/---: 170----------
08-15 17:12:41.588 27151-27151/com.yqd.smartcontact D/---: 172----------
08-15 17:12:41.605 27151-27151/com.yqd.smartcontact D/---: 175----------
08-15 17:12:41.622 27151-27151/com.yqd.smartcontact D/---: 178----------
08-15 17:12:41.638 27151-27151/com.yqd.smartcontact D/---: 181----------
08-15 17:12:41.655 27151-27151/com.yqd.smartcontact D/---: 184----------
08-15 17:12:41.672 27151-27151/com.yqd.smartcontact D/---: 187----------
08-15 17:12:41.688 27151-27151/com.yqd.smartcontact D/---: 189----------
08-15 17:12:41.705 27151-27151/com.yqd.smartcontact D/---: 191----------
08-15 17:12:41.722 27151-27151/com.yqd.smartcontact D/---: 192----------
08-15 17:12:41.738 27151-27151/com.yqd.smartcontact D/---: 193----------
08-15 17:12:41.755 27151-27151/com.yqd.smartcontact D/---: 194----------
08-15 17:12:41.805 27151-27151/com.yqd.smartcontact D/---: 188----------
08-15 17:12:41.822 27151-27151/com.yqd.smartcontact D/---: 178----------
08-15 17:12:41.839 27151-27151/com.yqd.smartcontact D/---: 166----------
08-15 17:12:41.855 27151-27151/com.yqd.smartcontact D/---: 153----------
08-15 17:12:41.871 27151-27151/com.yqd.smartcontact D/---: 142----------
08-15 17:12:41.895 27151-27151/com.yqd.smartcontact D/---: 132----------
08-15 17:12:41.906 27151-27151/com.yqd.smartcontact D/---: 122----------
08-15 17:12:41.921 27151-27151/com.yqd.smartcontact D/---: 113----------
08-15 17:12:41.938 27151-27151/com.yqd.smartcontact D/---: 106----------
08-15 17:12:41.955 27151-27151/com.yqd.smartcontact D/---: 98----------
08-15 17:12:41.972 27151-27151/com.yqd.smartcontact D/---: 91----------
08-15 17:12:41.988 27151-27151/com.yqd.smartcontact D/---: 84----------
08-15 17:12:42.005 27151-27151/com.yqd.smartcontact D/---: 77----------
08-15 17:12:42.023 27151-27151/com.yqd.smartcontact D/---: 68----------
08-15 17:12:42.039 27151-27151/com.yqd.smartcontact D/---: 60----------
08-15 17:12:42.054 27151-27151/com.yqd.smartcontact D/---: 52----------
08-15 17:12:42.071 27151-27151/com.yqd.smartcontact D/---: 43----------
08-15 17:12:42.088 27151-27151/com.yqd.smartcontact D/---: 35----------
08-15 17:12:42.104 27151-27151/com.yqd.smartcontact D/---: 24----------
08-15 17:12:42.121 27151-27151/com.yqd.smartcontact D/---: 16----------
08-15 17:12:42.138 27151-27151/com.yqd.smartcontact D/---: 7----------
08-15 17:12:42.155 27151-27151/com.yqd.smartcontact D/---: 0----------
```

## 最终效果

这样就能实现随着recycleview的滑动来动态改变标题栏背景的透明度。

![](https://dev.tencent.com/u/liqingxue/p/picture_list/git/raw/master/gif/gifhome_480x960_4s.gif)