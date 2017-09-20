---
layout:     post
title:      "手把手教学RxBinding"
subtitle:   " \"Hello World, Hello Android\""
date:       2016-12-16 13:00:00
author:     "Cuieney"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - 手把手教学
---


## 前提

这篇文章针对对Rx家族的了解过得同学们···，不然学起来会有点麻烦哦！（jakewharton 大神膜拜一下）
这是 [RxBinding](https://github.com/JakeWharton/RxBinding)的链接地址，用过的都说好，好了废话不多说，接下来我们来看看怎么写一份自己的RxBinding示例</p>

[跳过废话，直接看技术实现 ](#build) 


### 了解步骤
------

第一步，你需要了解RxJava 中的观察者和被观察者。
第二步，你需要了解观察者和被观察之间的关系。
第三步，最重要的一步，他们的创建方式和订阅方式。

### 叙述
------

好了，了解了上面的三步骤之后，咱们就开始了解RxBinding的神奇之处。其实关于RxBinding的使用已经有很多博主写过了，而且很详细，想学习可以搜一下即可。废话不多说，我们来说一下RxBinding如何自己手写吧。

1.首先从RxView开始说吧。看看大神是如何写的，至于它的优点我这就不多说了，想知道的可以google一下。下面是RxView的使用方法

```
RxView.clicks(forgetPwd)        
.throttleFirst(1,TimeUnit.SECONDS)        
.subscribe(new Action1<Void>() {          
  @Override            
  public void call(Void aVoid) {                
    startActivityForResult(new Intent(context,ResetPwdActivity.class),1);           
 } });

```

可以看到clicks是一个静态的方法，然后返回的对象是一个Observable，然后订阅了Observer，但是在Observer之前我们同样可以做一些（比如说，按钮防抖，延迟，过滤）等操作。我们从RxView的源码进行分析，看看，里面是如何写出来的，请看下面（只说一些重要的源码内容）

```
@CheckResult @NonNull
public static Observable<Void> clicks(@NonNull View view) {    
checkNotNull(view, "view == null");  
return Observable.create(new ViewClickOnSubscribe(view));
}

```
从上面可以看出，RxView.clicks这个方法中要传入一个view ，然后通过Observable.create的方式创建一个被观察者。由于setOnClickListener事件返回值是当前对象吗，我们也就把泛型写错Void即可。好的现在最重要的来了，就是
ViewClickOnSubscribe这个类的内容，通过什么方式穿件的点击订阅内容呢。


```
final class ViewClickOnSubscribe implements Observable.OnSubscribe<Void> {  
final View view;  
  ViewClickOnSubscribe(View view) {  
      this.view = view;  
  }    
  @Override
  public void call(final Subscriber<? super Void> subscriber) {    
    checkUiThread();    
    View.OnClickListener listener = new View.OnClickListener() {      
        @Override         
      public void onClick(View v) {        
          if (!subscriber.isUnsubscribed()) {          
            subscriber.onNext(null);       
           } } };   
       view.setOnClickListener(listener);   
       subscriber.add(new MainThreadSubscription() {     
       @Override 
       protected void onUnsubscribe() {       
         view.setOnClickListener(null);     
      } }); 
 }}

```

如果这段代码看懂之后，大家已经就回写RxBinding了，首先我们看一下ViewClickOnSubscribe这个类，他实现了Observable.OnSubscribe<T>这个方法，构造函数把View 传了进来，首先可以看出view被当成一个被观察的形式传了进来，这时候我们是不是要监听这个被观察者的动态，然后发给观察者呢，是的就是这样，代码逻辑 也是这样，RxView.clicks这个方法就是来做点击事件的监听的，既然我们ViewClickOnSubscribe实现了这个类，我们便可以在call回调的方法中去监听view 的点击事件，然后传给观察者，代码中可以看到，我们实现了OnClickListener的方法，然后在onClick中我们订阅者对这个事件进行了回调。同时订阅者也添加取消订阅的操作，在订阅取消的时候对view的监听停止。

其实大家点开源码看RxView中不止clicks这个功能，还有许多比如pressed，selected，visibility，都是再一次的封装，大家有兴趣可以研究一下。

接下来就是重点了，写一个属于自己的RxBinding。我就已传感器为例子说一下实现的步骤吧。
 
 
```
public class TestActivity extends Activity implements SensorEventListener {           
       SensorManager sensormanager;    
       @Override    
       public void onCreate(Bundle savedInstanceState) {        
              super.onCreate(savedInstanceState);                    
              setContentView(R.layout.activity_main);        
              sensormanager =       
              (SensorManager)getSystemService(Context.SENSOR_SERVICE);            
                    sensormanager.registerListener(this,                    
                sensormanager.getDefaultSensor(Sensor.TYPE_ACCELEROMETER),                    
                SensorManager.SENSOR_DELAY_GAME);   
 }    
    @Override   
    public void onSensorChanged(SensorEvent event) {
            }    
    @Override   
    public void onAccuracyChanged(Sensor sensor, int accuracy) { 
            }    

    @Override    
    protected void onDestroy() {        
        super.onDestroy();        
        sensormanager.unregisterListener(this);    
}}


```



传感器信息的回调对吧，通过onSensorChanged获取三轴的重力加速度
，其中onAccuracyChanged这个方法几乎用不到的，我们完成没有必要写，但是因为SensorEventListener我们必须的实现。而且传感器的三轴的重力加速度的回调的速度，你没法控制。只能通过代码逻辑来实现。但是我们把它封装成RxBinding的形式不仅减少了代码量，同时也可以实现控制三轴的重力加速度的回调速度。
但是我们又该如何封装成RxBinding呢，这时候我们可以根据上面分析的RxView 来进行模仿，不会写没问题，但是我们可以模仿啊。好的，现在我们来分析如何写错RxView的形式。首先我们的先创建的RxSensor（RxView）这个类把。分析一下，我们需要观察什么呢，当然是传感器了（废话），那我们我们怎么来观察它呢！好问题，那就是实现SensorEventListener这个监听才能观察传感器，那么我们观察得有个东西来接收这个监听把，这时候就需要SensorManager（View）对的就是要传入这个类。我们接下来来创建传感器的被观察者

```
public static Observable<float[]> registerSensor(SensorManager sensorManager){    
return Observable.create(new SensorChangeOnSubscribe(sensorManager));
}
```

是不是有一种似曾相识的感觉，和上面我们分析的clicks方法很相似把。但是，我们这里面要拿到传感器的数据，需要把泛型写错数组形式，其实重点当然是SensorChangeOnSubscribe这个类里面的具体内容了，相信很多小伙伴已经迫不及待了。

```
public class SensorChangeOnSubscribe implements Observable.OnSubscribe<float[]> {    
final SensorManager sensormanager;    
public SensorChangeOnSubscribe(SensorManager sensormanager) {        
          this.sensormanager = sensormanager;   
 }    
@Override    
public void call(final Subscriber<? super float[]> subscriber) {        
    final SensorEventListener listener = new SensorEventListener() {            
            @Override            
          public void onSensorChanged(SensorEvent event) {                
                    float[] values = event.values;                
                    subscriber.onNext(values);            
          }           
           @Override            
          public void onAccuracyChanged(Sensor sensor, int accuracy) {            
          }   };       
     sensormanager.registerListener(listener, 
     sensormanager.getDefaultSensor(Sensor.TYPE_ACCELEROMETER), 
     SensorManager.SENSOR_DELAY_GAME);        
     subscriber.add(new MainThreadSubscription() {            
          @Override 
          protected void onUnsubscribe() {                        
                sensormanager.unregisterListener(listener);           
          }  });       

    subscriber.onNext(new float[]{0,0,0});    
}}
```

好了看到这是不是豁然开朗啊，我们通过SensorChangeOnSubscribe的构造函数拿到SensorManager来注册传感器，然后通过传感器的回调拿到SensorEvent的数据通过订阅者把数据回传出去，然观察者拿到数据。而且还不需要写各种实现方法了比如onAccuracyChanged（），最后一步还是需要在最后一步再写一步

```
subscriber.onNext(new float[]{0,0,0});

```
不懂得可以看看RxTextView的源码。相信你会了解到。接下来展示一下，通过RxSensor实现的Activity

```
public class TestActivity extends Activity {    
    SensorManager sensormanager;    
    @Override    
    public void onCreate(Bundle savedInstanceState) {             
               super.onCreate(savedInstanceState);          
               setContentView(R.layout.activity_main);        
               sensormanager = (SensorManager) getSystemService(Context.SENSOR_SERVICE);   
     
             RxSensor.registerSensor(sensormanager)                
                            .debounce(1, TimeUnit.SECONDS)               
                            .subscribe(new Action1<float[]>() {                    
                            @Override                    
                            public void call(float[] floats) {   
                                           //拿出传感器数据
                             }               
                            });  
   }
}

```

### 总结
------

是不是简单了很多很多很多，这就是RxBinding的神奇之处，debounce就延迟操作，相当于每秒取一次数据。
好了，就说到这把。好累好累 我要去休息了。





—— Cuieney 后记于 2016.12


