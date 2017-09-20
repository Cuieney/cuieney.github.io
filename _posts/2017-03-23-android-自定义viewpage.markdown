---
layout:     post
title:      "背景可移动的viewpager"
subtitle:   " \"Hello World, Hello Android\""
date:       2016-12-16 13:00:00
author:     "Cuieney"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - 自定义view
---

> 这个控件是由于在开发新项目中需求要使用的，当时感觉这个需求真是反人类啊😁，让viewpager可以带背景而且背景可以跟着pager的滑动在移动

### 标题省略
但是设计师当然可以随便画画，js脚本写写就出来了啊，mdzz但是代码实现当然要各种逻辑然后互相调用才能达到那个效果，也有可能效果并不如意。

下面附上效果图

![second.gif](http://upload-images.jianshu.io/upload_images/3415839-4ad4aab8467e2215.gif?imageMogr2/auto-orient/strip)

就是这个样子的，感觉引用场景真的很少，我们的app大概是能用到的，哈哈哈~~~

### 原理分析

1.首先我们看到这个应该向肯定是在act 里面弄一个imageview 然后在吧viewpager 覆盖在上方，然后pager里面添加的view货fragment 背景透明就好，然后监听viewpager的滚动事件，然后拿到x轴的移动距离然后动态的移动imageview，对吧。but 哼那是不行的。（窝草 我感觉我说了这么多废话啊。mdzz）

2.进入主题，那我们怎么实现这个需求呢，当然是有方法的，想到可以横向滚动的控件了吗，心里有数了吧 HorizontalScrollView 这个控件来和viewpager实现联动

献上我的代码 layout

```
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">


    <HorizontalScrollView
        android:id="@+id/scrollView"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:scrollbars="none">

        <ImageView
            android:id="@+id/image"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:scaleType="fitXY"
            android:src="@mipmap/bg_feet" />


    </HorizontalScrollView>


    <android.support.v4.view.ViewPager
        android:id="@+id/vpg"
        android:layout_width="match_parent"
        android:layout_height="match_parent" />


    <LinearLayout
        android:id="@+id/point"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginTop="@dimen/point_margin_top"
        android:gravity="center"
        android:orientation="horizontal" />
</RelativeLayout>


```

然后就是我们怎么样实现这个联动效果呢，如果设置的不到位，你会得到很不好的效果，不仅滑动相反，划不动 ，背景不动，什么事都会有的。
再次奉献上我的代码，多次调试后发现这样才可以完成这个动作，但是在低配机器上会有点卡顿的。我们只需要做的就是监听viewpager 然后滑动HorizontalScrollView，即可

```
 vpg.addOnPageChangeListener(new ViewPager.OnPageChangeListener() {
            @Override
            public void onPageScrolled(int position, float positionOffset, int positionOffsetPixels) {

                // pager所有子页面的总宽度
                float widthOfPagers = vpg.getWidth() * adapter.getCount();
                // 背景图片的宽的
                float widthOfScroll = image.getWidth();

                // ViewPager可滑动的总长度
                float moveWidthOfPagers = widthOfPagers - vpg.getWidth();
                // 背景图的可滑动总长度
                float moveWidthOfScroll = widthOfScroll - vpg.getWidth();

                // 可滑动距离比例
                float ratio = moveWidthOfScroll / moveWidthOfPagers;
                // 当前Pager的滑动距离
                float currentPosOfPager = position * vpg.getWidth() + positionOffsetPixels;

                // 背景滑动到对应位置
                horizontalScrollView.scrollTo((int) ((currentPosOfPager * ratio) / 3),
                        horizontalScrollView.getScrollY());

            }


```
就是这样，上面说的很明白吧，每个都给你写好了注释。我是多体贴啊。完美

### 结束语

然后我在这基础上进行了封装，把他写出了一个自定义控件，然而你什么都不需要设置，只是在xml文件配置一下即可。

代码已开源GitHub： [ExtendVpg](https://github.com/Cuieney/ExtendVpg )

欢迎star 互相关注哦！！！




—— Cuieney 后记于 2017.03


