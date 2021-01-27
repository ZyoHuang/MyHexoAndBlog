---
title: GameFramework篇：VFS详解
date:
updated:
tags: [Unity技术, 游戏框架, GameFramework]
categories:
  - - 游戏引擎
    - Unity
  - - GamePlay
    - 游戏框架
cover: https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190105164355598.png
keywords:
top_img:
aplayer:
---
<meta name="referrer" content="no-referrer" />

# GF的VFS

### 简介

虚拟文件系统（VFS）使用类似磁盘的概念对零散文件进行集中管理，优化资源加载时产生的内存分配，甚至可以对资源进行局部片段加载，这些都将极大提升资源加载时的性能。

官方文档：https://gameframework.cn/document/filesystem/

前置知识：[计算机操作系统磁盘相关知识](https://www.cnblogs.com/Benjious/p/10041285.html)



### 原理

通过将许多小文件集合成一个大文件的方式来集中管理零散资源，减少物理磁盘寻址次数从而得以提高性能，并且这个过程是我们自定义的，可以进行加密操作，提高安全性。



### 学习方式

#### 环境准备

先去下载最新的Starforce：https://github.com/EllanJiang/StarForce，以及Gamrframework：https://github.com/EllanJiang/GameFramework

然后使用我们下载的Gamrframework源码去替换Starforce中的Gameframework，传统艺能了，还不知道怎么操作的小伙伴可以去：https://www.bilibili.com/video/BV14E411a71o

VFS只有在开启非编辑器资源模式下才有效，所以我们要关闭GameFramework场景下的BuildIn上Base的Editor Resource Mode选项

![image-20200719155015598](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200719155015598.png)

然后我们需要打AB包，因为下载的Demo已经帮我们配置好了所有的AB规则以及VFS相关的东西，所以我们不需要做额外的操作，直接使用Resource Builder进行打包即可

![image-20200719155618849](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200719155618849.png)

![image-20200719155737605](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200719155737605.png)

当然如果我们后面要自定义AB内容的话还是要知道怎么配置的

![image-20200719155824241](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200719155824241.png)

![image-20200719155904718](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200719155904718.png)

这个窗口中的所有内容数据都来自ResourceCollection.xml文件，图中圈出来的部分就是我们FileSystem的配置，需要我们自己手动配置

![image-20200719160032105](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200719160032105.png)

打包好之后，我们StreamingAssets目录如下图所示，之前版本的Starforce打包后会有很多的AB文件，但是我们可以看到，现在只有这几个AB文件了，这是因为GF打包时根据我们设置的VFS而对资源进行了拼接，把多个资源合并成了一个，现在环境即准备完毕

![image-20200719160201537](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200719160201537.png)



#### 查看VFS的总体结构

我们可以通过对FileSystemComponentInspector.cs的OnInspector部分进行断点，可以直观的查看VFS总体结构，然后我们点击运行游戏

并且点击

![image-20200719160350463](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200719160350463.png)

即可进入我们的断点，然后我们就可以查看内容了

![image-20200719160523440](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200719160523440.png)



### 架构

![GF_VFS架构](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/GF_VFS架构.png)

具体含义解释：

- FileSystemStream：一个VFS对应的文件流类，具体有两个实现，一个是常规VFS，一个是安卓特化VFS（因为安卓不能通过System.FileStream去搞StreamingAssets）
- HeaderData：概述整个VFS的Info，包括最大文件数，最大数据块数，加密信息等
- StringData：用于描述VFS中单个File的文件名（加密）
- BlockData：用于描述VFS中单个File的Info，包括对应StringData索引（文件名），簇索引（用于计算偏移），文件长度，每次进行文件写入，删除操作时都会涉及到BlockData的改变，所以如果需要对一个VFS进行写入的话，就需要提供大于真正MaxFileCount的MaxBlockCount，也就有了下面的2 ~ 32倍之说
- FileInfo：用于描述单个File的Info，包括名称，偏移，长度（解密后）
- FileSystem：VFS的具体实现，提供对File的**增删改查**等接口
- FileSystemManger：管理项目中所有的VFS



### 使用方法

获取文件系统组件

```c#
FileSystemComponent fileSystemComponent = GameEntry.GetComponent<FileSystemComponent>();
```

检查是否存在文件系统

```c#
string fullPath = Path.Combine(Application.persistentDataPath, "FileSystem.dat");

// 检查是否存在文件系统，参数要传递的是文件系统的完整路径
bool hasFileSystem = fileSystemComponent.HasFileSystem(fullPath);
```

获取文件系统

```c#
string fullPath = Path.Combine(Application.persistentDataPath, "FileSystem.dat");

// 获取文件系统，参数要传递的是文件系统的完整路径
IFileSystem fileSystem = fileSystemComponent.GetFileSystem(fullPath);
```

创建文件系统

```c#
// 要创建的文件系统的完整路径
string fullPath = Path.Combine(Application.persistentDataPath, "FileSystem.dat");

// 要创建的文件系统最大能容纳文件数量
int maxFileCount = 16;

// 要创建的文件系统最大能容纳的块数据数量
int maxBlockCount = 256;

// 创建文件系统（使用读写模式进行访问）
IFileSystem fileSystem = fileSystemComponent.CreateFileSystem(fullPath, FileSystemAccess.ReadWrite, maxFileCount, maxBlockCount);

// 创建文件系统（使用只写模式进行访问）
IFileSystem fileSystem = fileSystemComponent.CreateFileSystem(fullPath, FileSystemAccess.Write, maxFileCount, maxBlockCount);
```

> 文件系统块数据（Block）是 Game Framework 文件系统中引入的一个数据结构，用于索引文件内容数据区在文件系统中的偏移、长度等信息。即使文件从文件系统中被删除，文件系统块数据依然会保留，并将原先的文件内容数据区标记为文件系统碎片。
>
> 文件系统当前已使用的块数据数量 = 文件系统当前文件数量 + 文件系统当前碎片数量
>
> 显然，一个文件系统最大能容纳的块数据数量不应少于最大能容纳的文件数量。
> 当文件系统增加新文件时，会优先分配合适的文件系统碎片的空间用于存储新文件。

加载文件系统

```c#
// 要加载的文件系统的完整路径
string fullPath = Path.Combine(Application.persistentDataPath, "FileSystem.dat");

// 加载文件系统（使用读写模式进行访问）
IFileSystem fileSystem = fileSystemComponent.LoadFileSystem(fullPath, FileSystemAccess.ReadWrite);

// 加载文件系统（使用只读模式进行访问）
IFileSystem fileSystem = fileSystemComponent.LoadFileSystem(fullPath, FileSystemAccess.Read);
```

销毁文件系统

```c#
// 销毁文件系统，传入已创建或者已加载的文件系统
// 第二个参数 deletePhysicalFile 指示是否删除文件系统对应的物理文件，传 true 时要小心
fileSystemComponent.DestroyFileSystem(fileSystem, false);
```

获取文件系统数量

```c#
// 获取文件系统数量
int fileSystemCount = fileSystemComponent.Count;
```

获取所有文件系统集合

```c#
// 使用返回数组的方式，获取所有文件系统集合
IFileSystem[] fileSystems = fileSystemComponent.GetAllFileSystems();

// 使用填充列表的方式，获取所有文件系统集合
List<IFileSystem> fileSystems = new List<IFileSystem>();
fileSystemComponent.GetAllFileSystems(fileSystems);
```

文件系统相关操作

``` c#
// 获取文件系统完整路径
string fullPath = fileSystem.FullPath;
 
// 获取文件系统访问方式
FileSystemAccess access = fileSystem.Access;
 
// 获取文件数量
int fileCount = fileSystem.FileCount;
 
// 获取最大文件数量
int maxFileCount = fileSystem.MaxFileCount;
 
// 获取文件信息，包括文件名称、文件偏移、文件长度信息
FileInfo fileInfo = fileSystem.GetFileInfo("FileName.dat");
 
// 使用返回数组的方式，获取所有文件信息
FileInfo[] fileInfos = fileSystem.GetAllFileInfos();
 
// 使用填充列表的方式，获取所有文件信息
List<FileInfo> fileInfos = new List<FileInfo>();
fileSystem.GetAllFileInfos(fileInfos);
 
// 检查是否存在指定文件
bool hasFile = fileSystem.HasFile("FileName.dat");
 
// 使用返回数组的方式，读取指定文件
byte[] bytes = fileSystem.ReadFile("FileName.dat");
 
// 使用填充数组的方式，读取指定文件
byte[] buffer = new byte[X]; // 读取文件使用的 buffer，用此方式能够复用 buffer 来消除 GCAlloc
int startIndex = 0;
int length = buffer.Length - startIndex;
int bytesRead = fileSystem.ReadFile("FileName.dat", buffer);
int bytesRead = fileSystem.ReadFile("FileName.dat", buffer, startIndex);
int bytesRead = fileSystem.ReadFile("FileName.dat", buffer, startIndex, length);
 
// 使用填充流的方式，读取指定文件
Stream stream = new X; // 读取文件片段使用的 stream，用此方式能够复用 buffer 来消除 GCAlloc
int bytesRead = fileSystem.ReadFile("FileName.dat", stream);
 
// 使用返回数组的方式，读取指定文件片段
byte[] bytes = fileSystem.ReadFileSegment("FileName.dat", length);
byte[] bytes = fileSystem.ReadFileSegment("FileName.dat", offset, length);
 
// 使用填充数组的方式，读取指定文件片段
int offset = M; // 要读取片段的偏移
int length = N; // 要读取片段的长度
byte[] buffer = new byte[X]; // 读取文件片段使用的 buffer，用此方式能够复用 buffer 来消除 GCAlloc
int bytesRead = fileSystem.ReadFileSegment("FileName.dat", buffer, length);
int bytesRead = fileSystem.ReadFileSegment("FileName.dat", buffer, startIndex, length);
int bytesRead = fileSystem.ReadFileSegment("FileName.dat", offset, buffer, length);
int bytesRead = fileSystem.ReadFileSegment("FileName.dat", offset, buffer, startIndex, length);
 
// 使用填充流的方式，读取指定文件片段
Stream stream = new X; // 读取文件片段使用的 stream，用此方式能够复用 buffer 来消除 GCAlloc
int bytesRead = fileSystem.ReadFileSegment("FileName.dat", stream, length);
int bytesRead = fileSystem.ReadFileSegment("FileName.dat", offset, stream, length);
 
// 将字节数组写入指定文件
bool result = fileSystem.WriteFile("FileName.dat", buffer);
bool result = fileSystem.WriteFile("FileName.dat", buffer, startIndex);
bool result = fileSystem.WriteFile("FileName.dat", buffer, startIndex, length);
 
// 将流写入指定文件
bool result = fileSystem.WriteFile("FileName.dat", stream);
 
// 将物理文件写入指定文件
bool result = fileSystem.WriteFile("FileName.dat", @"E:\PhysicalFileName.dat");
 
// 指定文件另存为物理文件
bool result = fileSystem.SaveAsFile("FileName.dat", @"E:\PhysicalFileName.dat");
 
// 重命名指定文件
bool result = fileSystem.RenameFile("OldFileName.dat", "NewFileName.dat");
 
// 删除指定文件
bool result = fileSystem.DeleteFile("FileName.dat");
```



# 经典问答环节

### 为什么AndroidStream要封装AndroidJavaObject？

- 安卓不能直接用FileStream访问StreamingAssets，虽然可以通过自行实现算法来读取apk，但考虑到代码可读性，GF选择使用调用Java库函数来读取StreamingAssets里的VFS
- 在设计上为了支持操作Android StreamingAssets里存储的VFS文件（如果不使用streamingAssets目录的话确实可以不用），比如GameData.dat可以在StreamingAssets下和persistentDataPath下分别存为俩VFS



### 一些VFS适用场景

例如，在制作玩家聊天自定义表情（玩家可以自行上传表情图片并通过聊天频道发送给其他玩家）的时候，可以考虑将玩家上传和下载的自定义表情图片存储于一个由游戏逻辑创建并管理的 CustomEmotions 文件系统中，而不是将这些图片散放于磁盘上，这样将很好的提升磁盘 IO 性能并能一定程度上提升安全性。



### VFS会对资源管理流程产生影响吗？

用不用VFS在资源管理上没有任何区别，只不过是不是把散资源打包的区别，文件更新的时候，散资源挪入VFS，VFS挪成散资源，从一个VFS挪入另一个VFS，都是可以的。



### VFS设计相关

- 然后当前VFS的寻址空间并不限制，不过鉴于手机系统和Unity的BUG，不要超过4GB为好，Unity2020之后才支持的从4GB以后的偏移处加载AB，2020.1 解除了这个限制。https://forum.unity.com/threads/bug-4gb-limit-to-textures-in-standalone-build.441116/page-5#post-5300130
- VFS的设计理念是功能全又不失轻巧，因为一开始不想让VFS占内存，包括VFS头部信息都不占内存，是个纯磁盘行为，后来考虑手机IO性能和发热问题，还是拿出点内存换取性能
- 读写文件时考虑使用带有 buffer 缓存的重载方法，可以预先准备好一个 byte[] buffer 作为缓存，进而复用此 buffer 来消除 GCAlloc。
- 读取文件时考虑使用读取文件片段方法，对于数据表、本地化字典表这类文本或二进制数据，在实际应用的多数情形下，将全部内容加载到内存中是非常浪费的，对于这类数据不妨在使用某条数据时再去读取这条数据，即所谓的懒加载或者延迟加载。通过合理构造数据结构并利用文件系统能够读取文件片段的特性，即可做到这一点。例如，先读取字典表的全部 key 值，并记录每个 value 值对应的偏移。在需要获取某个 key 的 value 值时，根据已经记录的偏移值利用读取文件片段的方法实时加载 value 值并缓存。
- 创建文件系统时，合理设置 MaxFileCount 和 MaxBlockCount创建文件系统时，需要设置 MaxFileCount（能容纳的最大文件数量）和 MaxBlockCount（能容纳的最大块数据数量），这两个参数决定了文件系统初始化化时头部块的大小。一般来说，用于存储不变资源的只读文件系统，假设需要写入 N 个文件，那么 MaxFileCount 和 MaxBlockCount 都设置为 N 即可。
  用于存储可变资源的文件系统，根据预估文件数量，设置合适的 MaxFileCount 即可，MaxBlockCount 可根据此文件系统中文件更新的频率，设置为 MaxFileCount 的 2 ~ 32 倍是比较合适的，文件系统中的文件被更新频率越高，倍数可以相应设置的越高。当文件系统碎片占用的块数据过多导致所有块数据被填满时，等同于触发文件系统容量已满，无法再写入新文件。*备注：在后续版本的文件系统中，Game Framework 有计划引入 MaxFileCount 和 MaxBlockCount 的自动扩充机制和文件系统碎片整理机制。*



### VFS未来展望

VFS后续，可能很往后，会加自动扩充最大文件和最大block，看需求吧，还可以加碎片整理方法。

