---
title: ET篇：云端分布式服务器部署教程
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
cover: https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/09/QQ截图20190917124558.png
aplayer:
---
<meta name="referrer" content="no-referrer" />

### 前言
终于还是躲不了，要部署云服务器，但是这方面真的一点经验和知识都没有，所以只能求助别人，顺便共同讨论，一起进步。
感谢`快二十了还一事无成`，`喵大王`的大力支持。
在阅读本博客前，请务必先阅读哲学大佬的单服务器部署教程：[https://gameinstitute.qq.com/community/detail/119615](https://gameinstitute.qq.com/community/detail/119615) 以及Tinkingli的博客：[http://www.tinkingli.com/?p=25](http://www.tinkingli.com/?p=25)
### 正文
我决定还是以代码作为切入点，来理解服务器的配置。
首先我们来到客户端和服务端第一次通信的地方。
```csharp
客户端的LoginHelper.cs
//根据全局配置的服务端地址进行第一次通信
ETModel.Session session = ETModel.Game.Scene.GetComponent<NetOuterComponent>().Create(GlobalConfigComponent.Instance.GlobalProto.Address);
Session realmSession = ComponentFactory.Create<Session, ETModel.Session>(session);
R2C_Login r2CLogin = (R2C_Login) await realmSession.Call(new C2R_Login() { Account = account, Password = "111111" });
...
服务端的C2R_LoginHandler.cs
StartConfig config = Game.Scene.GetComponent<RealmGateAddressComponent>().GetAddress();
//这里的gate取的是内网地址，用于和realm进行通讯
IPEndPoint innerAddress = config.GetComponent<InnerConfig>().IPEndPoint;
Session gateSession = Game.Scene.GetComponent<NetInnerComponent>().Get(innerAddress);
G2R_GetLoginKey g2RGetLoginKey = (G2R_GetLoginKey)await gateSession.Call(new R2G_GetLoginKey() {Account = request.Account});
//注意这个取得是外网地址2
string outerAddress = config.GetComponent<OuterConfig>().Address2;
response.Address = outerAddress;
...

```
OK，上面的代码流程涉及到了内网和外网地址2，那么外网地址1在哪里用了呢？
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/09/QQ截图20190917123148.png)
是启动程序那里的，`意为监听客户端传来的信息`。所以一般外网地址1设置为0.0.0.0（监听本主机所有IP连接）（注意要和外网地址2的端口一致）。
现在外网地址都解决了，内网地址该填什么呢？
如果你的多台服务器已经组成了一个局域网（服务器群组），可以填每个服务器的私有地址。如果只是单纯的多个服务器，还是需要填公网IP，并且端口不得与外网地址一致。
总结一下，就是这样
我们约定，服务器公网IP为`192.168.0.1`，内网IP为`44.117.7.8`.如果连接不上，就把外网地址1换成0.0.0.0
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/09/QQ截图20190917124558.png)
最后是全局配置这里
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/09/QQ截图20190917124716.png)