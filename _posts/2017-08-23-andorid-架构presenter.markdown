---
layout:     post
title:      "Presenter层如何高度的复用"
subtitle:   " \"Hello World, Hello Android\""
date:       2016-12-16 13:00:00
author:     "Cuieney"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - 架构
---

> “Yeah It's on. ”


### 前提
>这篇文章主要讲的是在我们的MVP CLEAN 涉及到PRESENTER层的架构，如何高度的复用我们已经写好的presenter，从而减少了很多代码量。写这篇文章也是自己的所思所想吧。有兴趣的同学可以看看一下两个链接关于Clean 和 mvp架构的。（如果文章中有错误或者结构问题欢迎指正）

* [CleanArchitecture](https://github.com/android10/Android-CleanArchitecture)
* [android-architecture](https://github.com/googlesamples/android-architecture)

这篇文章的目的很简单，如何高度抽象接口实现presenter复用。主要解决了UI页面中多数据请求中如何复用已写好的presenter。

#### 动机
我们先来讨论一下，当我把项目搭建成mvp架构的时候，是不是对于每一个页面的数据操作都要写好相应的presenter view model，同样你也可以写个Contract来完成view 和 presenter 的连接，这样也可以实现mvp的效果。

1. 如果对于单个页面单个数据请求可能会少写很多代码量
2. 对于单个页面的多个数据操作，我们的presenter会显得很臃肿。
3. 而且这些页面（单页面单数据请求，单页面多数据请求）会有很多相同的数据操作内容。没有得到复用

这篇文章就是为了解决这个问题

#### 看结构图
![architecture.png](http://upload-images.jianshu.io/upload_images/3415839-569f25dbfaa99e94.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

哈哈哈图画的可能比较简单，希望不要介意，好吧我们就从这个图开始说吧。

我这里就简单的说一下这个图的结构吧，因为每一个api（不论是网络请求还是数据库操作）都是有用的，所以我们可以把每个数据操作抽象成一个presenter 和 view 对于单数据操作的页面可以直接调用我们写好的preseter 对于多数据操作的可以通过一个powerPresenter来完成对这个页面的请求。

单数据操作的没有什么亮点，主要在于多数据操作如何完成view 层接口的回调和相应事件的回调。来完成这个powerPresenter

#### base的完成
>每个人写base可能都会有所不同，都是根据不同的业务来完成base的封装，我们如果想完成presenter 的高度复用base也是关键，我们View层的Base Presenter层的Base Act层的Base都是息息相关的。最终完成我们的目的，废话不多说上代码。

* View Layer

```
    public interface BaseView {
    
    void showLoading();

    void hideLoading();

    void showError(String message);

}

```
这里没什么内容也就是自己封装的基础业务。


* Presenter Layer

```
public interface Presenter<T extends BaseView> {

    void destroy();

    void attachView(T view);

    void detachView();
    
}

```
这层的主要功能也可以从接口上了解 就是绑定相应的View 和解绑防止内存泄漏（和大多数封装的接口都差不多）。Presenter 没有这么简单我们又再次对这一层进行了二次包装

```
public abstract class WrapperPresenter<T extends BaseView> implements Presenter<T> {

    T mView;

    @Override
    public void attachView(T view) {
        mView = view;
    }

    @Override
    public void detachView() {
        mView = null;
    }

}

```

这个包装的目的很简单就是省了每次写presenter需要传view进来直接从Activity中获取即可。

* Activity 

```
public abstract class BaseMvpAct<T extends WrapperPresenter> extends AutoLayoutActivity implements BaseView {
    protected T mPresenter;
    private Unbinder bind;


    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(addContentView());
        bind = ButterKnife.bind(this);
        mPresenter = createP();
        if (mPresenter != null) {
            mPresenter.attachView(this);
        }
        initialize();
    }

    public abstract void initialize();

    public abstract T createP();

    public abstract int addContentView();

    @Override
    protected void onDestroy() {
        super.onDestroy();
        if (mPresenter != null) {
            mPresenter.detachView();
        }
        if (bind != null) {
            bind.unbind();
        }
    }
    
}

```
这个就是BaseAct的基类，可以通过泛型看到我们要基础这个Act需要传WrapperPresenter的子类。我们刚刚也看到了在WrapperPresenter里面对view进行了绑定，而我们在Base就完成了这个操作，所以我们在实现一个api的presenter时就没必要把view 当参数传入了。
#### 子类的实现
>说了这么多，没看到复用啊，辣鸡，哎呀小伙子不要急嘛，慢慢来。

有了这些Base我们就可以展现真正的技术了，写好了这些base是不是开始疯狂的撸代码完成每个数据操作的Presneter 和相应的view 啊。我们就简单的完成写一个Presenter 和View 吧，老板上代码！！！！来了来了

###### view
```
public interface BannerView extends BaseView {
    void render(BannerBean bean);
}

```
###### presenter
```
public class BannerPresenter extends WrapperPresenter<BannerView>  {
    //大家不要想这个task啥东西就是网络请求一样
    private BannerTask task;
    public BannerPresenter() {
        task  = new BannerTask(new ApiIml(),new UIThread());
    }

    public void initialize(String type){
        mView.showLoading();
        this.getBanner(type);
    }

    public void getBanner(String type){
        //执行网络请求 回调到Observer 然后通过父类里面的view回参
        task.execute(new BannerObserver(), BannerTask.Params.forType(type));
    }

    private void BannerShow(BannerBean bean){
        mView.render(bean);
    }

    private void BannerLoadFailed(Throwable ex){
        mView.showError(ex.getMessage());
    }

    @Override
    public void destroy() {
        task.dispose();
    }

    private final class BannerObserver extends DefaultObserver<BannerBean>{
        @Override
        public void onNext(BannerBean bannerBean) {
            //回参到view中
            BannerPresenter.this.BannerShow(bannerBean);
        }

        @Override
        public void onComplete() {
            mView.hideLoading();
        }

        @Override
        public void onError(Throwable exception) {
            BannerPresenter.this.BannerLoadFailed(exception);
            mView.showError(exception.getMessage());
        }
    }
}

```
上面代码已经写了相应的注释我们就不必讲太多了。对于那个task 也不用管那么多 就是一个网络请求的封装。

###### Act
```
public class BannerAct extends BaseMvpAct<BannerPresenter> implements BannerView {
    @Override
    public void showLoading() {
        
    }

    @Override
    public void hideLoading() {

    }

    @Override
    public void showError(String message) {
        Log.i("cuieney", "showError: ");
    }

    @Override
    public void render(BannerBean bean) {
        Log.i("cuieney", "render: "+bean.toString());
    }

    @Override
    public void initialize() {
        mPresenter.initialize("banner");
    }

    @Override
    public BannerPresenter createP() {
        return new BannerPresenter();
    }

    @Override
    public int addContentView() {
        return 0;
    }
}


```
***代码很简单，看看都懂，完成了上图中的左半边单页面单数据请求。***

#### 重点（单页面多数据请求）
>终于回到主题了，如何完成presenter的复用 对于单页面多数据请求，而不用写多余的presenter。

从上面的图可以看出来我们input给了一些东西 （需要哪些数据操作的presenter），然后PowerPresenter output返回了相应的数据操作。不多说上代码

```
public class PowerPresenter <T extends BaseView> extends WrapperPresenter<T>{
    private T mView;

    private List<Presenter> presenters = new ArrayList<>();
    @SafeVarargs
    public final <Q extends Presenter<T>> void requestPresenter(Q... cls){
        for (Q cl : cls) {
            cl.attachView(mView);
            presenters.add(cl);
        }
    }

    public PowerPresenter(T mView) {
        this.mView = mView;
    }

    @Override
    public void destroy() {
        for (Presenter presenter : presenters) {
            presenter.destroy();
        }
    }

}

```
就问你们这个代码简不简单，容不容易。而且他也是WrapperPresenter的子类，也就是说我们BaseAct通用

* 首先我们看他的构造函数需要一个View 这个View 是继承BaseView的我们拿到这个View 干嘛呢当然是为了给数据操作完成后回参啊
* 然后我们又看到了这个方法很亮点requestPresenter里面传入一个数据，可以满足多页面多数据请求的操作。只要把我们需要的请求通过之前写好的presenter 传入即可。而不需要在写相应的presenter。从而达到复用。
* 第三个亮点是他完美的吧view 关联上了。由于每个数据请求都有对应的presenter 和view 这个类通过构造函数吧view 传了进来然后通过遍历数据吧view attachView上 完美的结合。
* 同样在destroy 的是完成了数据的解绑。

###### 单页面多数据操作如何完成
> 上代码，简单暴力 还是之前的BannerAct 修改成单页面多数据请求

```
public class BannerAct extends BaseMvpAct<PowerPresenter> implements BannerView {
    private BannerPresenter bannerPresenter;

    @Override
    public void showLoading() {

    }

    @Override
    public void hideLoading() {

    }

    @Override
    public void showError(String message) {
        Log.i("cuieney", "showError: ");
    }

    @Override
    public void render(BannerBean bean) {
        Log.i("cuieney", "render: "+bean.toString());
    }

    @Override
    public void initialize() {
        bannerPresenter.initialize("dsds");
    }

    @Override
    public PowerPresenter createP() {
        PowerPresenter presenter = new PowerPresenter<>(this);
        bannerPresenter = new BannerPresenter();
        ``````(这里你可以添加任何你需要的presenter)``````
        presenter.requestPresenter(bannerPresenter);
        return presenter;
    }

    @Override
    public int addContentView() {
        return 0;
    }
}

```
通过上面代码可以看到，我们在实现了父类里面的createP方法，然后通过创建PowerPresenter 然后把我们需要的BannerPresenter网络请求加到了requestPresenter请求中 达到了BannerPresenter 的复用，简短的几句代码。


### ending
又到了说再见的时候了，希望老铁们看到以上文章有不对的地方和需要改善的地方指正一下。么么哒


—— Cuieney 后记于 2017.03


