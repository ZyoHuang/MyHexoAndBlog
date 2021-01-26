---
title: ET篇：master客户端初始化流程的介绍
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

  **要学习一个Demo，不能漫无目的的乱看，看到哪是哪，要有条理，我习惯于从程序入口开始**

 **进入Init场景**

 **发现貌似只有一个Init脚本发挥着作用**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190331171408105.png)

 **那我们就进入Init脚本**

 **这个便是入口了，他调用了StartAsync()函数，至于为什么后面加上Coroutine()（Coroutine是个空函数，并没有什么具体的作用）**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190331171611341.png)

 **熊猫大大说是为了防止这种情况（好吧。。。）**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190331171759469.png)

 ** 接下来我一步一步的解读Init函数里面的语句**

```c#
 //把所有的异步回调都放到主线程执行
SynchronizationContext.SetSynchronizationContext(OneThreadSynchronizationContext.Instance);
```

 **这个如果不理解的，应该去看看我前面的ETBook笔记**

```c#
 //使当前游戏物体不会被销毁
DontDestroyOnLoad(gameObject);
//添加ETModel.dll，并解析维护当中的特性
Game.EventSystem.Add(DLLType.Model, typeof(Init).Assembly);
```

 **第一句没什么好说的**

 **我们来看第二句做了什么**

 **好了，看完了（看官：？？？脑瘫博主，关注了）**

 **我来总结一下**

1. **将程序集交管给字典维护** 
2. **对于每个程序集的每个类型，都会尝试得到自定义特性，并放到types里面维护** 
3. **清空生命周期系统中所有维护的类型** 
4. **解析types中的属于组件生命周期的特性，并放到相应的字典进行维护** 
5. **解析types中的属于事件系统的特性，并放到相应的字典进行维护** **接下来是添加重要的组件，组件的功能细节实现大家可以自己进去看看**

```c#
 //添加计时器组件
Game.Scene.AddComponent<TimerComponent>();
//添加全局配置组件（主要是AB包服务器地址）
Game.Scene.AddComponent<GlobalConfigComponent>();
//添加外网通讯组件（用于发送和接收消息）
Game.Scene.AddComponent<NetOuterComponent>();
//添加资源组件（用于加载资源）
Game.Scene.AddComponent<ResourcesComponent>();
//添加玩家组件（使用字典维护，可当做抽象化的玩家，处于不同的游戏流程会有不同的身份）
Game.Scene.AddComponent<PlayerComponent>();
//添加单位组件（这是游戏中物体的最小单元，继承自Entity）
Game.Scene.AddComponent<UnitComponent>();
//添加UI组件，管理UI
Game.Scene.AddComponent<UIComponent>();
```

 **注意，每个被Add的组件，都会执行其Awake（前提是他有类似的方法），这也是ETBook中的内容，不懂的同学回去补课哦**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190331194350630.png)

 **该添加的组件都添加好了，框架大致没问题，那么就该下载更新资源了 **

```c#
 // 下载ab包
await BundleHelper.DownloadBundle();
```

 **资源下载完成后，那就去加载具体的更新代码吧，顺便插一句，热更的代码已经被打成dll，当做资源下载**

 **被打成ab包的资源全在这哦**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190331195459617.png)

```c#
 //加载热更新程序集
Game.Hotfix.LoadHotfixAssembly();
```

 **我们再来看一下这里面的实现**

 **好了，看完了，我再来总结一下（看官：？？？）**

1. **加载资源** 
2. **读取Hotfix.dll** 
3. **根据当前模式来决定使用Hotfix.dll的方式，看不懂这部分代码的，需要去补习ILRuntime哦** 
4. **卸载资源** 

```c#
 //加载配置文件
Game.Scene.GetComponent<ResourcesComponent>().LoadBundle("config.unity3d");
//添加配置组件，来读取并部署配置文件
Game.Scene.AddComponent<ConfigComponent>();
//卸载配置文件
Game.Scene.GetComponent<ResourcesComponent>().UnloadBundle("config.unity3d");
//添加OpcodeType组件（是双端通讯协议的重要组成部分）
Game.Scene.AddComponent<OpcodeTypeComponent>();
//添加消息分发组件（确保从服务端收到的消息能正确的送到到接受者手中）
Game.Scene.AddComponent<MessageDispatcherComponent>();
//执行HotFix程序集中的函数（在Game.Hotfix.LoadHotfixAssembly();已经设置好）
Game.Hotfix.GotoHotfix();
//测试
Game.EventSystem.Run(EventIdType.TestHotfixSubscribMonoEvent, "TestHotfixSubscribMonoEvent");
```

 **到这里，万事俱备**

 **然后利用Unity生命周期函数，整个框架就跑起来了**

```c#
 		private void Update()
		{
			OneThreadSynchronizationContext.Instance.Update();
			Game.Hotfix.Update?.Invoke();
			Game.EventSystem.Update();
		}

		private void LateUpdate()
		{
			Game.Hotfix.LateUpdate?.Invoke();
			Game.EventSystem.LateUpdate();
		}

		private void OnApplicationQuit()
		{
			Game.Hotfix.OnApplicationQuit?.Invoke();
			Game.Close();
		}
```
