---
title: ET篇：斗地主的资源工作流
date:
updated:
tags:
categories:
keywords:
top_img:
cover:
katex: true
aplayer:
---
<meta name="referrer" content="no-referrer" />

 **有了master的学习经验，斗地主的学习将不会太多精细化，更多细节大家可以自行查看，本系列文章旨在帮助大家理解整个开发流程**

### **资源划分策略**

 **先来到Asset下的Bundles文件夹，这里是游戏内用到的所有的资源，都被打成ab包，正式发布时将会删除，从资源服务器下载文件**

 **Independent**

- **Code 包含热更新模块的dll文件** 
- **Config 包含客户端的配置文件（连接配置，所用到的玩家，敌人等数据结构）** **UI**

- **Landlords/Altas 包含游戏内用到的所有图集** 
- **Landlords/Content 玩家正式开局游戏内的个人信息** 
- **Landlords/HandCard 玩家手牌UI** 
- **Landlords/LandlordsEnd 游戏结束界面** 
- **Landlords/Interaction 玩家的正式游戏内操作界面（出牌，不要，抢地主，不抢等）** 
- **Landlords/LandlordsLobby 游戏大厅** 
- **Landlords/LandlordsLogin 登录界面** 
- **Landlords/LandlordsRoom 正式游戏界面UI** 
- **Landlords/PlayerCard 玩家的单个手牌** 
- **UILoddy/UILogin 没有用到的master的版本资源** **Unit 没有用到的小骷髅资源**

### **资源打包策略**

 **每个资源都被划分在不用的xxx.unity3d文件里，可使用ET提供的工具也可以自己手动编写，**

 **如果使用ET提供的tag工具，需要自己制定ab策略，适宜的的ab粒度对于提升项目性能有极大的帮助。**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190421103215343.png)

 **配置好之后选择打包工具**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190421103259827.png)

 **按需求进行打包即可**

### **资源的下载**

 **ET提供了一个资源服务器来供我们方便测试，那么我们正式上线的项目该怎么配置呢**

 **这里做个示例**

 **先下载HFS，这能帮助我们极快的建立一个资源服务器**

 **打开之后将这个地址放到全局配置里**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190421155125780.png)

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190421155137857.png)

 **按下图进行打包，打好的ab包将输出在Release下面**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190421151224491.png)

 **这里只有一个PC**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190421155213311.png)

 **记住这个位置**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190421155236311.png)

 **选中刚刚的PC文件夹，添加**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190421155254562.png)

 **然后运行打包好的exe，成功**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190421155442833.png)

### **资源的加载使用步骤**

 **加载使用资源的时候写类似的如下代码**

```
                 //加载AB包
                ResourcesComponent resourcesComponent = ETModel.Game.Scene.GetComponent<ResourcesComponent>();
                resourcesComponent.LoadBundle($"{type}.unity3d");

                //加载登录界面预设并生成实例
                GameObject bundleGameObject = (GameObject)resourcesComponent.GetAsset($"{type}.unity3d", $"{type}");
                GameObject login = UnityEngine.Object.Instantiate(bundleGameObject);
```
