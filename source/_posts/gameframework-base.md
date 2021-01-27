---
title: GameFramework篇：框架基本理解和源码下载
date:
updated:
tags: [Unity技术, 游戏框架, GameFramework]
categories:
  - - 游戏引擎
    - Unity
  - - GamePlay
    - 游戏框架
cover: https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190105164355598.png
aplayer:
---
<meta name="referrer" content="no-referrer" />

### 先偷偷观察一下[E神的GitHub](https://github.com/EllanJiang)

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20181105212731283.png)

 只有三个项目，尽管如此，我还是感受到来自E神的压迫感。

 我们再来看一下官网的描述

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20181105212603747.png)

### 什么个意思呢？

### 我们先看UGF

 可以看到有GameFramework的dll文件，那么它是从哪来的呢？

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20181105213554140-1.png)

### 它来自GameFramework

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20181105213704969.png)


### 再看UGF的Scripts，里面就是**UnityGameFramework.Runtime和UnityGameFramework.Editor模块啦**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20181105213941403.png)

### 我们可以这样理解，GameFramework是完全和Unity分离的的，由它衍生出与Unity相交互的UnityGameFramework，这也是我们在Unity中使用GameFramework框架必不可少的部分，我们可以把UGF当成平时使用的插件。最后UGF和Unity相结合，就做出游戏啦~

### 然后就是演示Demo——StarForce（一目了然）

 然后我们要把这三个项目都下载

 为什么全都要呢，因为我们平时学习肯定要顺藤摸瓜来看代码实现而，GF是GF.dll的来源，也就是最终源码，相对于反编译dll，我更习惯于直接看源码~所以三个都不能少

 打开StarForce项目，如果有报错的话是因为![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20181229110914833.png)没有被下载

 **如果使用git下载的StarForce工程**

 **当使用git clone下来的工程中带有submodule时，初始的时候，submodule的内容并不会自动下载下来的，此时，只需执行如下命令：**

 **git submodule update --init --recursive**

 **即可将子模块内容下载下来后工程才不会缺少相应的文件。**

 **如果直接下载的StarForce工程的zip**

 **就把我们下载的UGF直接拖进Assets文件夹下，点击进入GameFramework场景，开始游戏！
