---
layout:     post
title:      "DexClassLoader热修复的入门到放弃"
subtitle:   " \"Hello World, Hello Android\""
date:       2016-12-16 13:00:00
author:     "Cuieney"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - HotFix
---

> “Yeah It's on. ”


### 前提
>写这篇文章的目的也是为了了解android源码及hack技术，读了这篇文章相信你也可以了解到Dalvik的工作流程，apk的生成过程，及build.gradle中plugin中ApplicationPlugin的Task有哪些，如何通过hack技术来完成hotfix。有兴趣的同学也可以看看groovy如何编写Plugin，及如何优化dex来让优化app

#### 热修复需要注意的几个问题
- 如何进行hack来达到热修复
- hack操作中需要使用哪些class
- apk是生成的生命周期
- gradle build 脚本(groovy)
我们先了解这些问题后再进行具体的操作步骤，来个循序渐进。问题接下来会一一的详解

##### 如何进行hack来达到热修复
为什么会有热修复这个东西呢？大家都知道如果我们的线上的app 由于某种原因crash？我们这时候不能怨测试没测好，后台接口有变化什么的，这不是解决问题的最终方式！要是以前我们肯定就是把重新上传app到各大渠道，从新上线，这个过程严重的影响到我们的用户体验非常不好，而且很耗时！作为程序员如何通过代码进行线上修复crash bug。。。呢？所以有了热修复这个功能 bat 每家都有自己的开源热修复库？我这里就讲一下如何通过反射的方式来实现修复功能吧！也就是通过DexClassLoader。如果大家对其他的开源库想要了解的话可以通过一下传送门
[AndFix](https://github.com/alibaba/AndFix)
[tinker](https://github.com/Tencent/tinker)
[HotFix](https://github.com/dodola/HotFix)
[Robust](https://github.com/Meituan-Dianping/Robust)
我这里也就讲一下Dex的方式修复

#### hack操作中需要使用哪些class
- 顾名思义DexClassLoader这个必须要用到的
- javaassist用于代码的打桩（就是class文件代码的植入，这里不详解了）
- groovy一个android plugin插件开发语言 底下会提及到
-

#### apk是生成的生命周期
我们的项目如何在编译的时候变成apk呢？
1. 第一步当然是把我们的资源文件生成R.Java文件了
2. 处理AIDL文件，生成对应的.java文件（当然，有很多工程没有用到AIDL，那这个过程就可以省了）
3. 编译Java文件，生成对应的.class文件
4. 把.class文件转化成Davik VM支持的.dex文件
5. 打包生成未签名的.apk文件
6. 对未签名.apk文件进行签名
7. 对签名后的.apk文件进行对齐处理（不进行对齐处理是不能发布到Google Market的）
我感觉还是贴图比较靠谱不然看文字没有感觉

![build.png](http://upload-images.jianshu.io/upload_images/3415839-6deb39382c68ddad.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这就是一个apk编译所走的生命周期，但是我们的build脚本到底走了哪些任务呢。如果想看的话可以在我们module中的build.gradle 加入如下代码 即可在console中看到相应的任务
```
tasks.whenTaskAdded { task ->
    println(task.name+"===")
}

```
这个就是我们apkbuild的时候的每一个task。既然知道了这些task 那我们如何才能知道这些task到底在后台做了些什么呢？
#### gradle build 脚本(groovy)
时常见到却不知道他在干嘛的一句代码，apply plugin: 'com.android.application'如果我们把com.android.application代替为com.android.library，那我们的build目录下的output那就是aar包了。组件化开发会用到这样的切换想了解的可以看看[（组件化）](http://www.jianshu.com/p/b7d4e6612e0c)

想要了解这句话干嘛的那你必须的知道这个开发语言groovy，他是支持android studio的。我们可以自定义我们想要的插件，在编译的时候进行一些好玩的操作。这里面可以定义许多task，回归正题，这句代码到底干了哪些事情呢，[那我们就必须的了解这个源码](https://android.googlesource.com/platform/tools/build/+/cab495f54cd31e4e93c36e6aa4b7af661aac2357/gradle/src/main/groovy/com/android/build/gradle/AppPlugin.groovy)想要了解的同学可以看看这里就不多说了，这里面有我们build中的所有task

####app启动过程
以上了解了这么多，接下来就要进入正题了！现在说app的启动过程，过程就不细说了，因为经历了很多复杂的过程，我就说一下与DexClassLoader有关的事情吧。

app每次启动fork一个进程但同时也会同样会分配一个dalvik虚拟机供这个app运行，在dalvik中每次运行都需要读取apk里面的***dex***文件，这样会耗费很多cpu资源，然后采用odex，把dex文件优化成***odex***文件，那么odex操作给我们热修复带来了哪些问题呢？我们先把这个问题记录下来，之后会分析具体原因。启动先说到这！！！

#### 如何进行热修复
- 通过上面了解了app启动过程中每次都要通过dvm来加载dex文件。
- 同样大家也知道了dex文件是由 .java->.class->dex 一步一步转化来的
了解了上面的两个重要的东西，热修复就是每次在我们app启动的时候加载我们自己的patch.dex文件而不是加载就是我们修复的dex文件，这样就可以达到热修复了（这时大概会有很多同学困惑，dvm怎么知道就用我们的patch.dex而不用之前的呢？好问题 让老夫徐徐道来）

#### 动态加载patch.dex
- 在 Android 中，App 安装到手机后，apk 里面的 class.dex 中的 class 均是通过 PathClassLoader 来加载的。
- DexClassLoader 可以用来加载 SD 卡上加载包含 class.dex 的 .jar 和 .apk 文件
- DexClassLoader 和 PathClassLoader 的基类 BaseDexClassLoader 查找 class 是通过其内部的 DexPathList pathList 来查找的
- DexPathList 内部有一个 Element[] dexElements 数组，其 findClass() 方法（源码如下）的实现就是遍历该数组，查找 class ，一旦找到需要的类，就直接返回，停止遍历：


```
public Class findClass(String name, List<Throwable> suppressed) {
    for (Element element : dexElements) {
        DexFile dex = element.dexFile;
        if (dex != null) {
            Class clazz = dex.loadClassBinaryName(name, definingContext, suppressed);
            if (clazz != null) {
                return clazz;
            }
        }
    }
    if (dexElementsSuppressedExceptions != null) {
        suppressed.addAll(Arrays.asList(dexElementsSuppressedExceptions));
    }
    return null;
}

```
通过上面的步骤是不是知道了我们app每次启动一个class是如何找到类的呢？现在知道了吧，DexClassLoader -> DexPathList -> Element[] 
好的 现在应该有一些系统的了解了，通过上面的步骤可以知道 每次查找类都是通过Element[]中查找的。如果找到就会return 而不会继续找！这时候嘿嘿嘿我们知道了他是如何findclass 的那我们就可以悄悄的干些坏事了（这里会有一些同学会懵逼，Element[]是什么鬼）
##### Element[]是什么鬼
我们在每次创建DexClassLoader时他的构造函数是这样的

```
 public DexClassLoader(String dexPath, String optimizedDirectory,
            String libraryPath, ClassLoader parent) {
        super(dexPath, new File(optimizedDirectory), libraryPath, parent);
    }

```
从源码可以看出第一个参数顾名思义是dex路径，第二个呢可以看看源码,第二个要传一个路径dex优化后odex的路径。第三个呢就是父类吗，直接getClassLoader()就好那么我们看看他的父类拿这些参数干了些什么

```
public BaseDexClassLoader(String dexPath, File optimizedDirectory,
            String libraryPath, ClassLoader parent) {
        super(parent);

        this.originalPath = dexPath;
        this.pathList =
            new DexPathList(this, dexPath, libraryPath, optimizedDirectory);
    }

```
父类也就做了一些初始化操作。最主要的是初始化了DexPathList这个类，然后我们看看BaseDexClassLoader里面的findClass做了些什么呢源码如下

```
  @Override
protected Class<?> findClass(String name) throws ClassNotFoundException {
        Class clazz = pathList.findClass(name);

        if (clazz == null) {
            throw new ClassNotFoundException(name);
        }

        return clazz;
    }

```
看到了吗通过我们构造函数初始化的DexPathList来查找的，上面我们已经贴了DexPathList内部findclass的他是通过Element[]来拿到的。接下来我们来看看DexPathList的构造函数

```
public DexPathList(ClassLoader definingContext, String dexPath,
            String libraryPath, File optimizedDirectory) {
        if (definingContext == null) {
            throw new NullPointerException("definingContext == null");
        }

        if (dexPath == null) {
            throw new NullPointerException("dexPath == null");
        }

        if (optimizedDirectory != null) {
            if (!optimizedDirectory.exists())  {
                throw new IllegalArgumentException(
                        "optimizedDirectory doesn't exist: "
                        + optimizedDirectory);
            }

            if (!(optimizedDirectory.canRead()
                            && optimizedDirectory.canWrite())) {
                throw new IllegalArgumentException(
                        "optimizedDirectory not readable/writable: "
                        + optimizedDirectory);
            }
        }

        this.definingContext = definingContext;
        this.dexElements =
            makeDexElements(splitDexPath(dexPath), optimizedDirectory);
        this.nativeLibraryDirectories = splitLibraryPath(libraryPath);
    }
```
好了这就是他的源码了可看到在够着函数中有一个很重要的一步就是对（makeDexElements这个方法）Element[]初始化 说了这么多终于到这个地方了 这是什么鬼，进入这个方法来看一下

```
private static Element[] makeDexElements(ArrayList<File> files,
            File optimizedDirectory) {
        ArrayList<Element> elements = new ArrayList<Element>();

        /*
         * Open all files and load the (direct or contained) dex files
         * up front.
         */
        for (File file : files) {
            ZipFile zip = null;
            DexFile dex = null;
            String name = file.getName();

            if (name.endsWith(DEX_SUFFIX)) {
                // Raw dex file (not inside a zip/jar).
                try {
                    dex = loadDexFile(file, optimizedDirectory);
                } catch (IOException ex) {
                    System.logE("Unable to load dex file: " + file, ex);
                }
            } else if (name.endsWith(APK_SUFFIX) || name.endsWith(JAR_SUFFIX)
                    || name.endsWith(ZIP_SUFFIX)) {
                try {
                    zip = new ZipFile(file);
                } catch (IOException ex) {
                    /*
                     * Note: ZipException (a subclass of IOException)
                     * might get thrown by the ZipFile constructor
                     * (e.g. if the file isn't actually a zip/jar
                     * file).
                     */
                    System.logE("Unable to open zip file: " + file, ex);
                }

                try {
                    dex = loadDexFile(file, optimizedDirectory);
                } catch (IOException ignored) {
                    /*
                     * IOException might get thrown "legitimately" by
                     * the DexFile constructor if the zip file turns
                     * out to be resource-only (that is, no
                     * classes.dex file in it). Safe to just ignore
                     * the exception here, and let dex == null.
                     */
                }
            } else {
                System.logW("Unknown file type for: " + file);
            }

            if ((zip != null) || (dex != null)) {
                elements.add(new Element(file, zip, dex));
            }
        }

        return elements.toArray(new Element[elements.size()]);
    }

```
代码有点多哈...不要着急 我来慢慢说，首选呢构造函数传进来了一个file数组不论是jar文件还是apk文件我们在这一步都是吧他们转换成dex文件相当于做了一个操作把patch.jar 改成patch.dex然后转存到最初我们传进来的那个optimizedDirectory文件夹下。然后我们的Element这个类是个静态内部类，可以看看下面的源码的构造函数
```
 public Element(File file, ZipFile zipFile, DexFile dexFile) {
            this.file = file;
            this.zipFile = zipFile;
            this.dexFile = dexFile;
        }

```
可以看到他传入了这些参数。好了现在知道了这是什么鬼了吧，一系列的源码恐怕会看的头晕脑胀的吧。反正知道了Element就是存储我们dex文件的每次findclass的时候从这里面取得，知道这个就行了。

#### 插入我们需要的加载的patch.dex
现在知道往哪里插入我们的dex文件了吧，只要在我们app启动的时候，把我们的dex文件加载到Element[]数组最前面就行了，每次findclass的时候肯定先查找我们的dex了。这样不就可以达到热修复了吗！
- 第一步创建一个我们的DexClassLoader 把我们的patch.dex（或.jar）文件传进去
- 第二步通过反射拿到我们创建的DexClassLoader里面的DexPathList里面的Element[]
- 拿到apk的DexClassLoader（getClassLoader（）这个方法就可以拿到）然后同样反射的方式拿到DexPathList里面的Element[]。
- 最关键的一部就是把我们patch的Element[]和apk的Element[]合并在一起然后通过反射修改apk里面的Element[]（别合并错了，要把我们的数据插入最前面）
- 以上步骤要在Application生命周期中的attachBaseContext进行执行不然，在onCreate里面执行的话app就已经初始化好了
这里我就用Nuva热修复的代码来举例吧，他这边写的很详细的 [git传送门](https://github.com/jasonross/Nuwa)哈哈哈博主已弃坑 放弃维护了

```
 DexClassLoader dexClassLoader = new DexClassLoader(dexPath, defaultDexOptPath, dexPath, getPathClassLoader());
        Object baseDexElements = getDexElements(getPathList(getPathClassLoader()));
        Object newDexElements = getDexElements(getPathList(dexClassLoader));
        Object allDexElements = combineArray(newDexElements, baseDexElements);
        Object pathList = getPathList(getPathClassLoader());
        ReflectionUtils.setField(pathList, pathList.getClass(), "dexElements", allDexElements);

```
这就是我所说的那四步。

大功告成是不是很带劲吧。md终于把我们的dex文件插入了。嘿嘿嘿黑科技啊，热修复原来如此简单。别高兴的太早。接下来重点来了

#### 坑1（CLASS_ISPREVERIFIED）预定义
这时候你运行项目的时候会发现app 挂了 哈哈哈 真是日了狗了，不出意外的话会报一下错误class ref in pre-verified class resolved to unexpected implementation 这个就是上面所说的odex操作带来的麻烦。
出问题吗？当然要慢慢解决了。先了解一下odex吧
- 在apk安装的时候系统会将dex文件优化成odex文件，在优化的过程中会涉及一个预校验的过程
- 如果一个类的static方法，private方法，override方法以及构造函数中引用了其他类，而且这些类都属于同一个dex文件，此时该类就会被打上CLASS_ISPREVERIFIED
- 如果在运行时被打上CLASS_ISPREVERIFIED的类引用了其他dex的类，就会报错
- 所以你的类中引用另一个dex的类就会出现上文中的问题
- 正常的分包方案会保证相关类被打入同一个dex文件
- 想要使得patch可以被正常加载，就必须保证类不会被打上CLASS_ISPREVERIFIED标记。而要实现这个目的就必须要在分完包后的class中植入对其他dex文件中类的引用
- 要在已经编译完成后的类中植入对其他类的引用，就需要操作字节码，惯用的方案是插桩。常见的工具有javaassist，asm等

这时候大家就要了解这个了javaassist，一个代码植入库，几个简单的api大家看看都会

```
 /**
     * 植入代码
     * @param buildDir 是项目的build class目录,就是我们需要注入的class所在地
     * @param lib 这个是hackdex的目录,就是AntilazyLoad类的class文件所在地
     */
    public static void process(String buildDir, String lib) {
        System.out.println(buildDir)
        println(lib);
        ClassPool classes = ClassPool.getDefault()
        classes.appendClassPath(buildDir)
        classes.appendClassPath(lib)

        // 将需要关联的类的构造方法中插入引用代码
        CtClass c = classes.getCtClass("cn.jiajixin.nuwasample.Hello.Hello")
        if (c.isFrozen()) {
            c.defrost()
        }
        println("====添加构造方法====")
        def constructor = c.getConstructors()[0];
        constructor.insertBefore("System.out.println(com.cuieney.hookdex.AntilazyLoad.class);")
        c.writeFile(buildDir)


    }

```

如果我们想不被打上标记就只能这样了，就是通过这个方法，让现在这个类Hello在当dex里面引用其他的dex文件里面的AntilazyLoad.class简单的说就是对其他dex文件有依赖就不会被打上标记。

那么我们这个代码段改在哪里运行呢。好问题！！！这也是重点。不知道老铁们还记得上面的代码吗。apk的生成过程的生命周期，就是在build的是那几个步骤。我们需要在.class文件编程成.dex文件前 进行代码植入。这样是不是很完美呢。那我们从哪里下手呢。当然是我们的build.gradle文件下手，我们编译项目的时候每次是不是都是在这里进行操作的

这里要用到一个新的姿势哦（不对是知识哈哈哈）groovy这个语言plugin插件语言。我们原生的android studio 是对groovy支持的。在我们的项目中创建一个***buildsrc***项目，一定要这个名字。然后我们在项目中创建一个类patch.groovy 目录结构如下不用的都删了。
![Screen Shot 2017-06-27 at 11.23.21 AM.png](http://upload-images.jianshu.io/upload_images/3415839-2174e1b59971e1cc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后我们在我们的app的build.gradle里面做一下操作

```
task ('processWithJavassist') << {
    String classPath = file('build/intermediates/classes/debug')//项目编译class所在目录
    com.cuieney.groovy.PatchClass.process(classPath, project(':hookdex').buildDir
            .absolutePath + '/intermediates/classes/debug')//第二个参数是hackdex的class所在目录

}

```
这个是执行代码植入操作project(':hookdex')这个使我们植入的类的module
但是我么这个task你得保证在.class 到 .dex文件之间操作，我们怎么保证呢？接下来见证奇迹的时候到了在build.g里在添加如下代码

```
applicationVariants.all { variant ->
        variant.dex.dependsOn << processWithJavassist //在执行dx命令之前将代码打入到class中
    }

```

这样就完成了我们的代码植入操作  哈哈哈哈 牛逼不牛逼不

but  你在植入代码之前一定要把我们的植入的类的dex提前插入到Element[]里面不然 会报找不到这个类的。 然后在只要真正的patch.dex 我们的补丁。

#### 补丁制作

- 将class文件打入一个jar包中 jar cvf path.jar xxxx.java
- 将jar包转换成dex的jar包 dx --dex --output=path_dex.jar path.jar
- 用adb将你的path_dex.jar push到你的dexpath中。每次app启动吧这个补丁打入就好


#### 坑2 以上代码植入在高版本的gradle不行
包以下错误Gradle1.40 里TransformAPI无法打包的情况，只兼容Gradle1.3-Gradle2.1.0版本
哈哈哈我也没则，目前RocooFix这个项目博主通过一种新的方式进行了代码植入(之前我们通过植入代码来完成避免打上标志，他则是反其道而行，PatchClassLoader每次加载apk里面的dex时，把标志去了这样也可以防止出现之前那种crash 只能说牛逼牛逼，里面代码还在研究...)有兴趣的同学可以看一下。

#### ending
说了这么多，其实网上这种帖子很多，自己只是想系统的整理一下，其实在这个过程自己学到了很多，不论是源码还是各方面的扩展知识吧，对自己都有很大的提升，不论老铁们看没看玩，希望这次分享给大家带来的知识的提升。 stay hungry stay foolish

下一篇文章
[手把手教你写热修复（HOTFIX）](https://juejin.im/post/595d02d5f265da6c375a90bf)

—— Cuieney 后记于 2017.03


