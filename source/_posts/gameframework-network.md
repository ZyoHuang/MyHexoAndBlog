---
title: GameFramework篇：Network模块使用示例
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

# Network模块


## 连接ET服务端
### 前言

 **过了那么久，我又回来了，因为我开始研习服务端了（欠的技术债总要还的），因为GF已经越来越熟练，并且使用过程中也十分稳定，所以我已经决定了，使用GF做客户端，至于服务端，因为对于中大型项目，服务端框架也必不可少，我选择了ET（另一个很强大的框架，是双端的链接：[https://github.com/egametang/ET](https://github.com/egametang/ET)）。**

### 项目工程下载

 **既然要学习，那就要一个案例，身为服务端小白，我是写不出什么案例的，E大的StarForce中的Network功能已经做了示例，但是木头大佬已经讲解过了，我就没必要再说一遍了，链接[http://www.benmutou.com/archives/2630](http://www.benmutou.com/archives/2630)，那么用啥案例呢？这就要请出我们的毛毛大佬了，奉上GayHub [https://github.com/Maomao110/StarForce](https://github.com/Maomao110/StarForce)**

 **下载之后，需要下载submoudle，不然会报错，不会的同学可以参考这篇的下载方式 [https://blog.csdn.net/qq_15020543/article/details/86774781](https://blog.csdn.net/qq_15020543/article/details/86774781)**

 **至于ET，我专门联系了毛毛大佬，毛毛大佬也是很豪爽，直接把工程发来了**

 **链接：https://pan.baidu.com/s/1fgUsJdlA6G0mizW_xd8lbw   
 提取码：diwc   
 复制这段内容后打开百度网盘手机App，操作更方便哦**

### **环境要求**

 **.Net Core 2.2**

 **.NetFramework 4.7.2**

 **Rider 2018.4**

 **Unity 2018.3**

 **OK，做好这些准备工作，我们来运行一下吧，毕竟一个Demo跑起来才能让人有学习的欲望**

 **首先运行ET的服务端**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190308220237865.png)

 **再运行StarForce的ETNetSample场景，发送消息，即可看到反馈**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190308220429956.png)


  **在正式开始讲之前，我们需要明确GF的Network模块的运行机制，感觉说起来太抽象了，上UML吧**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190307215454853.png)

 **应该。。。不难理解吧**

 **在这个案例中，为了迎合ET的服务端，协议为，前2个字节代表消息的长度， 第3个字节代表消息flag，1表示rpc消息，0表示普通消息 第4和第5个字节表示消息的Id**

 项目结构图（我做了修改。。。方便讲解）

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190307212349169.png)

 **来到程序入口ProcedureNetSample**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190307214258764.png)

 **说一下HotfixMessage.proto的作用，因为所有的消息都是需要被响应的，比如你客户端发送登录请求（C2S_Login），服务端要对这个消息(S2C_Login)进行回应吧，如果这个时候，这两个都需要改动一下，直接编辑proto文件，直接复制粘贴到服务端，生成代码，即可。**

 **至于ET那边，已经不属于GF的范畴了，主要就是收到客户端发送的消息，进行解析，然后给客户端回消息**

 **更多细节大家自己追踪以下代码即可**
