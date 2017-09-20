---
layout:     post
title:      "彩虹进度条"
subtitle:   " \"Hello World, Hello Android\""
date:       2016-12-16 13:00:00
author:     "Cuieney"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - 自定义view
---

> 某天的某个时刻，看到了代码家的开源库NumberProgressBar，里面有着色彩鲜明的进度条。无所事事的我，就想到了写个彩虹进度条。idea来源于这

### 首先呢

> 对于彩虹肯定要有色彩丰富的颜色了，而且还要美观大方，这个自定义view并不难，相信写过的自定义view的都有能力写出这样的进度条。

那么不多说先上图：

![progress.gif](http://upload-images.jianshu.io/upload_images/3415839-fe6132ebe9954508.gif?imageMogr2/auto-orient/strip)

就是这么简单的一个进度条，没有任何花哨，简单大方。

### 源码解析 _ _!

1.进度条吗，肯定你的首先新建一个class吧，然后呢当然是继承View了，咱们就建个RainbowProgressBar这个类吧，对于view 的绘制流程那我就不说了，大家自行google。

2.那么类建好了要干嘛呢，当然是配置需要的attribute的属性了，和代码中的初始化方法了。下面是attrs里面的内容

```

<resources>
    <declare-styleable name="RainbowProgressBar">
        <attr name="progress_current" format="integer"/>
        <attr name="progress_max" format="integer"/>

        <attr name="progress_start_color" format="color"/>
        <attr name="progress_end_color" format="color"/>
        <attr name="progress_unreached_color" format="color"/>

        <attr name="progress_height" format="dimension"/>
        <attr name="progress_radius" format="dimension"/>

        <attr name="progress_type" format="enum">
            <enum name="circle" value="0"/>
            <enum name="line" value="1"/>
        </attr>
    </declare-styleable>
</resources>

```

好了我的属性也就这些。

3.那么接下来干嘛呢，当然是在构造方法里获取这些用户需要配置的属性了
and 初始化画笔

```

  public RainbowProgressBar(Context context, AttributeSet attrs) {
        super(context, attrs);
        TypedArray attributes = context.obtainStyledAttributes(attrs,
                R.styleable.RainbowProgressBar);
        max = attributes.getInteger(R.styleable.RainbowProgressBar_progress_max,100);
        progress = attributes.getInteger(R.styleable.RainbowProgressBar_progress_current,0);
        startColor = attributes.getColor(R.styleable.RainbowProgressBar_progress_start_color,startColor);
        endColor = attributes.getColor(R.styleable.RainbowProgressBar_progress_end_color,endColor);
        radius = attributes.getDimension(R.styleable.RainbowProgressBar_progress_radius,dp2px(35));
        progressHeight = attributes.getDimension(R.styleable.RainbowProgressBar_progress_height,dp2px(5));
        unreachedColor = attributes.getColor(R.styleable.RainbowProgressBar_progress_unreached_color,unreachedColor);
        type = attributes.getInteger(R.styleable.RainbowProgressBar_progress_type,1);
        attributes.recycle();
        init();
    }
    

```

大概就这些属性了。这些都无关紧要的。
#### 我们来说重点吧，如何画出这样的彩虹呢

1.在开始说这个之前你必须的了解一下某个类，可能用的不多吧，主要功能是用来渲染色彩的 duang duang duang 没错就是他让我们的进度条上色了LinearGradient
总共要传七个参数，ヾ(｡｀Д´｡)我擦咋这么多呢。前四个参数就是你的着色范围，后三个参数，有一个还可以为空呢，另一个呢就是要你配置它的色彩，然后最后一个呢，那就是着色的模式了。简单吧

2.了解了LinearGradient这个类之后那我们开始画画了哦，but 现在还是不行的，我们需要获取这些 用户配置好的起始颜色和结束颜色，直接的过度颜色啊，不然你怎么画啊
用户给我们的肯定是hex 16进制的颜色啊（#ff00ff）就这样的,那我们现在上代码吧，下面这个代码是，hex 转 ARGB

```
 private int[] convertColor(int color) {
        int alpha = (color & 0xff000000) >>> 24;
        int red = (color & 0x00ff0000) >> 16;
        int green = (color & 0x0000ff00) >> 8;
        int blue = (color & 0x000000ff);
        return new int[]{alpha, red, green, blue};
    }


```
那我们怎么获取每个点（就是每个进度点）对应的颜色呢，不多说上代码

```

 private int getCurrentColor(int progress) {
        int alpha = 0;
        int red = 0;
        int green = 0;
        int blue = 0;

        if (progress == 0) {
            return startColor;
        }
        int[] startARGB = convertColor(startColor);
        int[] endARGB = convertColor(endColor);

        alpha = startARGB[0] - ((startARGB[0] - endARGB[0]) / max * progress);
        red = startARGB[1] - ((startARGB[1] - endARGB[1]) / max * progress);
        green = startARGB[2] - ((startARGB[2] - endARGB[2]) / max * progress);
        blue = startARGB[3] - ((startARGB[3] - endARGB[3]) / max * progress);

        return Color.argb(alpha, red, green, blue);
    }
        
```

我们根据每个进度的位子拿到当前的颜色点即可，

3.刚刚说的那个叼叼类还记得吗（LinearGradient），我们做的这一切准备都是为了给他colors数组的，就问你这家伙叼不刁吧，给他吧，谁让他这么叼

```
 private void initColors() {
        for (int i = 0; i < max; i++) {
            colors[i] = getCurrentColor(i);
        }
    }


```

4.好了我们准备工作做好了，那么现在要干嘛呢，当然是拿着我们的画板（Canvas）和笔（Paint） 画画了
因为我们配置文件中设置了type类型所以会有线性和圆的区别

```
    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        if (type == CIRCLE_TYPE) {
            drawCircle(canvas);
        }else if(type == LINE_TYPE){
            drawLine(canvas);
        }
    }

```
接下来奉上线性代码

```
   private void drawLine(Canvas canvas) {
        measuredWidth = getWidth();
        measuredHeight = getHeight();
        canvas.drawLine(0, 0, measuredWidth, 0, paint);
        float x1 = measuredWidth / max * progress;
        shader = new LinearGradient(0, 0, x1, 0, colors, null,
                Shader.TileMode.CLAMP);
        rainbowPaint.setShader(shader);
        canvas.drawLine(0, 0, x1, 0, rainbowPaint);
    }

```

接下来奉上圆形代码

```
    private void drawCircle(Canvas canvas) {
        measuredWidth = getWidth();
        measuredHeight = getHeight();
        float x = radius+progressHeight;
        float y = radius+progressHeight;
        float sweepAngle = 360f / max * progress;

        canvas.drawCircle(x, y, radius, paint);
        shader = new LinearGradient(x - radius, y - radius,
                x - radius + radius, y + radius, colors, null, Shader.TileMode.MIRROR);
        rainbowPaint.setShader(shader);
        RectF rect = new RectF(((int) (x - radius)),
                ((int) (y - radius)),
                ((int) (x + radius)),
                ((int) (y + radius)));
        canvas.drawArc(
                rect,
                -90, sweepAngle, false, rainbowPaint
        );
    }

```
怎么样，还算满意吗！！！

这不就结束了彩虹进度条哦！！！！

![line 上午11.38.23.png](http://upload-images.jianshu.io/upload_images/3415839-6f4945c3c8cc39a2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

样子就是这样的

### ending

代码已开源GitHub： [RainbowProgressBar](https://github.com/Cuieney/RainbowProgressBar )

欢迎star 互相关注哦！！！



—— Cuieney 后记于 2017.02


