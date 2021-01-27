---
title: GameFramework篇：Setting（存档模块）使用示例
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

# 存档模块

## 存档模块的简单使用
  **GF自带了Setting模块,以键值对的形式存储玩家数据，对 UnityEngine.PlayerPrefs 进行封装。**

 **主要的API在DefaultSettingHelper里面**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190120205333502.png) ![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190120205353852.png) ![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190120205300952.png) ![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190120204729862.png)

 **其中Load功能是个空方法,并没有写,需要大家自己去写,一想也是,每个项目数据结构都不一样**

 **下面是个简单的例子**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190120210223320.png) ![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190120210308572.png) ![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190120210244395.png)

 **然后大家根据自己项目需要再对此模块进行封装即可**
