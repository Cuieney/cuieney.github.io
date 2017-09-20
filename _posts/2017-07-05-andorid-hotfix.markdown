---
layout:     post
title:      "手把手教你写热修复（HOTFIX）"
subtitle:   " \"Hello World, Hello Android\""
date:       2016-12-16 13:00:00
author:     "Cuieney"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - HotFix
---

> “Yeah It's on. ”


## 前提
>写这篇文章的目的呢，也是理一下自己的思路吧，同时把最近看到的一些热修复知识献给读者们。不知道同学们最近是不是听到了很多关于热修复的事情，各大厂商，各界大佬们都有属于自己的热修复框架，最近阿里不也推出了个爆炸消息，堪称最牛逼的修复框架Sophix，同时还推出了对应的一本pdf（叫什么深入理解Android热修复技术原理），不知道多少同学看过，深入看应该是可以看到个原理，但是我感觉看了我也写不出这样的代码，毕竟大厂大佬。这篇文章呢，就简单的教大家如何写一个属于自己公司或者自己的热修复框架

#### 友情提醒
1.这篇文章的重点在于.class文件的打桩，可能会偏重于groovy语言，与java相通没事，相信我你绝对能看懂。

2.如果没有看过我之前的那篇文章可能会有些懵哦，之前的那篇文章讲的是原理，通过DexClassLoader如何热修复。

3.因为热修复关键点还是在于打桩生成差异文件的dex，而不是在于把这个dex文件插入到已安装的app中（两个相辅相成（打桩和插入dex）），因为google的multidex里面已经把这个操作做的很好了，我们只需要修修改改就可以完成这个插入操作，还有就是之前那篇文章还留了一个坑。

4.如果没有接触过热修复的同学可能会对下面的一些词汇比较闷（`打桩`修改字节码文件（.class）打桩目的为了解决CLASS_ISPREVERIFIED预定义，不明白的可参考上一篇文章）

5.附文章传送门及这篇文章的项目源码

[DexClassLoader热修复的入门到放弃](http://www.jianshu.com/p/9594a48ca0d6)

[AutoFix](https://github.com/Cuieney/AutoFix)欢迎star，fork，issue


### 小节
- 如何正确的打桩避免Gradle1.4以上Transform API导致的无法打包（解决之前文章的坑）
- 如何在编译成dex文件前进行打桩
- 如何打桩
- 如何区分差异文件及正确的打包出patch.jar(只对修改后的文件进行打包)


### 解决Gradle1.4以上的Transform问题
因为google的gradle升级了嘛，主要是他引入了transform API(官网解释The API existed in 1.4.0-beta2 but it's been completely revamped in 1.5.0-beta1)，导致我们的plugin找不到之前我们写好的task任务名。

下面我给大家讲一下通过plugin进行打桩操作，接下来我会给大家看一下`nuwa`热修复项目中的部分代码。

```
def preDexTask = project.tasks.findByName("preDex${variant.name.capitalize()}")
def dexTask = project.tasks.findByName("dex${variant.name.capitalize()}")
def proguardTask = project.tasks.findByName("proguard${variant.name.capitalize()}")

```
这是nuwa热修复的源码他事先定义好了这些任务，这些任务就是把字节码文件打包成dex文件的任务，上面的代码意思就是获取这些任务的名字。（就是apply plugin: 'com.android.application'里面的任务）。从上面的代码可以看到，我们定义的任务名称分别是（preDex${variant.name.capitalize()}）（dex${variant.name.capitalize()}）（proguard${variant.name.capitalize()}）（$这个符号就是拼接字符串的意思和kotlin一样，variant.name.capitalize()这个就是获取的字符串是debug 还是release）。然后上面我们也说了，gradle1.4之后google改名字了，我们当然找不到这些任务名了，当然报错了哦。现在呢我们只需要做一些简单的if判断操作不就可以了吗？根据不同的gradle版本号修改一下名字不就得了,下面贴出RocooFix的代码块如何解决的

```

    static String getProGuardTaskName(Project project, BaseVariant variant) {
        if (isUseTransformAPI(project)) {
            return "transformClassesAndResourcesWithProguardFor${variant.name.capitalize()}"
        } else {
            return "proguard${variant.name.capitalize()}"
        }
    }

    static String getPreDexTaskName(Project project, BaseVariant variant) {
        if (isUseTransformAPI(project)) {
            return ""
        } else {
            return "preDex${variant.name.capitalize()}"
        }
    }

    static String getDexTaskName(Project project, BaseVariant variant) {
        if (isUseTransformAPI(project)) {
            return "transformClassesWithDexFor${variant.name.capitalize()}"
        } else {
            return "dex${variant.name.capitalize()}"
        }
    }


```
看到了吗？就是判断一下当前的项目gradle版本号，然后修改一下名称返回给你。简单吧。几行代码解决了兼容性问题。

### 何时打桩
之前那篇文章也说了，apk编译的生命周期。所以这边顾名思义当然是在被打成dex文件之前对class文件的时候操作啊。

接下来问题来了，如何在被打成dex文件前操作呢，刚刚上面说的获取那些task名称还记得吗？这就是关键。因为在groovy中有这么一个语法，任务之间可以通过dependsOn来添加依赖。

那么好现在举个例子。
```
task A{}
task B{}

A dependsOn B

```
很明显吗？就是执行A前必须B执行完了才行。知道了这个吗？我们下面继续看项目源码如何设置在我们打桩完成之后再执行dex操作

```
def autoJarBeforeDexTask = project.tasks[autoJarBeforeDex]

 autoJarBeforeDexTask.dependsOn dexTask.taskDependencies.getDependencies(dexTask)
 autoJarBeforeDexTask.doFirst(prepareClosure)
 autoPatchTask.dependsOn autoJarBeforeDexTask
 dexTask.dependsOn autoPatchTask

```
一下来这么多代码 而且这些代码还不认识可能会有点蒙哦，不要着急，一行一句的讲解给你听他们的依赖关系（简单讲一下上面代码的意思
autoJarBeforeDexTask这个任务就是进行打桩的任务，dexTask这个任务就是打包成dex的任务，prepareClosure初始化操作的，autoPatchTask打包成补丁文件的任务 
）

第一行呢 获取项目中的task 名字是autoJarBeforeDex

第二行呢 先说一下这句话的意思（dexTask.taskDependencies.getDependencies(dexTask) 这句话就是拿到dexTask的依赖任务），然后我们的autoJarBeforeDexTask这个任务对dexTask的依赖任务 进行依赖

第三行呢 autoJarBeforeDexTask 点doFirst(prepareClosure)意思就是说在执行autoJarBeforeDexTask任务前先执行这个prepareClosure

第四行和第五行不用说了吧 

最后逻辑如prepareClosure -> autoJarBeforeDexTask -> autoPatchTask -> dexTask 依次执行

这样操作就解决了在dex文件生成前进行字节码文件的插入

### 如何打桩
首先呢，打桩你的先获取字节码文件吧。就是这些file。如何获取这些文件呢，因为我们编译项目的时候会在项目中生成一个build目录，里面有项目相关的所有文件，我们可以通过下面的方式获取这些文件，我们打桩只需要获取jar文件和intermediates/class下面对于的class文件，代码如下

```
 static Set<File> getDexTaskInputFiles(Project project, BaseVariant variant, Task dexTask) {
        if (dexTask == null) {
            dexTask = project.tasks.findByName(getDexTaskName(project, variant));
        }

        if (isUseTransformAPI(project)) {
            def extensions = [SdkConstants.EXT_JAR] as String[]

            Set<File> files = Sets.newHashSet();

            dexTask.inputs.files.files.each {
                if (it.exists()) {
                    if (it.isDirectory()) {
                        Collection<File> jars = FileUtils.listFiles(it, extensions, true);
                        files.addAll(jars)
                        //intermediates/class下面对应的class文件
                        if (it.absolutePath.toLowerCase().endsWith("intermediates${File.separator}classes${File.separator}${variant.dirName}".toLowerCase())) {
                            files.add(it)
                        }
                        //jar包
                    } else if (it.name.endsWith(SdkConstants.DOT_JAR)) {
                        files.add(it)
                    }
                }
            }
            return files
        } else {
            return dexTask.inputs.files.files;
        }
    }

```
文件这时候我们已经拿到了。然后我们要遍历这些文件依次给他们打桩，同时要过滤掉jar包中不需要打桩的文件不然会耗时


```
    //打桩等一些工作
  def autoJarBeforeDex = "autoJarBeforeDex${variant.name.capitalize()}"
                project.task(autoJarBeforeDex) << {
                    //获取build/intermediates/下的文件
                    Set<File> inputFiles = AutoUtils.getDexTaskInputFiles(project, variant, dexTask)
                    inputFiles.each { inputFile ->
                        def path = inputFile.absolutePath
                        if (path.endsWith(SdkConstants.DOT_JAR)) {
                            //对jar包进行打桩
                            NuwaProcessor.processJar(hashFile,hashMap,inputFile, patchDir, includePackage, excludeClass)
                        } else if (inputFile.isDirectory()) {
                            //intermediates/classes/debug 目录下面需要打桩的class
                            def extensions = [SdkConstants.EXT_CLASS] as String[]
                            //过滤不需要打桩的文件class
                            def inputClasses = FileUtils.listFiles(inputFile, extensions, true);
                            inputClasses.each {
                                inputClassFile ->
                                    def classPath = inputClassFile.absolutePath
                                    //过滤R文件和config文件
                                    if (classPath.endsWith(".class") && !classPath.contains("/R\$") && !classPath.endsWith("/R.class") && !classPath.endsWith("/BuildConfig.class")) {
                                        //引用nuwa而来的
                                        if (NuwaSetUtils.isIncluded(classPath, includePackage)) {
                                            if (!NuwaSetUtils.isExcluded(classPath, excludeClass)) {
                                                def bytes = NuwaProcessor.processClass(inputClassFile)

                                                if ("\\".equals(File.separator)) {
                                                    classPath = classPath.split("${dirName}\\\\")[1]
                                                } else {
                                                    classPath = classPath.split("${dirName}/")[1]
                                                }
                                                def hash = DigestUtils.shaHex(bytes)
                                                hashFile.append(AutoUtils.format(classPath, hash))
                                                //根据hash值来判断当前文件是否为差异文件需要做成patch吗？
                                                if (AutoUtils.notSame(hashMap,classPath, hash)) {
                                                    def file = new File("${patchDir}${File.separator}${classPath}")
                                                    file.getParentFile().mkdirs()
                                                    if (!file.exists()) {
                                                        file.createNewFile()
                                                    }
                                                    FileUtils.writeByteArrayToFile(file, bytes)
                                                }
                                            }

                                        }
                                    }

                            }
                        }
                    }
                }

```
好吧代码有点长，但是每一个关键点都有相应的注释，
上面的代码的意思简单的说就是 先判断文件是jar包还是路径 如果是jar包，进行jar包的打桩方式，如果是路径的话 找到class文件判断这个class是否要打桩（如R文件就不需要）。然后根据文件的hash值来来判断这个类是否修改过，如果修改过吧吧这些类放在一个文件夹中，最后统一打包成补丁。

打桩代码有两处

```
 //对jar包进行打桩 
 NuwaProcessor.processJar(hashFile,hashMap,inputFile, patchDir, includePackage, excludeClass)

```

```
//对文件进行打桩
def bytes = NuwaProcessor.processClass(inputClassFile)

```
这次介绍的打桩用的是asm这个库，具体代码都在项目中可以去看看，这里就不详细说了。

### 如何区分差异文件打包成patch.jar
这个关键点在于项目要在gradle中配置一些信息 有兴趣的同学可以看一下项目里面有集成过程
[AutoFix](https://github.com/Cuieney/AutoFix)

```
auto_fix {
    lastVersion = '1'//需要打补丁的情况下打开此处
}

```
如果细心的同学会发现我们的项目中创建了hashFile这个文件。这个文件是用来记录每个版本的打桩了的字节码文件的hash值。

上面的代码也可以看出需要配置上一次的版本号，你出现bug了肯定要修改你的versionCode 然后把之前的填写到lastVersion上，他会根据上次的hashFile来和这次生成的hashFile进行对比，如果不相同说明这个类被修改过，然后吧这个文件copy已发到patch目录下，打桩完成之后我们，我们到对于的patch目录下会找到这些文件然后把它们打包成patch.jar生成相应的补丁文件。有这么一段代码

```
//根据hash值来判断当前文件是否为差异文件需要做成patch吗？
    if (AutoUtils.notSame(hashMap,classPath, hash)) {
        def file = new File("${patchDir}${File.separator}${classPath}")
        file.getParentFile().mkdirs()
        if (!file.exists()) {
            file.createNewFile()
        }
        FileUtils.writeByteArrayToFile(file, bytes)
    }

```
这就是上面说的意思根据hashMap 和当前的hash值来做判断。最终生成差异文件。打包成补丁文件（问题又来了，如何生成补丁文件呢。还记得之前说的执行任务的流程吗？prepareClosure -> autoJarBeforeDexTask -> autoPatchTask -> dexTask 依次执行）autoPatchTask这个任务就是打补丁任务可以看一下源码

```
//制作patch补丁包
                def autoPatchTaskName = "applyAuto${variant.name.capitalize()}Patch"
                project.task(autoPatchTaskName) << {
                    if (patchDir) {
                        AutoUtils.makeDex(project, patchDir)
                    }
                }
                def autoPatchTask = project.tasks[autoPatchTaskName]

```
没错就是他，还是上一篇文章讲到的打包操作只不过代码话了 具体代码可以去项目中看，这里就不详解了。

### 总结

说了这么多，发现我咋还不会写呢，怎么办，这篇文章说的都是tmd什么打桩，对的你没听错，因为热修复就是`插入dex` 然后就是`打桩` 就这两件事之前的文章讲的是`插入dex`这篇文章讲的是`打桩`，如果在不会可以去看看项目源码。

### ending

有什么疑问和不解可以留言哦，希望这次分享带给大家带来的不是时间的浪费，而是能力的提升。谢谢

源码奉上：[AutoFix](https://github.com/Cuieney/AutoFix)欢迎star，fork，issue

—— Cuieney 后记于 2017.03


