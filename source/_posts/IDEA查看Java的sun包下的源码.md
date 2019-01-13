---
title: IDEA查看Java的sun包下的源码
date: 2019-01-04 15:50:43
tags:
    - Java
---

由于JDK的src.zip里面没有sun包的源码，所以要在IDEA里面查看sun包下的源码的时候只能看到通过反编译得到的代码，反编译得到的代码没有注释，而且很多局部变量名都变成了var1，var2这样子，看起来不太舒适。如果只是想随便看一看，可以直接在[github](https://github.com/openjdk-mirror/jdk/tree/jdk8u/jdk8u/master)或者[OpenJDK](http://hg.openjdk.java.net/jdk8u)上查看，但是这样看还是比较麻烦的，因为这样在网页上看不能像在IDE一样点击某个类然后跳转到某个类的源码。那么有没有办法可以直接在IDEA里面查看sun包下的源码，而不是在网页上看呢？您好，可以的。

## 下载源码到本地
使用git把源码clone到本地，然后切换到你想要查看的源码的分支
```bash
git clone https://github.com/openjdk-mirror/jdk.git
cd jdk
git checkout jdk8u/jdk8u/master
```
因为我要看JDK8的sun包的源码，所以我切换到了`jdk8u/jdk8u/master`分支，后面会在这个jdk目录复制里面的文件夹

## 魔改src.zip
IDEA默认配置的源码文件为src.zip，所以只要把上面下载的源码里面的sun包直接复制到src.zip里面就能通过IDEA直接查看源码了。
src.zip在你的JDK目录下。比如在我的电脑中，我把JDK安装到了**/opt/java/jdk1.8.0_192**目录，所以我的src.zip文件就在这个目录下。

![Imgur](https://i.imgur.com/rQ3OAWV.png)

把src.zip拷到其它目录，然后解压，然后把下载的源码的sun文件夹整个拷贝到解压的src.zip的根目录。sun文件夹在前面下载的源码的**src/share/classes**目录下

![Imgur](https://i.imgur.com/RcG47Kp.png)

另外sun包还有一些平台相关的代码，在windows，mac还有linux下的实现是不同的，所以需要根据你的平台来选择复制哪个，比如我是linux平台，我还需要复制的是**src/solaris/classes**目录下的sun文件夹（不复制也可以，因为这些代码在不同的平台的实现是不一样的，基本上只要复制**src/share/classes**目录下的sun文件夹就足够了，这个sun文件夹的代码是几个平台通用的）

![Imgur](https://i.imgur.com/8RGIfZQ.png)

你还可以根据需要复制**src/share/classes**目录下的其它文件夹到解压后的src.zip目录(这个目录下的Java代码实现都是平台无关的)，然后就可以在IDEA查看其它原本的src.zip不提供的源码了。我把这个目录下的全部文件夹都复制进去了，然后还把平台相关的也复制进去了（贪得无厌警告

复制完之后，就把解压后的目录重写打包成zip

![Imgur](https://i.imgur.com/eUxZSbR.png)


## 用魔改后的src.zip覆盖原来的
将魔改后的src.zip复制回原来的目录，覆盖原本的src.zip，比如我的目录是**/opt/java/jdk1.8.0_192**，就把魔改后src.zip复制回这里，覆盖原来的src.zip。然后重启IDEA，就能看到效果了

![Imgur](https://i.imgur.com/mFai5vN.png)

可以看到我的rt.jar包下多了个sun目录

然后在我想要查看`java.net.URLConnection`的具体实现，比如`sun.net.www.protocol.file.FileURLConnection`的时候，就能查看真正的源码，而不是反编译后的代码了

![Imgur](https://i.imgur.com/ufYeq7y.png)

![Imgur](https://i.imgur.com/meOnZJM.png)

体验极佳

