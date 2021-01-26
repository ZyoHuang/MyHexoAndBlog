---
title: AssetBundle热更新完整工作流与知识点解析
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

# 前言
虽然这一块内容是比较基础的，但是知识点比较分散，所以我还是决定写一篇博客来记录梳理一下。
环境：Unity 2018.4.0
# 参考文献
Unity官方文档：[https://docs.unity3d.com/Manual/index.html](https://docs.unity3d.com/Manual/index.html)
不会C++的码农知乎文章：[https://zhuanlan.zhihu.com/p/25111851](https://zhuanlan.zhihu.com/p/25111851)


# AssetBundle相关知识
## 什么是AssetBundle？
一个AssetBundle可以当做一个文件集合，它包含了Unity可以在运行时加载的特定于平台的非代码资产(例如模型、纹理、预制组件、音频，甚至整个场景)。AssetBundle可以表示彼此之间的依赖关系;例如，一个AssetBundle中的material可以引用另一个AssetBundle中的texture。为了在减轻网络传输压力，您可以根据需求选择内置算法(LZMA和LZ4)来压缩AssetBundle。
`AssetBundle对于可下载内容(DLC)、减少初始安装大小、加载针对最终用户平台优化的资产以及减少运行时内存压力都很有用。`
## AssetBundle里有什么？
“AssetBundle”可以指两个不同但相关的东西。
首先是磁盘上的实际文件。这称为AssetBundle archive。AssetBundle archive是一个容器，就像一个文件夹一样，其中包含了额外的文件。这些额外的文件包括两类:
一个序列化文件，它包含您的资产分解成它们各自的对象并写入到这个文件中。
另一个是资源文件，这是二进制数据块单独存储的某些资产(纹理和音频)，以允许Unity高效地在另一个线程从磁盘上加载他们。
“AssetBundle”也可以指通过代码与实际的AssetBundle对象交互，从特定的AssetBundle加载资产。这个对象包含了所有我们当初添加到包里面的内容。
## AssetBundle依赖与冗余介绍

Unity 5.x版本里提供了更加人性化的依赖自动管理机制——对指定打包的资源，Unity会自动收集并分析其依赖的资源，如果该资源依赖的某个资源没有被显式指定打包到ab中，就将其依赖的这个资源打包进该资源所在的ab里；如果已经被指定打包进其他ab里，那么这两个ab之间就会构成依赖关系，加载ab时，先加载其依赖的ab即可。
`请避免ab循环依赖，比如a依赖b，b也依赖a，那么加载a的时候会去先加载a在b中的依赖资源，那么就得去加载b，加载b前又得去加载a，造成死循环。`
这一套依赖管理机制使用方便的同时也会带来一个问题，如果两个ab A和B中的一些资源都依赖了一个没有被指定要打包的资源C，那么C就会同时被打进ab A和B中，造成资源的冗余，增大ab和安装包的体积。
至于处理这些问题的方法，大家可以自己去寻找。

## AssetBundle工作流

先来一张图
[![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/11/QQ截图20191126220128.png)](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/11/QQ截图20191126220128.png)

### 打包AB
使用Unity打出AB包：[https://docs.unity3d.com/Manual/AssetBundles-Workflow.html](https://docs.unity3d.com/Manual/AssetBundles-Workflow.html)
压缩算法目前LZ4压缩方式天下第一（BuildAssetBundleOptions.ChunkBasedCompression）

### 加载AB
#### Unity中资源相关目录介绍

- Resources：全部资源都会被压缩，转化成二进制。打包后该路径不存在，不可写也不可读。只能使用Resources.Load加载资源。
- StreamingAssets：全部资源原封不动打包。在移动平台下，是只读的，不能写入数据，其他平台下可以使用System.File类进行读写。在`任意平台`都可以使用AssetBundle.LoadFromFile来从此文件夹读取加载ab包。

#### 资源相关路径介绍

- Application.streamingAssetsPath：对应了StreamingAssets目录。安卓，PC平台WWW,UWR,AssetBundle.LoadFromFile对于streamingAssets里的文件访问直接用Application.streamingAssetsPath即可，苹果就用`"file://{Application.streamingAssetsPath}"`
- Application.persistentDataPath：可读写目录，任意平台可以使用System.File库进行读写操作，WWW,UWR,AssetBundle.LoadFromFile更不在话下。移动平台可以使用"`{Application.persistentDataPath}/{Application.productName}/"`进行访问，非移动平台直接使用Application.persistentDataPath即可访问。

#### 加载AB的API
- AssetBundle.LoadFromFile(path)：同步加载，path为本地路径
- AssetBundle.LoadFromFileAsync(path)：异步加载，path为本地路径
- AssetBundle.LoadFromMemory(byte[] binary)：从字节数组加载，binary为目标ab二进制流
- AssetBundle.LoadFromMemoryAsync(byte[] binary)：从字节数组异步加载，binary为目标ab二进制流
- UnityWebRequest.GetAssetBundle(string uri)：url为ab文件路径，可为本地，也可为云端，

### 从AB中加载具体的资产（Asset）API
- `assetBundle.LoadAsset<T>(name)：`T为目标资产类型，name为资产名称，会返回一个T实例
- assetBundle.LoadAsset(name,type)：name为资产名，type为资产类型
- `assetBundle.LoadAllAssets<T>()`：T为目标资产类型，会返回一个assetBundle中所有T类型资产数组
- assetBundle.LoadAllAssets()：加载assetBundle中所有资产，返回一个assetBundle中所有资产数组

### 卸载AB的API

- assetbundle.Unload(bool unloadAllLoadedObjects)：unloadAlLoadedlObjects：是否卸载所有加载的资源，参数为false时，AssetBundle内的文件内存镜像会被释放，实例化的物体还都保持完好。简单的说就是断开了AssetBundle内存镜像和实例之间的联系。如果再次实例化对象，也不会返回以前初例化过的AssetBundle内存镜像，而是重新实例化一个新的AssetBundle内存镜像，那么这样就出现了冗余，同样的资源，内存中会出现多份。参数为true时，就简单多了，卸载AssetBundle，并且删除被引用的资源。注意！如果AssetBundle中有资源在场景中被引用，则会出现资源丢失的情况。这种卸载方式，最为彻底，完全从内存移除，缺点是你需要一套机制（目前流行的是引用计数），来关注是不是还有资源引用，会不会引起异常。
- AssetBundle.UnloadAllAssetBundles(bool unloadAllObjects)：unloadAllObjects：是否卸载所有加载的资源，如果为true，则会卸载所有资源，包括正在使用着（被依赖）的资源。，如果为false，则会卸载未被依赖的资源，被其他资源依赖的资源不会被卸载。

总结：AssetBundle.Unload(false)适用于一次性使用的资源，获得资源引用后直接调用，当删除引用后，下次调用Resources.UnloadUnusedAssets后就删除了。AssetBundle.Unload(true)在使用中，最好的做法是给创建出来的实例都添加计数，当计数不为0时，表示场景或代码中仍有引用，而当计数为0时，表示没有引用了，这样就可以放心大胆的AssetBundle.Unload(true)了。

# 资源热更新完整工作流
1. 自己创建定义version.txt文件，此文件包含所有热更新信息，譬如`游戏版本号，资源版本号，ab包以及其manifast信息等`，譬如ab包名称，ab包大小，ab唯一标识符（hash算法，如md5值）等一系列需要与云端文件服务器进行对比的信息，设计好格式之后传送到文件资源服务器。
2. 三方对比，将第一步的version.txt一式两份，本地StreamingAssets一份（或者不放也行，看自己怎么处理了），文件资源服务器一份，游戏首次运行，先下载文件资源服务器的version.txt，并保存到Application.persistentDataPath，然后会拿本地的Application.streamingAssetsPath下的version.txt与Application.persistentDataPath下的version.txt进行对比，记录差异文件，最后统一更新。之后每次进入游戏都会下载文件资源服务器的version.txt与Application.persistentDataPath中的version.txt进行对比，发现差异文件就会记录，然后更新具体ab和更改Application.persistentDataPath中的version.txt数据。更新完毕进入游戏。
3. 资源解压缩，或许我们可以追求更高的网络传输性能，可以再次对准备上传到文件资源服务器的ab进行压缩，下载到本地时再进行解压，能节省更多文件服务器带宽和资源。

# 优秀AssetBundle框架推荐
或许我们并不想把过多的精力花费在这些底层的设计上，那么我们可以选择站在巨人的肩膀上。
GameFramework框架：[https://github.com/EllanJiang/UnityGameFramework](https://github.com/EllanJiang/UnityGameFramework "https://github.com/EllanJiang/UnityGameFramework")
xasset框架：[https://github.com/xasset/xasset.git](https://github.com/xasset/xasset.git "https://github.com/xasset/xasset.git")
