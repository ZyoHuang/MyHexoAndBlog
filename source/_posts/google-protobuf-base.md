---
title: Google.Protobuf篇：官方案例AddressBook讲解
tags: [网络编程, ProtoBuffer]
categories:
  - - GamePlay
    - 网络通信
date: 2019-04-29 16:50:18
---

<meta name="referrer" content="no-referrer" />



据我目前所知,ET使用的数据交换协议是Google.Protobuf,所以今天就来学习一下,但是网上许多人都是以高下立判的方式讲解的,对新手小白很不友好,所以我今天就以纯小白的视角和大家一起学习

### 学习环境:

.NET Core2.2 .NET Framework 4.7.2 Rider protobuf-csharp-3.7.0-rc-2

### Google.Protobuf简介

protocolbuffer(以下简称PB)是google 的一种数据交换的格式，它独立于语言，独立于平台。 google 提供了多种语言的实现：java、c#、c++、go 和 python，每一种实现都包含了相应语言的编译器以及库文件。 由于它是一种二进制的格式，比使用xml 进行数据交换快许多。可以把它用于分布式应用之间的数据通信或者异构环境下的数据交换。 作为一种效率和兼容性都很优秀的二进制数据传输格式，可以用于诸如网络传输、配置文件、数据存储等诸多领域。

### 下载Google.Protobuf

https://github.com/protocolbuffers/protobuf/releases ![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190208194411464.png) 下载完成解压,使用Rider打开 ![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190208194525582.png) 打开加载完成之后,面板就如图,而箭头所指AddressBook就是我们今天的重头戏了 ![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190208194632369.png) 先是程序入口,Program类这个类的大体功能已在注释写明 ![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190208194935274.png) 细分一下就是,从指定的数据文件读取数据,并按照指令使用不同函数来对数据文件进行不同的操作 L为展示文件的具体内容 A为增加新的数据 Q为退出程序 那我们就再跟进AddPerson类(按住Ctrl+鼠标左键即可) 来看它的Main函数 ![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190208195816564.png) 再来看ListPerson类的Main函数 ![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190208195951638.png) 至此,程序的主体已经运行结束了,但我们发现,除了我们提到的三个类,还有两个类 ![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190208200148187.png) 我们先看Addressbook类 最顶上写了这么一句,意为是protocl工具生成,不要编辑修改他,所以我们就不管他了,(看也看不懂),但我还是看了一下 ![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190208200240450.png) 它里面包含了两个类,分别是AddressBook和Person,而这两个类里面又包含各自的属性和字段,不过这不重要,因为他明显不是给人看的,知道怎么回事就行了 ![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/2019020820211299.png) 再来看SampleUsage类发现他只是一个示例类 ![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190208200820870.png) 好了,基本的讲的差不多了,接下来我们运行一下 运行是没啥问题了,但当我们想进行操作的时候,就出现问题了 ![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190208200946492.png) 回到Program类 ![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190208201039944.png) 所以我们要新建一个数据文件 ![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190208201112517.png) ![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190208201136509.png) 再次运行就发现没啥问题了,功能也都可以使用了