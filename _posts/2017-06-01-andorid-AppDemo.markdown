---
layout:     post
title:      "App界的一股清流"
subtitle:   " \"Hello World, Hello Android\""
date:       2016-12-16 13:00:00
author:     "Cuieney"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - App
---

> “Yeah It's on. ”


![logo.png](http://upload-images.jianshu.io/upload_images/3415839-1f12062b30cad974.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



#### 目前项目持续更新重构中(向kotlin靠拢)
>life 是一个多媒体信息app，基于Material Design Kotlin + MVP + RxJava + Retrofit + Dagger2 + GreenDAO + Glide

### 前提
做这款app主要是出于Android日常开发中或多或少的都会仿着ios的样式来写ui（可能设计师就做了一份ios交互设计，android只能跟着去写相同ui），完全舍弃了MD风格，第一出于学习目的做的，第二出于想写一个完全按照MD风格的App。

### 目前包括以下内容

视频来自：开眼 <http://www.eyepetizer.net/>汇集各种炫酷视频

音乐来自：余音 <http://app.mi.com/details?id=fm.wawa.mg/>文艺骚年专属

文章来自：余音 <http://www.wufazhuce.com/>韩寒主编和监制

### tips
1.本项目目前只是在开发测试阶段，发现bug或有好的建议欢迎
[issues](https://github.com/Cuieney/vld/issues/ "悬停显示")
Email <cuieney@163.com> link.

2.本项目仅作为学习交流使用，API数据内容所有权归原公司所有，请勿用于其他用途

### Preview

![city.gif](http://upload-images.jianshu.io/upload_images/3415839-947aa36617544c49.gif?imageMogr2/auto-orient/strip)


![pano_home.png](http://upload-images.jianshu.io/upload_images/3415839-697431c5fc601472.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![pano.png](http://upload-images.jianshu.io/upload_images/3415839-4c9ae85731bb8f83.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![media.png](http://upload-images.jianshu.io/upload_images/3415839-2e1d6715d1bf9d50.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)![media_home.png](http://upload-images.jianshu.io/upload_images/3415839-2b8a96d83515445d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![media_home_detail.png](http://upload-images.jianshu.io/upload_images/3415839-a72df567f3fd8514.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![music.png](http://upload-images.jianshu.io/upload_images/3415839-13419e0ba426dc2a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![music_home.png](http://upload-images.jianshu.io/upload_images/3415839-cd874bf763828ede.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![music_detail.png](http://upload-images.jianshu.io/upload_images/3415839-1907e28ed13a33f1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![essay_home.png](http://upload-images.jianshu.io/upload_images/3415839-de6f2fd3de1fe2b8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![essay_detail.png](http://upload-images.jianshu.io/upload_images/3415839-c6a564e6ca43d14b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### Download APK
(Android 5.0 or above)

[URL](https://github.com/Cuieney/vld/blob/master/image/app-debug.apk "悬停显示")

### Points
	使用Rx家族配合Retrolambda减少代码量
	使用RxJava配合Retrofit2做网络请求
	使用Rxlifecycle对订阅的生命周期做管理
	使用RxBus来方便组件间的通信
	使用RxJava其他操作符来做延时、轮询、转化、筛选等操
	使用okhttp3对网络返回内容做缓存，还有日志、超时重连、头部消息的配置
	使用Material Design控件和动画
	使用Ijkplayer来实现播放视频功能
	使用MVP架构整个项目，对应于model、ui、presenter三个包
	使用Dagger2将M层注入P层，将P层注入V层，无需new，直接调用对象
	使用GreenDAO做阅读记录和收藏记录的增、删、查、改
	使用Glide做图片的处理和加载
	使用Fragmentation简化Fragment的操作和懒加载
	使用Statusbaruitl动态的改变通知栏颜色
	使用XRecyclerView实现下拉刷新、上拉加载
	使用SVG及其动画实现progressbar的效果
	包含搜索、收藏、检测更新等功能


### Version

#### V2.5.0
1.增加vr模块panorama liveview 代码（to kotlin）

2.类似于insta360 全景图片预览

3.你们的支持就是我最大的动力，持续更新中 （to kotlin）


#### V2.4.0
1.更新音乐和视频播放页面代码（to kotlin）

2.添加umeng 收集错误

3.你们的支持就是我最大的动力，持续更新中 （to kotlin）

#### V2.3.0
1.music tab 更新为kotlin代码(功能并未完善)

2.你们的支持就是我最大的动力，持续更新中 （to kotlin）

#### V2.2.0
1.video tab 更新为kotlin代码(功能并未完善)

2.你们的支持就是我最大的动力，持续更新中 （to kotlin）

#### V2.1.0
1.添加kotlin代码块，essay tab 目前项目是kotlin and java 混编

2.增加kotlin act and fragment 基类 dagger2等 (功能并未完善)

3.持续更新中 （to kotlin）

#### V2.0.0
1.增加essay tab页面，修改了一些bug 更新了app icon(功能并未完善)


#### V1.0.0
1.第一版本提交(功能并未完善)
### Thanks

#### API

[开眼](http://www.eyepetizer.net/ "悬停显示") [余音](http://app.mi.com/details?id=fm.wawa.mg/ "悬停显示")
[一个](http://www.wandoujia.com/apps/one.hh.oneclient/ "悬停显示")

#### APP:

[Material Design](https://material.io/guidelines/components/bottom-navigation.html#bottom-navigation-behavior "悬停显示") 官方提供了部分设计思路

[android-architecture](https://github.com/googlesamples/android-architecture"悬停显示")和
[GankClient-Kotlin](https://github.com/githubwing/GankClient-Kotlin "悬停显示")提供了Dagger2配合MVP的架构思路

还参考了很多大神的类似作品，感谢大家的开源精神

#### RES

[iconfont](http://www.iconfont.cn/plus/search/index "悬停显示")
 提供了icon素材
 
 [material UP ](https://material.uplabs.com/ "悬停显示")
 提供了Material Design风格的素材


### LIB
#### UI

1. [BottomNavigation](https://github.com/Ashok-Varma/BottomNavigation "悬停显示")
 
2. [floatingsearchview](https://github.com/arimorty/floatingsearchview "悬停显示")

3. [expandableTextView](https://github.com/Manabu-GT/ExpandableTextView "悬停显示")

4. [xrecyclerview](https://github.com/jianghejie/XRecyclerView "悬停显示")

5. [statusbaruitl](https://github.com/laobie/StatusBarUtil "悬停显示")

#### RX
1. [RxJava](https://github.com/ReactiveX/RxJava "悬停显示")

2. [RxAndroid](https://github.com/ReactiveX/RxAndroid "悬停显示")

3. [RxPermissions](https://github.com/tbruyelle/RxPermissions "悬停显示")

4. [RxLifecycle](https://github.com/trello/RxLifecycle "悬停显示")

#### VIDEO
1. [ijkplayer](https://github.com/Bilibili/ijkplayer "悬停显示")

2. [GSYVideoPlayer](https://github.com/CarGuo/GSYVideoPlayer "悬停显示")

#### NETWORK

1. [Retrofit](https://github.com/square/retrofit "悬停显示")

2. [OkHttp](https://github.com/square/okhttp "悬停显示")

3. [Glide](https://github.com/bumptech/glide "悬停显示")

4. [Gson](https://github.com/google/gson "悬停显示")

#### DI
1. [Dagger2](https://github.com/google/dagger "悬停显示")

2. [ButterKnife](https://github.com/JakeWharton/butterknife "悬停显示")

#### FRAGMENT
1. [Fragmentation](https://github.com/YoKeyword/Fragmentation "悬停显示")

#### LOG
1. [Logger](https://github.com/orhanobut/logger "悬停显示")

#### DB
1. [greenDAO](https://github.com/greenrobot/greenDAO "悬停显示")

#### CANARY
1. [BlockCanary](https://github.com/markzhai/AndroidPerformanceMonitor "悬停显示")
2. [leakcanary](https://github.com/square/leakcanary "悬停显示")


### ending

喜欢的同学欢迎star issues and fork 
[项目地址](https://github.com/Cuieney/kotlin-life "悬停显示")

—— Cuieney 后记于 2017.03


