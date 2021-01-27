---
title: ET篇：master项目结构梳理
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
cover: https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190314214035631.png
aplayer:
---
<meta name="referrer" content="no-referrer" />

**我发现先把项目结构做个梳理有利于后面的学习，所以就整理了这篇笔记**

 **由于我能力有限，可能有些地方理解的不对，恳请各路大大指正，感激不尽**

 **使用Rider编译器打开项目，并把项目视图调整为Solution**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190314214035631.png)
 下面是更加详细的部分

## 1.Book

没看过ET Book的小伙伴应该去看看，写的非常好，这些是示例代码，不在本篇笔记范围内

## 2.Client

ET客户端代码

### Unity.Editor 
**里面是ET客户端所有的编辑器拓展工具，以后开发的时候我们会经常用到**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190314214339560.png)

- AstarPathfindingProject：`A*`插件的编辑器拓展
- BuildEditor：一键打包工具
- ComponentViewEditor：Component可视化工具
- ExcelExporterEditor：Excel导表工具
- GlobalConfigEditor：全局配置工具
- Helper：编辑器拓展工具中常用的辅助函数
- ILRuntimeHelper：ILRuntime一键绑定代码
- Protoc2CSEditor：从protoc一键生成cs代码
- ReferenceCollectorEditor：引用收集工具
- RsyncEditor：同步工具
- ServerCommandLineEditor：服务端命令行配置工具
- ServerManagerEditor：服务端管理工具

### Unity.Hotfix 
**里面是热更新部分的代码，建议项目中准备热更新的代码放在这里，每次回到Unity，这个项目都会被处理成dll文件放到Res/Code文件夹下面，等待使用**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190314214636964.png)

 **详情**

- **Base 主要为组成ET客户端框架的基础类**
- - **Base.Event 定义事件的种类和接口**
- - **Base.Helper 常用功能辅助类**
- - **Base.Object 包含ET中重要的事件系统，组件系统，对象池等**
- - **Base.Log ET的日志类**
- **Entity 实体类**
- **Module 主要为游戏中的功能模块**
- - **Module.Actor以及Module.ActorLocation ET的Actor消息机制**
- - **Module.Config 配置类**
- - **Module.Demo.UnitConfig 规定Unit的所包含的属性** 
- - **Module.Demo.Helper 为Demo中登录和进入游戏主场景的辅助类** 
- - **Module.Demo.UI 为Demo中的UI相关逻辑** 
- - **Module.Demo目录下的脚本为消息处理类** 
- - **Module.Message 包含由proto文件生成的消息类，消息缓存池，消息分发组件，Opcode组件，消息处理接口类，Session组件** 
- - **Module.UI ET客户端的UI工厂** 
- **Init 将会被Unity.Model所反射调用，主要进行相关组件的生成以及热更层的回调**

### Unity.Model 
**里面的代码不能热更新，通常将游戏中不会变动的部分放在这个项目里**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190314222837384.png)
 **详情**

- **Base 主要为组成ET客户端框架的基础类，我们发现这里的代码有一部分和Unity.Hotfix.Base中的代码有部分是重复的，这是因为使用ILRuntime要尽量避免跨域继承，不然要写适配器什么的，比较麻烦。**
- **Component ET客户端常用基础组件**
- - **Component.Config 所用到的配置格式**
- - **Entity 实体类**
- **Helper 常用功能辅助类**
- **ILBinding 根据Unity.Hotfix所自动生成的代码**
- **Module 游戏中常用功能模块以及游戏主体部分**
- - **Module.Actor以及Moudule.ActorLocation Actor消息机制的基础** 
- - **Module.AssetBundle AB包相关类** 
- - **Module.Config 配置类** 
- - **CoroutineLock 协程锁相关类**
- - **Module.Demo Demo相关类** 
- - - **Module.Demo.Config 游戏中Unit配置类** 
- - - **Module.Demo.UI 游戏中UI的相关类，与Unity.HotFix不同，这里主要是对UI的操作，如显示，隐藏，增加组件等** 
- - - **Module.Demo的其余脚本 为游戏中所用到的组件以及相关类** 
- - **Module.Message 包括网络传输协议以及各种消息传输，分发组件，辅助类，可以说是一大核心** 
- - **Module.Numeric 数值组件** 
- - **Module.Pathfinding 寻路组件** 
- - **Module.UI UI相关**
- **Other ****其余定义类**

**Init 对框架进行初始化，增添必须的组件，启动框架**

### Unity.ThirdParty 用到的第三方插件集合

- Google.Protobuf 消息传输载体
- ILRuntime：代码热更新库
- MongoDB：MongoDB数据库相关，这里主要是Bson库

## 3.Server :ET服务端代码

### ThirdParty 服务端用到的第三方库

- Google.Protobuf 消息传输载体
- MongoDBDriver MongoDB数据库驱动
- KcpLib KCP协议库

### APP 服务端启动入口

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190315171309899.png)

### Hotfix 服务端热更新代码

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/2019031518444978.png)

- **Handler 服务端基础的事件处理者，包括重载Hotfix.dll实现不停机更新，ping请求等**
- **Helper 热更新辅助类，目前里面只有一个HotfixHelper，用来依据Hotfix.dll来创建实例**
- **Module 功能模块**
- - **Module.Actor以及Module.ActorLocation Actor消息机制** 
- - **Module.Benchmark 用来做压力测试的** 
- - **Module.Config 配置组件，和配置相关的功能都在这** 
- - **Module.DB MongoDB数据库相关功能，包括数据保存与查询等** 
- - **Module.Demo masterDemo的相关逻辑，主要为事件处理类，消息类** 
- - **Module.Http Http测试类** 
- - **Module.Message 消息组件和系统** 
- - **Scene Scene类，用以动态创建进程**

### Model 服务端代码（不能热更）

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190315190103545.png)

- **Base 服务器基础类，很大部分的代码是引用的Unity.Model中的代码，代码共用，所以下面说一下不同于Unity.Model的代码**
- - **Base.Logger 和NLog有关的Log类** 
- - **Base.UnityEngine Unity的相关函数（主要是数学上的）**
- **Component 服务器组件类，同客户端一样，也是组件式编程**
- **Entity 实体类**
- **Module 功能模块，与Server.Hotfix中的结构大抵类似，但代码不同，注意查看**
- - **Module.DB MongoDB相关组件，主要是配置以及缓存数据用的** 
- - **Module.Demo 大部分为Demo中所用到的组件，包括移动，寻路，Session组件等** 
- - **Module.Http Http组件**
- - **Module.Message 包含了消息池，网络组件，消息信息以及处理类（大多引用自Unity.Model和Unity.Hotfix）** 
- - **Module.Pathfinding 寻路组件**
- **Other 其余配置**
