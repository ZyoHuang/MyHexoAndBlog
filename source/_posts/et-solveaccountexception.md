---
title: ET篇：账号异常解决方案汇总
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

## 前言
**当然了，网络游戏中的异常太多了，断线重连，封号处理之类的，但是目前我还没有接触到那些模块，就没必要做超前的处理，等做到了也会整合到这篇博客里，本篇博客主要是讲解思路，可能代码上面有些描述不够清晰，大家可以去我的项目里翻一下完整代码**
**[https://gitee.com/NKG_admin/MKGMobaBasedOnET](https://gitee.com/NKG_admin/MKGMobaBasedOnET "https://gitee.com/NKG_admin/MKGMobaBasedOnET")**

### 常见异常为
1. **账号被人顶下来**
2. **客户端自身或者网络问题(死机,网络不良)无法连接服务器或者与服务器断开连接**
3. **客户端突发状况（断电,退出游戏）,与服务器断开连接**
### 在开始说明解决方案之前，我们先明确几个概念
1. **如果客户端这边退出游戏，将会调用Session.Dispose()，并且服务端与之对应的Session也会执行Dispose**
2. **网络不良或者没有网络，将不会调用Session.Dispose()，要靠双端的心跳包情况来让服务端判断是否要断开连接（执行Session.Dispose()）,然后双端各自执行断线后的逻辑**
3. **`emmm，为什么这两个简单的概念把我绕了一天？（观众：是不是脑瘫就不用我们多说了⑧）`**
## 对应的三个解决方案
### 账号被人顶下来的解决方案

**既然是被人顶下来，说明服务端知晓整个过程，所以就用服务端通知客户端的方式来实现整个逻辑**

#### 客户端
**首先是断线组件，他被添加在一个Session上面（一般是gateSession），当Session.Dispose时，它的Dispose也会被执行**
```csharp
namespace ETHotfix
{
    /// <summary>
    /// 用于Session断开时触发下线
    /// </summary>
    public class SessionOfflineComponent: Component
    {
        public override void Dispose()
        {
            if (this.IsDisposed)
            {
                return;
            }
            base.Dispose();
            Game.Scene.RemoveComponent<SessionComponent>();
            ETModel.Game.Scene.RemoveComponent<ETModel.SessionComponent>();
        }
    }
}
```
#### 定义热更层离线协议
```csharp
message G2C_PlayerOffline // IMessage
{
	int32 m_playerOfflineType = 1;
}
```
#### 对服务端发来的离线消息进行处理
```csharp
namespace ETHotfix
{
    [MessageHandler]
    public class G2C_PlayerOfflineHandler: AMHandler<G2C_PlayerOffline>
    {
        protected override void Run(ETModel.Session session, G2C_PlayerOffline message)
        {
            Log.Info("收到了服务端的下线指令");
            switch (message.MPlayerOfflineType)
            {
				// 可拓展为根据消息的离线类型执行不同的操作
                case 1:
                    Log.Info("由于账号被顶而离线");
                    // 执行相关操作
                    break;
            }
        }
    }
}
```
#### 服务端
**服务端要做的事情就比较多了，因为要涉及各个服务器消息的流转**
**先在InnerMessage定义几个需要在内部流转的协议**
```
message G2R_PlayerOnline // IRequest
{
    int32 RpcId = 90;
    long PlayerId = 1;
    string playerAccount = 3;
    int GateAppID = 2;
}

message R2G_PlayerOnline // IResponse
{
    int32 RpcId = 90;
    int32 Error = 91;
    string Message = 92;
}

message G2R_PlayerOffline // IRequest
{
    int32 RpcId = 90;
    long PlayerId = 1;
    string playerAccount = 3;
}

message R2G_PlayerOffline // IResponse
{
    int32 RpcId = 90;
    int32 Error = 91;
    string Message = 92;
}

message R2G_PlayerKickOut // IRequest
{
    int32 RpcId = 90;
    long PlayerId = 1;
    PlayerOfflineTypes Playerofflinetypes = 2;
    string PlayerAccount = 3;
}

message G2R_PlayerKickOut // IResponse
{
    int32 RpcId = 90;
    int32 Error = 91;
    string Message = 92;
}
```
**另外，由于InnerMessage的特殊性，需要单独写一个枚举类型**
```csharp
namespace ETModel
{
    /// <summary>
    /// 非断网情况下的两种玩家离线情况
    /// </summary>
    public enum PlayerOfflineTypes
    {
        NoPlayForLongTime = 1,
        SamePlayerLogin = 2
    }
}
```
**新增OnlineComponent组件，缓存并维护玩家，这个组件应当添加到Game.Scene上**
```csharp
using System;
using System.Collections.Generic;

namespace ETModel
{
    /// <summary>
    /// 在线组件，用于记录在线玩家
    /// </summary>
    public class OnlineComponent: Component
    {
        private readonly Dictionary<string, Tuple<long, int>> m_dictionarty = new Dictionary<string, Tuple<long, int>>();

        /// <summary>
        /// 添加在线玩家
        /// </summary>
        /// <param name="playerAccount"></param>
        /// <param name="gateAppId"></param>
        public void Add(string playerAccount, long playerId, int gateAppId)
        {
            this.m_dictionarty.Add(playerAccount, new Tuple<long, int>(playerId, gateAppId));
        }

        /// <summary>
        /// 获取在线玩家ID
        /// </summary>
        /// <param name="playerId"></param>
        /// <returns></returns>
        public long GetPlayerId(string playerAccount)
        {
            Tuple<long, int> temp = new Tuple<long, int>(0, 0);
            this.m_dictionarty.TryGetValue(playerAccount, out temp);
            return temp.Item1;
        }

        /// <summary>
        /// 获取在线玩家网关服务器ID
        /// </summary>
        /// <param name="playerId"></param>
        /// <returns></returns>
        public int GetGateAppId(string playerAccount)
        {
            if (this.m_dictionarty.Count >= 1)
            {
                Tuple<long, int> temp = new Tuple<long, int>(0, 0);
                this.m_dictionarty.TryGetValue(playerAccount, out temp);
                return temp.Item2;
            }

            return 0;
        }

        /// <summary>
        /// 移除在线玩家
        /// </summary>
        /// <param name="playerId"></param>
        public void Remove(string playerAccount)
        {
            Tuple<long, int> temp;
            if (!this.m_dictionarty.TryGetValue(playerAccount, out temp)) return;
            this.m_dictionarty.Remove(playerAccount);
        }

    }
}
```
**修改SessionPlayerComponent，重写Dispose，在最后直接去OnlineComponent移除需要移除的玩家**
```csharp
namespace ETModel
{
    public class SessionPlayerComponent: Component
    {
        public Player Player;

        public override void Dispose()
        {
            base.Dispose();
            Game.Scene.GetComponent<OnlineComponent>().Remove(this.Player.Account);
        }
    }
}
```
**定义RealmHelper类，辅助我们对玩家执行下线操作**
```csharp
using System;
using System.Net;
using ETModel;

namespace ETHotfix
{
    public static class RealmHelper
    {
        /// <summary>
        /// 将玩家踢下线
        /// </summary>
        /// <param name="playerId"></param>
        /// <returns></returns>
        public static async ETTask KickOutPlayer(string playerAccount, PlayerOfflineTypes playerOfflineType)
        {
            //验证账号是否在线，在线则踢下线
            int gateAppId = Game.Scene.GetComponent<OnlineComponent>().GetGateAppId(playerAccount);
            if (gateAppId != 0)
            {
                // 获取内网gate，向realm发送离线信息
                StartConfig playerGateConfig = Game.Scene.GetComponent<StartConfigComponent>().Get(gateAppId);
                IPEndPoint playerGateIPEndPoint = playerGateConfig.GetComponent<InnerConfig>().IPEndPoint;
                Session playerGateSession = Game.Scene.GetComponent<NetInnerComponent>().Get(playerGateIPEndPoint);
                
                // 发送断线信息
                long playerId = Game.Scene.GetComponent<OnlineComponent>().GetPlayerId(playerAccount);
                Player player = Game.Scene.GetComponent<PlayerComponent>().Get(playerId);
                long playerSessionId = player.GetComponent<UnitGateComponent>().GateSessionActorId;
                Session lastGateSession = Game.Scene.GetComponent<NetOuterComponent>().Get(playerSessionId);
                
                switch (playerOfflineType)
                {
                    case PlayerOfflineTypes.NoPlayForLongTime:
                        // 因长时间未操作而强制下线
                        lastGateSession.Send(new G2C_PlayerOffline() { MPlayerOfflineType = 1 });
                        break;
                    case PlayerOfflineTypes.SamePlayerLogin:
                        // 因账号冲突而强制下线
                        lastGateSession.Send(new G2C_PlayerOffline() { MPlayerOfflineType = 2 });
                        break;
                }

                //服务端主动断开客户端连接
                await playerGateSession.Call(new R2G_PlayerKickOut() { PlayerAccount = playerAccount, PlayerId = playerId });

                Console.WriteLine($"玩家{playerId}已被踢下线");
            }
        }
    }
}
```
**接下来是具体的流转过程**
```csharp
==>//向realm服务器发送玩家上线消息，accout为玩家账号
C2G_LoginGateHandler==>await realmSession.Call(new G2R_PlayerOnline(){ playerAccount = account, PlayerId = player.Id, GateAppID = config.StartConfig.AppId });
==>G2R_PlayerOnlineHandler
==>//将已在线玩家踢下线
await RealmHelper.KickOutPlayer(message.playerAccount, PlayerOfflineTypes.SamePlayerLogin);
==>//向客户端发送断线消息
lastGateSession.Send(new G2C_PlayerOffline() { MPlayerOfflineType = 1 });
==>//服务端主动断开客户端连接
await playerGateSession.Call(new R2G_PlayerKickOut() { PlayerAccount = playerAccount, PlayerId = playerId });
==>//玩家上线，添加到onlineComponent字典里
onlineComponent.Add(message.playerAccount, message.PlayerId, message.GateAppID);
```
**至此账号互顶的双端逻辑结束**
### 客户端或服务端出故障的解决方案
#### 客户端
**同样的，先定义几个OutMessage协议**
```
message C2G_HeartBeat // IRequest
{
	int32 RpcId = 90;
}

message G2C_HeartBeat // IResponse
{
	int32 RpcId = 90;
	int32 Error = 91;
	string Message = 92;
}
```
**心跳组件，同样的，他也会被添加到一个Session上（一般为gateSession）**
```csharp
namespace ETModel
{
    [ObjectSystem]
    public class HeartBeatSystem: UpdateSystem<HeartBeatComponent>
    {
        public override void Update(HeartBeatComponent self)
        {
            self.Update();
        }
    }

    /// <summary>
    /// Session心跳组件(需要挂载到Session上)
    /// </summary>
    public class HeartBeatComponent: Component
    {
        /// <summary>
        /// 心跳包间隔
        /// </summary>
        public float SendInterval = 10f;

        /// <summary>
        /// 记录时间
        /// </summary>
        private float RecordDeltaTime = 0f;

        /// <summary>
        /// 判断是否已经离线
        /// </summary>
        private bool hasOffline;

        public async void Update()
        {
            if (this.hasOffline) return;

            // 如果还没有建立Session直接返回、或者没有到达发包时间
            if (Time.time - this.RecordDeltaTime < this.SendInterval) return;
            // 记录当前时间
            this.RecordDeltaTime = Time.time;

            // 开始发包
            try
            {
                G2C_HeartBeat result = (G2C_HeartBeat) await this.GetParent<Session>().Call(new C2G_HeartBeat());
            }
            catch
            {
                if (this.hasOffline) return;
                this.hasOffline = true;
                Log.Info("发送心跳包失败");
                // 执行相关操作
            }
        }
    }
}
```
#### 服务端
**心跳消息处理函数**
```csharp
namespace ETHotfix
{
    [MessageHandler(AppType.Gate)]
    public class C2G_HeartBeatHandler: AMRpcHandler<C2G_HeartBeat, G2C_HeartBeat>
    {
        protected override void Run(Session session, C2G_HeartBeat message, Action<G2C_HeartBeat> reply)
        {
            if (session.GetComponent<HeartBeatComponent>() != null)
            {
                session.GetComponent<HeartBeatComponent>().CurrentTime = TimeHelper.ClientNowSeconds();
            }
            reply(new G2C_HeartBeat());
        }
    }
}
```
**定义心跳组件，这个一般也要放gateSession上**
```csharp
using System;
using System.Net;

namespace ETModel
{
    [ObjectSystem]
    public class HeartBeatSystem: UpdateSystem<HeartBeatComponent>
    {
        public override void Update(HeartBeatComponent self)
        {
            self.Update();
        }
    }

    /// <summary>
    /// Session心跳组件(需要挂载到Session上)
    /// </summary>
    public class HeartBeatComponent: Component
    {
        /// <summary>
        /// 更新间隔
        /// </summary>
        public long UpdateInterval = 5;

        /// <summary>
        /// 超出时间
        /// </summary>
        /// <remarks>如果跟客户端连接时间间隔大于在服务器上删除该Session</remarks>
        public long OutInterval = 10;

        /// <summary>
        /// 记录时间
        /// </summary>
        private long _recordDeltaTime = 0;

        /// <summary>
        /// 当前Session连接时间
        /// </summary>
        public long CurrentTime = 0;


        public void Update()
        {
            // 如果没有到达发包时间、直接返回
            if ((TimeHelper.ClientNowSeconds() - this._recordDeltaTime) < this.UpdateInterval || this.CurrentTime == 0) return;
            // 记录当前时间
            this._recordDeltaTime = TimeHelper.ClientNowSeconds();

            if (TimeHelper.ClientNowSeconds() - CurrentTime > OutInterval)
            {
                Console.WriteLine("心跳失败");
                Game.Scene.GetComponent<NetOuterComponent>().Remove(this.Parent.InstanceId);
                Game.Scene.GetComponent<NetInnerComponent>().Remove(this.Parent.InstanceId);
            }
            else
            {
                Console.WriteLine("心跳成功");
            }
        }
    }
}
```
## 脑瘫解决方案
**`多玩游戏，多看大佬们吹牛逼。`**
