---
title: ET篇：master消息机制介绍
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

   **终于到了核心部分之一了——消息机制**
### 一般消息的流转

 **既然讲到消息，我们要到消息的源头——客户端**

 **来到Unity.Hotfix的Init文件，并定位到这一句**

```c#
 Game.EventSystem.Run(EventIdType.InitSceneStart);
```

 **他会执行这里的Run代码（不知道为何的去复习ETBook的EventSystem哦）**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/QQ截图20190429175655.png)

 **进入UILoginFactory，他是登录UI的工厂类，在这个工厂将完成一系列的操作**

```c#
 //获取资源组件，用以加载资源
ResourcesComponent resourcesComponent = ETModel.Game.Scene.GetComponent<ResourcesComponent>();
//加载UI的AB包
resourcesComponent.LoadBundle(UIType.UILogin.StringToAB());
//获取到登录UI
GameObject bundleGameObject = (GameObject)resourcesComponent.GetAsset(UIType.UILogin.StringToAB(), UIType.UILogin);
//实例化登录UI
GameObject gameObject = UnityEngine.Object.Instantiate(bundleGameObject);
//在组件工厂创建UI（主要是为了能给游戏物体添加上组件做准备）
UI ui = ComponentFactory.Create<UI, string, GameObject>(UIType.UILogin, gameObject, false);
//为UI添加登录UI组件
ui.AddComponent<UILoginComponent>();
```

 **然后我们知道添加一个组件会执行其的Awake函数（如果存在并写明了继承自AwakeSystem的类），我们看到，他在Awake里面从其挂载的ReferenceCollector取得自身子物体的引用，并且为Button添加了点击事件**

```c#
 public void Awake()
{
	ReferenceCollector rc = this.GetParent<UI().GameObject.GetComponent<ReferenceCollector>();
	loginBtn = rc.Get<GameObject>("LoginBtn");
	loginBtn.GetComponent<Button>().onClick.Add(OnLogin);
	this.account = rc.Get<GameObject>("Account");
}

public void OnLogin()
{
	LoginHelper.OnLoginAsync(this.account.GetComponent<InputField>().text).Coroutine();
}
```

 **那么我们去看OnLoginAsync函数 ，知道我为什么在最后空两行吗？（观众：装起来了？？？）**

 **是因为这里用的await 和 Session.Call方法，`await是阻塞方法`，在返回结果之前不会继续往下走，而Call是要求有反馈消息的，综上只有在收到消息或者遇到异常才会有可能（异常的话程序将停止运行）执行到`R2C_Login reCLogin`这里，然后才会走最后一行**

```c#
 // 创建一个ETModel层的Session
ETModel.Session session = ETModel.Game.Scene.GetComponent<NetOuterComponent>().Create(GlobalConfigComponent.Instance.GlobalProto.Address);
// 创建一个ETHotfix层的Session, ETHotfix的Session会通过ETModel层的Session发送消息
Session realmSession = ComponentFactory.Create<Session, ETModel.Session>(session);
//发送登录请求，账号，密码均为传来的参数（理应这样，但这是例子，就无所谓了）
R2C_Login r2CLogin = (R2C_Login) await realmSession.Call(new C2R_Login() { Account = account, Password = "111111" });
                
                
//释放realmSession
realmSession.Dispose();
```

 **注意 第二句的Awake注册了消息回调函数，并且这里的Session是ETHotfix的，与ETModel下的Session有区别**

```c#
 public override void Awake(Session self, ETModel.Session session)
{
	self.session = session;
	SessionCallbackComponent sessionComponent = self.session.AddComponent<SessionCallbackComponent>();
	sessionComponent.MessageCallback = (s, opcode, memoryStream) => { self.Run(s, opcode, memoryStream); };
	sessionComponent.DisposeCallback = s => { self.Dispose(); };
}
```

 **好，既然消息发走了，我们就去看服务端了 **

 **我们直接从NetworkComponent看起吧（事实上客户端和服务端用的是同一个NetworkComponent，所以这里的NetworkComponent组件看完就不需要重复看客户端的了）**

```c#
 // 外网消息组件
Game.Scene.AddComponent<NetOuterComponent, string>(outerConfig.Address);
 |
 |
\|/
K component = ComponentFactory.CreateWithParent<K, P1>(this, p1, this.IsFromPool);
 |
 |
\|/
Game.EventSystem.Awake(component, a);
 |
 |
\|/
[ObjectSystem]
public class NetOuterComponentAwake1System : AwakeSystem<NetOuterComponent, string>
{
	public override void Awake(NetOuterComponent self, string address)
	{
	}
}
 |
 |
\|/
public void Awake(NetworkProtocol protocol, string address, int packetSize = Packet.PacketSizeLength2)
{
}
```

 **因为默认是TCP，所以绑定OnAccept（当收到消息，他会被调用）**

```c#
 public void OnAccept(AChannel channel)
 |
 |
\|/
//ETModel.Session.cs
public void Awake(AChannel aChannel)
 |
 |
\|/
注册收到消息回调函数
//ETModel.Session.cs
public void OnRead(MemoryStream memoryStream)
 |
 |
\|/
//TChannel.cs
public override void Start()
 |
 |
\|/
开始接收消息
//TChannel.cs
private void StartRecv()
 |
 |
\|/
接收消息完毕，解析好收到的信息，通过消息回调函数处理消息，
如果是热更层的消息，就调用ETHotfix.Session里注册的回调函数处理，
否则就用ETModel.Session里的函数处理，
（其实两者也是大同小异，大家可以自行查看各个Session的Run函数）
这里面又涉及到消息分发组件MessageDispatcherComponent以及requestCallback的细节，
大家可以自行查看
最后发送消息
 |
 |
\|/
//TChannel.cs
public void StartSend()

```

 **更多细节我会在之后介绍，我们这里的Handler是C2R_LoginHandler.cs**

```c#
 //这里是进行数据库验证，作为例子，这里去掉了验证
//if (message.Account != "abcdef" || message.Password != "111111")
//{
//	response.Error = ErrorCode.ERR_AccountOrPasswordError;
//	reply(response);
//	return;
//}

// 随机分配一个Gate
StartConfig config = Game.Scene.GetComponent<RealmGateAddressComponent>().GetAddress();
//Log.Debug($"gate address: {MongoHelper.ToJson(config)}");

//读取内部服务器地址
IPEndPoint innerAddress = config.GetComponent<InnerConfig>().IPEndPoint;

//从地址缓存中取Session,如果没有则创建一个新的Session,并且保存到地址缓存中
Session gateSession = Game.Scene.GetComponent<NetInnerComponent>().Get(innerAddress);

// 向gate服务器请求一个key,客户端可以拿着这个key连接gate
G2R_GetLoginKey g2RGetLoginKey = (G2R_GetLoginKey)await gateSession.Call(new R2G_GetLoginKey() {Account = message.Account});

//上述过程已完成，将执行这句，获取外部服务器地址
string outerAddress = config.GetComponent<OuterConfig>().Address2;

//设置返回的信息
response.Address = outerAddress;
response.Key = g2RGetLoginKey.Key;
//回复客户端的登录请求
reply(response);
```

 **好了，服务端的消息发过来了，我们客户端要响应吧，再回到客户端，这时候`r2CLogin`已经被赋好值了，可以进行下面的操作了。**

###  Actor消息的流转
   **继续上一次的一般消息流转，那么这次，我们来走一遍ET的另一核心——Actor消息机制**
   **继续上次，我们回到客户端**

```c#
 // 创建一个ETModel层的Session,并且保存到ETModel.SessionComponent中
ETModel.Session gateSession = ETModel.Game.Scene.GetComponent<NetOuterComponent>().Create(r2CLogin.Address);
//添加ETModel.SessionComponent组件，并对Session赋值
ETModel.Game.Scene.AddComponent<ETModel.SessionComponent>().Session = gateSession;
				
// 创建一个ETHotfix层的Session, 并且保存到ETHotfix.SessionComponent中，
// 目前为止，所有客户端的消息的收发都将被GateSession管理
//（Model层和Hotfix层都有各自的GateSession，
// 本质上还是ETModel.Session和ETHotfix.Session）
Game.Scene.AddComponent<SessionComponent>().Session = ComponentFactory.Create<Session, ETModel.Session>(gateSession);
				
//向服务端请求登录进gate服务器，如果登录成功，此后与服务端的通信都将通过gate服务器这个中介者
G2C_LoginGate g2CLoginGate = (G2C_LoginGate)await SessionComponent.Instance.Session.Call(new C2G_LoginGate() { Key = r2CLogin.Key });

Log.Info("登陆gate成功!");
```

 **然后回到服务端**

```c#
 //从已经分发的KEY里面寻找，如果没找到，说明非法用户，不给他连接gate服务器
string account = Game.Scene.GetComponent<GateSessionKeyComponent>().Get(message.Key);
if (account == null)
{
	response.Error = ErrorCode.ERR_ConnectGateKeyError;
	response.Message = "Gate key验证失败!";
	reply(response);
	return;
}
//专门给这个玩家创建一个Player对象
Player player = ComponentFactory.Create<Player, string>(account);
//注册到PlayerComponent，方便管理
Game.Scene.GetComponent<PlayerComponent>().Add(player);

//给这个session安排上Player
session.AddComponent<SessionPlayerComponent>().Player = player;
//添加邮箱组件表示该session是一个Actor,接收的消息将会队列处理
session.AddComponent<MailBoxComponent, string>(MailboxType.GateSession);

response.PlayerId = player.Id;
				
//回复客户端的连接gate服务器请求
reply(response);

//向客户端发送热更层信息
session.Send(new G2C_TestHotfixMessage() { Info = "recv hotfix message success" });
```

 **再回到客户端**

```c#
 Log.Info("登陆gate成功!");

// 创建Player
Player player = ETModel.ComponentFactory.CreateWithId<Player>(g2CLoginGate.PlayerId);
PlayerComponent playerComponent = ETModel.Game.Scene.GetComponent<PlayerComponent>();
playerComponent.MyPlayer = player;

//分发登录完成的事件
Game.EventSystem.Run(EventIdType.LoginFinish);

// 测试消息有成员是class类型
G2C_PlayerInfo g2CPlayerInfo = (G2C_PlayerInfo) await SessionComponent.Instance.Session.Call(new C2G_PlayerInfo());
Debug.Log("测试玩家信息为" + g2CPlayerInfo.Message);
```

 **这下我们不用再回到服务端了，因为一般消息的流转我们了解的已经差不多了，我们直接来到下一个重要的阶段**

 **来到MapHelper.cs类，查看登录Map服务器相关代码**

```c#
 // 获取资源组件
ResourcesComponent resourcesComponent = ETModel.Game.Scene.GetComponent<ResourcesComponent>();
// 加载Unit资源
await resourcesComponent.LoadBundleAsync($"unit.unity3d");

// 加载场景资源
await ETModel.Game.Scene.GetComponent<ResourcesComponent>().LoadBundleAsync("map.unity3d");
// 切换到map场景
using (SceneChangeComponent sceneChangeComponent = ETModel.Game.Scene.AddComponent<SceneChangeComponent>())
{
     await sceneChangeComponent.ChangeSceneAsync(SceneType.Map);
}
				
//请求登录Map服务器
G2C_EnterMap g2CEnterMap = await ETModel.SessionComponent.Instance.Session.Call(new C2G_EnterMap()) as G2C_EnterMap;
```

 **好了，我们又要去服务端了，注意，下面的消息是属于服务器内部的消息流通，所以不用网络层的传输，直接进行内部数据传输最后被响应Handler处理**

```c#
 //获取Player对象引用
Player player = session.GetComponent<SessionPlayerComponent>().Player;
// 在map服务器上创建战斗Unit
IPEndPoint mapAddress = StartConfigComponent.Instance.MapConfigs[0].GetComponent<InnerConfig>().IPEndPoint;
Session mapSession = Game.Scene.GetComponent<NetInnerComponent>().Get(mapAddress);
                
//由gate服务器向map服务器发送创建战斗单位请求，这里的session.InstanceId将由IdGenerater创建，
//用以保证不会冲突
M2G_CreateUnit createUnit =
(M2G_CreateUnit) await mapSession.Call(new G2M_CreateUnit() { PlayerId = player.Id, GateSessionId = session.InstanceId });
```

 **那我们直接到它的Handler这里吧**

```c#
 //创建战斗单位（小骷髅给劲哦）
Unit unit = ComponentFactory.CreateWithId<Unit>(IdGenerater.GenerateId());
//增加移动组件
unit.AddComponent<MoveComponent>();
//增加寻路相关组件
unit.AddComponent<UnitPathComponent>();
//设置小骷髅位置
unit.Position = new Vector3(-10, 0, -10);
				
//给小骷髅添加信箱组件，队列处理收到的消息
await unit.AddComponent<MailBoxComponent>().AddLocation();
//添加同gate服务器通信基础组件，主要是赋予ID
unit.AddComponent<UnitGateComponent, long>(message.GateSessionId);
//将这个小骷髅维护在Unit组件里
Game.Scene.GetComponent<UnitComponent>().Add(unit);
//设置回复消息的ID
response.UnitId = unit.Id;
				
// 广播创建的unit
M2C_CreateUnits createUnits = new M2C_CreateUnits();
Unit[] units = Game.Scene.GetComponent<UnitComponent>().GetAll();
foreach (Unit u in units)
{
	UnitInfo unitInfo = new UnitInfo();
	unitInfo.X = u.Position.x;
	unitInfo.Y = u.Position.y;
	unitInfo.Z = u.Position.z;
	unitInfo.UnitId = u.Id;
	createUnits.Units.Add(unitInfo);
}
//广播所有小骷髅信息
MessageHelper.Broadcast(createUnits); 
				
//广播完回复客户端，这边搞好了
reply(response);
```

 **注意，这里面的MessageHelper.Broadcast(createUnits); 就是我们的核心Actor机制的体现了，我们只要把消息发到gatesession，gatesession将会自动根据id转发消息到相应客户端，而里面涉及到的相关ID，我们在由gate服务器向map服务器发送创建战斗单位请求的时候，已经传好参数了**

```c#
 // 从Game.Scene上获取ActorSenderComponent，然后通过InstanceId获取ActorMessageSender
ActorSenderComponent actorSenderComponent = Game.Scene.GetComponent<ActorSenderComponent>();
ActorMessageSender actorMessageSender = actorSenderComponent.Get(unitGateComponent.GateSessionActorId);
// send
actorMessageSender.Send(message);

// rpc
var response = actorMessageSender.Call(message);
```

 **好了，回到客户端**

```c#
 //设置UnitID
PlayerComponent.Instance.MyPlayer.UnitId = g2CEnterMap.UnitId;
				
//增加。。。emmm不知道怎么翻译这个组件好，他负责点击地面控制小骷髅移动
Game.Scene.AddComponent<OperaComponent>();
				
//分发进入正式游戏成功事件
Game.EventSystem.Run(EventIdType.EnterMapFinish);
```

 **然后我们突然想起来，刚刚亡灵领主出生的时候向别的小骷髅广播了一下来着，我们看会做什么操作**

```c#
 foreach (UnitInfo unitInfo in message.Units)
{
	if (unitComponent.Get(unitInfo.UnitId) != null)
	{
		continue;
	}
	//根据不同ID，创建小骷髅
	Unit unit = UnitFactory.Create(unitInfo.UnitId);
	unit.Position = new Vector3(unitInfo.X, unitInfo.Y, unitInfo.Z);
}
```

 **好了 ，小骷髅都安排好了，该向服务端发消息了，也就是寻路，还记得我们之前添加的****OperaComponent，他就是负责寻路的**

```c#
 //发送点击地图消息
ETModel.SessionComponent.Instance.Session.Send(frameClickMap);
 |
 |
 |
\|/
//服务端Frame_ClickMapHandler处理位置信息
Vector3 target = new Vector3(message.X, message.Y, message.Z);
unit.GetComponent<UnitPathComponent>().MoveTo(target).Coroutine();
 |
 |
 |
\|/
//移动到指定位置
await self.MoveAsync(self.ABPath.Result);
 |
 |
 |
\|/
// 每移动3个点发送下3个点给客户端
if (i % 3 == 1)
{
     self.BroadcastPath(path, i, 3);
}
 |
 |
 |
\|/
// 客户端处理服务端的同步信息，利用寻路组件进行寻路
unitPathComponent.StartMove(message).Coroutine();
```

### 后记

 **至此，连接服务器，创建小骷髅，并且自动寻路的操作就完成了，我中间很多细节没有讲，其实没必要讲（观众：懒还有理了？？？），那些东西需要大家自行理解和体会的，而到现在master的Demo解读也该告一段落了。**

## 消息总结

### 不同Session（服务器）相关名称及其含义

- **Manager：连接客户端的外网和连接内部服务器的内网，对服务器进程进行管理，自动检测和启动服务器进程。加载有内网组件NetInnerComponent，外网组件NetOuterComponent，服务器进程管理组件。自动启动突然停止运行的服务器，保证此服务器管理的其它服务器崩溃后能及时自动启动运行。** 
- **Realm：对ActorMessage消息进行管理（添加、移除、分发等），连接内网和外网，对内网服务器进程进行操作，随机分配Gate服务器地址。加载有ActorMessage消息分发组件ActorMessageDispatherComponent，ActorManager消息管理组件ActorManagerComponent，内网组件NetInnerComponent，外网组件NetOuterComponent，服务器进程管理组件LocationProxyComponent，Gate服务器随机分发组件。客户端登录时连接的第一个服务器，也可称为登录服务器。** 
- **Gate：对玩家进行管理，对ActorMessage消息进行管理（添加、移除、分发等），连接内网和外网，对内网服务器进程进行操作，随机分配Gate服务器地址，对Actor消息进程进行管理，对玩家ID登录后的Key进行管理。加载有玩家管理组件PlayerComponent，ActorMessage消息分发组件ActorMessageDispatherComponent，ActorManager消息管理组件ActorManagerComponent，内网组件NetInnerComponent，外网组件NetOuterComponent，服务器进程管理组件LocationProxyComponent，Actor消息管理组件ActorProxyComponent，管理登陆时联网的Key组件GateSessionKeyComponent。对客户端的登录信息进行验证和客户端登录后连接的服务器，登录后通过此服务器进行消息互动，也可称为验证服务器。** 
- **Location：连接内网，服务器进程状态集中管理（Actor消息IP管理服务器）。加载有内网组件NetInnerComponent，服务器消息处理状态存储组件LocationComponent，也可称为进程管理组件。** 
- **Map：连接内网，对ActorMessage消息进行管理（添加、移除、分发等），对场景内现在活动物体存储管理，对内网服务器进程进行操作，对Actor消息进程进行管理，对ActorMessage消息进行管理（添加、移除、分发等），服务器帧率管理。ActorMessage消息分发组件ActorMessageDispatherComponent，ActorManager消息管理组件ActorManagerComponent，内网组件NetInnerComponent，服务器进程管理组件LocationProxyComponent，服务器帧率管理组件ServerFrameComponent，可以称为内网组件。** 
- **AllServer：将以上服务器功能集中合并成一个服务器。另外增加DB连接组件DBComponent，DB管理组件DBProxyComponent。** 
- **Benchmark：连接内网和测试服务器承受力。加载有内网组件NetInnerComponent，服务器承受力测试组件BenchmarkComponent。** 

### 不同消息及其对应特性

- **不需要返回结果的消息 IMessage** 
- **需要返回结果的消息 IRequest** 
- **用于回复的消息 IResponse** 
- **不需要返回结果的Actor消息 IActorMessage，IActorLocationMessage** 
- **需要返回结果的Actor消息 IRequest IActorLocationRequest** 
- **用于回复的Actor消息 IResponse IActorLocationResponse**  
  **参考：[http://www.tinkingli.com/?p=76**
