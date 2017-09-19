---
layout:     post
title:      "新版bintray教你上传JCenter仓库"
subtitle:   " \"Hello World, Hello Android\""
date:       2016-12-16 13:00:00
author:     "Cuieney"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - 手把手教学
---

> “Yeah It's on. ”


<h1 id="">## 前言</h1>

<p>相信看到我这篇文章的人已经踩了不少坑了。新版bintray改版后，按照之前网上的教程出现了很多迷茫。很对东西对应不上。接下来我来说一下对应不上的几点。</p>

<h1 id="bintray">### Bintray账号注册</h1>

<p>首先你的有个 <a href="https://bintray.com" title="Title">bintray</a>账号，如果没有的话注册一个（记得好像163邮箱不能注册的）</p>

<p><img src="http://upload-images.jianshu.io/upload_images/3415839-9576b29e15476c5f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240" alt="Bintray.png" />
这就是新版的Bintray首页，多出了一个organization的tab和之前的有所不同，而且你的创建的仓库入口只能通过这个来创建。而不是通过本身来创建</p>

<h1 id="repository">### 创建一个属于自己的repository</h1>

<p>首页很明显可以看到一个Add New Repository的按钮，打开之后直接输入这个repository的name，和type即可，然后直接点击create 就行了（其他的随意）。
<img src="http://upload-images.jianshu.io/upload_images/3415839-7f760d0c726a6cb7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240" alt="repository.png" /></p>

<h1 id="repositorypackage">### 创建Repository的Package</h1>

<p>这个package主要就是来存放你的library
<img src="http://upload-images.jianshu.io/upload_images/3415839-134e8138f2d76c29.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240" alt="package.png" />
点击Add New Package之后需要输入一些信息
<img src="http://upload-images.jianshu.io/upload_images/3415839-0c298c8e18192ada.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240" alt="package.png" />
这是我的输入的一些信息，大家可以参考，website就是这个library在github上的地址，issue就是git的地址。</p>

<h1 id="studio">### 接下来就是studio的内容（很多坑就在此）</h1>

<p>1.配置Gradle Bintray Plugin，在项目最外层的gradle里面添加</p>

<pre><code>dependencies {
    classpath 'com.android.tools.build:gradle:1.2.3'
    classpath 'com.jfrog.bintray.gradle:gradle-bintray-plugin:1.2'
    classpath 'com.github.dcendents:android-maven-plugin:1.2'
}```

2.接下来修改local.properties。在里面定义api key的用户名以及被创建key的密码，用于bintray的认证。之所以要把这些东西放在这个文件是因为这些信息时比较敏感的，不应该到处分享，包括版本控制里面。
下面是要添加的2行代码：
</code></pre>

<p>bintray.user=YOUR<em>BINTRAY</em>USERNAME
bintray.apikey=YOUR<em>BINTRAY</em>API_KEY</p>

<pre><code>重点重点重点，不然你会出现各种奇葩的问题404 401等什么鬼的
=====
首先这个bintray.user名字是你的用户名而不是organization的名字
![name.png](http://upload-images.jianshu.io/upload_images/3415839-8f1d510315241c29.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
看到了吗就是这个账号的名字，不是下面的cuieneymaster，而是上面的cuieney
====
![organization.png](http://upload-images.jianshu.io/upload_images/3415839-d41bc7757de13a10.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

好了 那接下来就是bintray.apikey在哪里获取呢。看到上面那副图了吗？点击Edit profile。
![profile.png](http://upload-images.jianshu.io/upload_images/3415839-1535fb26cfcbffc1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
看到了把。就是这个API Key 输入用户密码就可以拿到这个key了。好了，坑1已经过去了。
我们继续studio部分

3.配置library里面的gradle内容。
打开gradle里面然后把下面两行代码添加进去
</code></pre>

<p>apply plugin: 'com.github.dcendents.android-maven'
apply plugin: 'com.jfrog.bintray'</p>

<pre><code>别问为什么，按照这样操作没问题
好吧我还是把图贴上吧
![屏幕快照 2016-12-29 上午11.54.10.png](http://upload-images.jianshu.io/upload_images/3415839-0aa456e89ce2e58b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
哦对了，还有个version版本号，就是标记你当前提交的版本号，没什么。记得要写哦，不过下面你也可以直接写在代码里面。

那我们继续，下面还有很多重要的代码要贴的，
</code></pre>

<p>def siteUrl = 'https://github.com/Cuieney/Crossfader'   // 项目的主页
def gitUrl = 'https://github.com/Cuieney/Crossfader.git'   // Git仓库的url
group = "com.cuieneylibrary.crossfader"            // Maven Group ID for the artifact，
install { <br />
      repositories.mavenInstaller { <br />
           // This generates POM.xml with proper parameters <br />
           pom { <br />
               project { <br />
                             packaging 'aar' <br />
                             // Add your description here <br />
                             name 'Let the music to keep up with your tempo' <br />
                             //项目的描述 你可以多写一点 <br />
                             url siteUrl <br />
                             // Set your license <br />
                             licenses { <br />
                                 license { <br />
                                        name 'The Apache Software License, Version 2.0' <br />
                                        url 'http://www.apache.org/licenses/LICENSE-2.0.txt' <br />
                                  } <br />
                             } <br />
                             developers { <br />
                                developer { <br />
                                       id 'cuieney'        //填写的一些基本信息 <br />
                                       name 'cuieney' <br />
                                       email 'cuieney@163.com' <br />
                                }
                             } <br />
                             scm { <br />
                                     connection gitUrl <br />
                                     developerConnection gitUrl <br />
                                     url siteUrl <br />
                             } <br />
                          } <br />
                     } <br />
                }
       }
 task sourcesJar(type: Jar)
 { <br />
    from android.sourceSets.main.java.srcDirs <br />
    classifier = 'sources'
 }
//task javadoc(type: Javadoc){
  //options.encoding = 'UTF-8'
  //source = android.sourceSets.main.java.srcDirs
  // classpath += project.files(android.getBootClasspath().join(File.pathSeparator))
//}
//task javadocJar(type: Jar, dependsOn: javadoc) {
  //    classifier = 'javadoc'
  //    from javadoc.destinationDir
//}
artifacts {
//    archives javadocJar <br />
     archives sourcesJar
}
Properties properties = new Properties()//读取properties的配置信息，当然直接把信息写到代码里也是可以的
properties.load(project.rootProject.file('local.properties').newDataInputStream())bintray { <br />
         user = properties.getProperty("bintray.user") <br />
         key = properties.getProperty("bintray.apikey") <br />
         configurations = ['archives'] <br />
         pkg { <br />
                userOrg="cuieneymaster" <br />
                repo = "test"          //这个应该是传到maven的仓库的 <br />
                name = "crossfader"    //发布的项目名字 <br />
                websiteUrl = siteUrl <br />
                vcsUrl = gitUrl <br />
                licenses = ["Apache-2.0"] <br />
                publish = true <br />
}}</p>

<pre><code>看到了这些代码肯定也是懵逼了。不要问直接粘贴过去，下面我会介绍其中的一些主要的字段到底干嘛的。
在gradle里面两个task被我注释掉了，主要是因为library中有中文不能生成文档，大家如果写测试的话可以释放开，其中上面很对字段需要写出自己的东西，可以改一下
### 重点1
===
</code></pre>

<p>group = "com.cuieneylibrary.crossfader" 
 ```
这句代码改如何替换，这个代码的意思是自己maven库的地址，大家可以自己创建一个属于自己的maven库 <a href="https://issues.sonatype.org/" title="Title">Sonatype</a>点击可以自己注册一个。如果有什么不明白的可以参考
http://www.trinea.cn/dev-tools/upload-java-jar-or-android-aar-to-maven-center-repository/
http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2015/0623/3097.html
这两篇文章。（group 这个字段应该可以随意写，因为我们只是上次到jCenter的，不防试试）</p>

<h1 id="2">### 重点2</h1>

<pre><code>pkg { 
       userOrg="cuieneymaster" 
       repo = "test" //这个应该是传到maven的仓库的 
       name = "crossfader" //发布的项目名字 
       websiteUrl = siteUrl 
       vcsUrl = gitUrl 
       licenses = ["Apache-2.0"] 
       publish = true 
}
</code></pre>

<p>这个地方如果写错你肯定出现404 not found ，主要是因为新版bintray多出个organization所以这里你会看到多出一个字段userOrg，不然你上传jCenter的时候会出现各种错误。上面的userOrg repo name 可以根据我下面的截图来填写
<img src="http://upload-images.jianshu.io/upload_images/3415839-3cd499c6d1a62c98.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240" alt="pkg.png" />
看见了吗上面有个类似于树状图的圆圈
<img src="http://upload-images.jianshu.io/upload_images/3415839-ea0e21602c368764.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240" alt="tree.png" />
对的就根据这个来填写你要上面的三个信息就可以了。说了这么多，到了最后一步了那就是上传到bintray。</p>

<h1 id="3">### 重点3（可忽略）</h1>

<p>在studio 下的shell下输入</p>

<pre><code>./gradlew install
./gradlew bintrayUpload
</code></pre>

<p>如果成功的话你的，恭喜你，你的library已经上传到网上了，全世界人都可以看到了，接下来还需要一步，那就是打开bintray主页，找到你library上传的package，然后点击add to jcenter即可，很快就会上传上去，与老版bintray相比，速度快多了。
<img src="http://upload-images.jianshu.io/upload_images/3415839-4a261b9f1585e6a6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240" alt="jcenter.png" />
上传成功就可以看到在Linked to多出来个jcenter的杯子 。这时候你的library已经上传到了jcenter，这时候你就可以在</p>

<pre><code>dependencies { 
    compile 'com.google.code.gson:gson:2.3.1'
}
</code></pre>

<p>一行代码导入你的库了。
在网页输入<a href="http://jcenter.bintray.com/">http://jcenter.bintray.com</a>
然后拼接上面的你刚刚在gradle里面配置的group 找到属于你的下载地址即可。</p>



—— Cuieney 后记于 2016.12


