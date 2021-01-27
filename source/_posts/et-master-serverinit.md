---
title: ET篇：master服务端初始化流程的介绍
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
cover: https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190408145412332.png
aplayer:
---
<meta name="referrer" content="no-referrer" />

  **讲过master客户端的框架初始化，再来说一下服务端的初始化流程吧，因为很多客户端代码在服务端是通用的，所以理解起来也不算困难，就像这样**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190408145412332.png)

 **先来到程序入口**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190408145441856.png)
**然后就是框架的初始化了（因为很多细节在客户端那篇讲过了，所以就不在这赘述了，内部实现是一样的）**

```c#
 //添加Model.dll到字典维护
Game.EventSystem.Add(DLLType.Model, typeof(Game).Assembly);
//添加Hotfix.dll到字典维护
Game.EventSystem.Add(DLLType.Hotfix, DllHelper.GetHotfixAssembly());
//添加并获取设置组件的引用
Options options = Game.Scene.AddComponent<OptionComponent, string[]>(args).Options;
//添加并获取初始配置的组件的引用
StartConfig startConfig = Game.Scene.AddComponent<StartConfigComponent, string, int>//判断配置文件是否正确
if (!options.AppType.Is(startConfig.AppType))
{
    Log.Error("命令行参数apptype与配置不一致");
    return;
}
```

```c#
 //增加计时器组件
Game.Scene.AddComponent<TimerComponent>();
//增加OpcodeType组件（是双端通讯协议的重要组成部分）
Game.Scene.AddComponent<OpcodeTypeComponent>();
//增加消息分发组件（确保从服务端收到的消息能正确的送到到接受者手中）
Game.Scene.AddComponent<MessageDispatcherComponent>();
// 根据不同的AppType添加不同的组件
OuterConfig outerConfig = startConfig.GetComponent<OuterConfig>();
InnerConfig innerConfig = startConfig.GetComponent<InnerConfig>();
ClientConfig clientConfig = startConfig.GetComponent<ClientConfig>();
```

 **然后是最为重要的服务端组件的添加 ，这里直接拿****case AppType.AllServer:中的组件举例，因为它几乎包括了所有组件**

```c#
 // 发送普通actor消息
Game.Scene.AddComponent<ActorMessageSenderComponent>();
// 发送location actor消息
Game.Scene.AddComponent<ActorLocationSenderComponent>();
//添加MongoDB组件，处理与服务器的交互
Game.Scene.AddComponent<DBComponent>();
//添加MongoDB代理组件，代理服务端对数据库的操作
Game.Scene.AddComponent<DBProxyComponent>();
// location server需要的组件
Game.Scene.AddComponent<LocationComponent>();
// 访问location server的组件
Game.Scene.AddComponent<LocationProxyComponent>();
// 这两个组件是处理actor消息使用的
Game.Scene.AddComponent<MailboxDispatcherComponent>();
Game.Scene.AddComponent<ActorMessageDispatcherComponent>();
// 内网消息组件
Game.Scene.AddComponent<NetInnerComponent, string>(innerConfig.Address);
// 外网消息组件
Game.Scene.AddComponent<NetOuterComponent, string>(outerConfig.Address);
// manager server组件，用来管理其它进程使用
Game.Scene.AddComponent<AppManagerComponent>();
Game.Scene.AddComponent<RealmGateAddressComponent>();
Game.Scene.AddComponent<GateSessionKeyComponent>();
// 配置管理
Game.Scene.AddComponent<ConfigComponent>();
// recast寻路组件
Game.Scene.AddComponent<PathfindingComponent>();
//添加玩家组件（使用字典维护，可当做抽象化的玩家，处于不同的游戏流程会有不同的身份）
Game.Scene.AddComponent<PlayerComponent>();
//添加单位组件（这是游戏中物体的最小单元，继承自Entity）
Game.Scene.AddComponent<UnitComponent>();
Game.Scene.AddComponent<ConsoleComponent>();
```

 ** 最后通过while循环让整个框架跑起来，线程沉睡一毫秒的作用是防止直接吃满CPU**

```c#
 				while (true)
				{
					try
					{
						Thread.Sleep(1);
						OneThreadSynchronizationContext.Instance.Update();
						Game.EventSystem.Update();
					}
					catch (Exception e)
					{
						Log.Error(e);
					}
				}
```
