---
title: GameFramework篇：资源热更新讲解
date:
updated:
tags:
categories:
keywords:
top_img:
cover:
aplayer:
---
<meta name="referrer" content="no-referrer" />

##  资源热更新步骤

  **准备工作：**

 **StarForce dev/Update分支 [https://github.com/EllanJiang/StarForce/tree/dev/Update](https://github.com/EllanJiang/StarForce/tree/dev/Update) 注意下载子库**
 ![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/ru1.png)

 ** HFS的下载，作为文件服务器 [http://www.rejetto.com/hfs/?f=dl](http://www.rejetto.com/hfs/?f=dl)**

 **在这个分支里，E大已经写好了完整的更新流程**
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/ru2.png)

 **具体细节就是**

 **ProcedureCheckVersion流程**

 **声明更新版本更新配置表回调函数**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/ru3.png)

 **订阅添加WebRequest任务请求的成功与失败事件**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/ru4.png)

 **添加WebRequest任务请求**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/ru5.png)

 **添加WebRequest任务请求成功，执行订阅的事件，反序列化为VersionInfo类（包含版本信息以及更新细节），如果为强制更新，就跳转到下载网页下载船新客户端（你没有体验过得船新版本），如果为热更新，就配置下载文件地址，校验本地客户端和下载到的版本资源列表版本号，如果一致，说明不用更新，直接进入下一流程，如果不一致，说明需要更新版本资源列表，更新成功后，进入下一流程。**
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/ru6.png)
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/ru7.png)
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/ru8.png)

 **ProcedureUpdateResource流程**

 **初始化相关数据，检查资源，并设置回调函数**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/ru9.png)

 **设置更新数量和更新的zip文件长度，如果不需要更新进入下一流程，如果需要更新，先判断用户是否处于移动网络，如果是弹出对话框** **询问是否更新，如果不是（说明在用WIFI咯），就直接进行更新**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/ru10.png)

 ** 开始更新资源，并安排好进度条**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/ru11.png)

 **更新完毕，进入下一流程**

**先来捋一遍思路**

 **首先，Full目录下的资源是Unity打出来的AssetBundle经过加密（如果选择了Load from memory and Decrypt或quick Decrypt）和Zip压缩（如果选择了Zip压缩）后，加上资源的hashcode在加上了一个dat后缀生成的。其中version.dat中记录了每个AssetBundle的原始大小，原始HashCode，压缩成Zip后的大小及Zip包的HashCode，你本地的AssetBundle的HashCode与服务器上的进行对比，如果不一致就需要更新，Zip包的HashCode是干啥的呢，因为你从服务器上下的资源是Zip压缩后的，资源下载下来后，需要计算你下载下来的这个资源的HashCode与服务器上这个资源的ZipHashCode做一个对比，不一致就说明你下载出错了，这个资源不能用。然后解压后，也需要将解压后计算得到的HashCode与服务器上的对比，ZipHashCode和解压后的HashCode都一致这个AssetBundle才能用。**

 **version.hashcode.dat这个文件是放在服务器上的，做资源更新时，你首先就好获取这个文件，那么这个文件你怎么校验呢，这就到了BuildLog.txt（`他在打包路径的BuildReport文件夹里`）这个文件。**

 **GameEntry.Config.BuildInfo.CheckVersionUrl是在Assets\GameMain\Configs\BuildInfo.txt中设置的，CheckVersionUrl指向版本信息配置表version.txt，是由BuildLog.txt来的，于是乎我们也创建这样一个version.txt放到我们自己的服务器上。**

 **因为这个文件的URL是从Assets\GameMain\Configs\BuildInfo.txt中获取的，所以这个文件也要改，把CheckVersionUrl指向你自己服务器上的version.txt。**

### 具体操作

 **1.打AB包，比当前版本高一版本即可（默认为1，但是我打过一次了，所以当前版本为2，所以我要打版本为3的AB包）**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/ru12.png)

 **2.添加AB文件到HFS**
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/QQ截图20191102132436.png)

 **4.配置文件**
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/QQ截图20191102132046.png)

 **新建version.txt文件，注意VersionListLength，VersionListHashCode，VersionListZipLength，VersionZipHashCode要对上BuildLog.txt的数值**
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/QQ截图20191102132156.png)
 **拖到HFS**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/ru17.png)

 **更改BuildInfo.txt，关键是CheckVersionUrl要配置好，其余的随意**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/ry18.png)

 **OK，运行游戏**
 **参考：QQ:471812771（李春）**
### 视频教程
[https://www.bilibili.com/video/av71419528/?p=5](https://www.bilibili.com/video/av71419528/?p=5)
