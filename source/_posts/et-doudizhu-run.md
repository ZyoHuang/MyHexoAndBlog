---
title: ET篇：运行斗地主Demo
date:
updated:
tags: [Unity技术, 游戏框架, GameFramework]
categories:
  - - 游戏引擎
    - Unity
  - - GamePlay
    - 游戏框架
keywords:
top_img:
cover: https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190205205414947.png
aplayer:
---
<meta name="referrer" content="no-referrer" />

### 环境准备：

 **下载斗地主Demo**

 **[https://github.com/Viagi/LandlordsCore.git](https://github.com/Viagi/LandlordsCore.git)**

 **准备2017.4.0版本的Unity**

 **[https://unity3d.com/cn/get-unity/download/archive](https://unity3d.com/cn/get-unity/download/archive)**

 **下载并配置MongoDB以及Studio 3T**

 **MongoDB地址：****[https://www.mongodb.com/download-center/community](https://www.mongodb.com/download-center/community)**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/2019020520155736.png)
 **选择Custom**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190205202000178.png)
 **我是在D盘新建了MongoDB文件夹来作为安装目录**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190205202117367.png)
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190205202140611.png)
 **这个可视化工具看个人喜好，我这里没有安装**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190205202158660.png)
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190205202314821.png)

 **找到电脑上的cmd.exe**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190205202438519.png)
 **复制一份到MongoDB安装目录的bin文件夹下面**

 **打开刚刚复制的cmd.exe**

 **执行以下命令(_dbpath后面的路径为你的MongoDB安装路径下的的data文件夹路径)**

```
 mongod -dbpath D:\MongoDB\data
```

 **然后他可能会这样**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190205203724234.png)

 **打开浏览器,输入**** [http://localhost:27017/](http://localhost:27017/)如果显示如下信息,表示连接成功**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190205203755370.png)

### Studio 3T的下载

 **[https://studio3t.com/download-now/#windows](https://studio3t.com/download-now/#windows)**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190205204133406.png)

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190205204422819.png)

 **至此,MongoDB和可视化工具都以及安排完毕**

### 进入斗地主工程

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190205204524878.png)

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190205204600209.png)

 **即**

```
 { "_t" : "StartConfig", "_id" : NumberLong("98547768819754"), "C" : [{ "_t" : "OuterConfig", "Address" : "127.0.0.1:10002", "Address2" : "127.0.0.1:10002" }, { "_t" : "InnerConfig", "Address" : "127.0.0.1:20000" }, { "_t" : "HttpConfig", "Url" : "http://*:8080/", "AppId" : 0, "AppKey" : "", "ManagerSystemUrl" : "" }, { "_t" : "DBConfig", "ConnectionString" : "mongodb://127.0.0.1:27017", "DBName" : "斗地主数据库" }], "AppId" : 1, "AppType" : "AllServer", "ServerIP" : "*" }

```

 **运行服务器**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190205204711208.png)

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190205204726374.png)

 **回到Unity,打包项目**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190205204748412.png)

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190205204809269.png)

 **打开资源服务器(别指望他会显示啥,就是个黑框框)**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190205205040395.png)

 **运行打包好的项目**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/2019020520483723.png)

 **注册登录,就可以运行了**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190205205342691.png)

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190205205414947.png)

 **后面我会继续学习这个斗地主项目,争取早日熟练使用ET,圆自己的网游梦!共勉!
