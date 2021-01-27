---
title: 提取LOL地图资源
date:
updated:
categories:
  - - 其他
    - 实用工具
keywords:
top_img:
cover: https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20201031214838.png
aplayer:
---
<meta name="referrer" content="no-referrer" />

### 前言

去年发布了一个提取LOL资源的视频，感谢大家喜爱，但是我发现很多朋友并不满足于视频中提到的那些，还想提取地图之类的资源，所以我干脆从零开始教大家提取**图片，声音，模型，地图**

### 环境准备

先下载工具

- Obsidian，WAD资源转换工具，用于分析WAD文件，导出我们认识的资源类型：[https://github.com/Crauzer/Obsidian](https://github.com/Crauzer/Obsidian)
- LoL-MAPGEO-Converter，地图导出工具：[https://github.com/FrankTheBoxMonster/LoL-MAPGEO-Converter](https://github.com/FrankTheBoxMonster/LoL-MAPGEO-Converter)
- lol2dae，用于导出模型：[https://sourceforge.net/projects/lol2dae/](https://sourceforge.net/projects/lol2dae/)

### Obsidian使用教程

先使用

![image-20201031220303105](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20201031220303.png)

打开一个WAD文件，然后我们发现是乱码，所以我们需要生成他的Hashtable来映射文件名和目录结构

![image-20201031214832271](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20201031214838.png)

并且选择我们刚刚导入的WAD文件

然后我们导入刚刚导出的Hashtable文本文件

![image-20201031220451659](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20201031220451.png)

我们就可以看到还原后的内容了

![image-20201031220534850](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20201031220534.png)

然后即可导出其中的资源

![image-20201031220553605](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20201031220553.png)

### 提取地图

LOL地图基本都位于

> 英雄联盟\Game\DATA\FINAL\Maps\Shipping

我们通过上面的方式导出后资源后，将一对mapgeo和bin文件拖到我们下载的LoL-MAPGEO-Converter.exe上面

![image-20201031220753631](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20201031220753.png)

如果需要导出的地图自带贴图，参考：[https://github.com/FrankTheBoxMonster/LoL-MAPGEO-Converter/issues/5](https://github.com/FrankTheBoxMonster/LoL-MAPGEO-Converter/issues/5)

### 提取模型

![image-20201031221146428](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20201031221146.png)
