---
title: 接入Recast寻路到ET5
date:
updated:
tags: [Unity技术, C++, 寻路, Recast]
categories:
  - - GamePlay
    - 实用工具
  - - 游戏引擎
    - Unity
keywords:
top_img:
cover: https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20201030231659.png
aplayer:
---
<meta name="referrer" content="no-referrer" />

## 前言

因为Unity版本的更新迭代，老版本的A*插件在新版本Unity已经无法正常使用，包括一些运行时代码也已经过时，重新接入要花费很多时间，干脆接入一个新的寻路方案吧。

这里选择的是久负盛名的[https://github.com/recastnavigation/recastnavigation](https://github.com/recastnavigation/recastnavigation)，但因为他是基于C++的，所以我们要使用C#的P/Invoke来调用它的dll来实现寻路。

由于篇幅与操作复杂的原因，本文会更加注重大体的工作流程，而不会有太多细节上的图片，但是会有一个配套的详细教学视频供大家学习，视频链接：[https://www.bilibili.com/video/bv1uK4y1E7CV](https://www.bilibili.com/video/bv1uK4y1E7CV)。

C++/C#的桥接源码也会以在码云开源的形式分享给大家，完整的示例可以在我的Moba项目 [https://gitee.com/NKG_admin/NKGMobaBasedOnET](https://gitee.com/NKG_admin/NKGMobaBasedOnET) 中看到。

通过本文和配套视频你将能学习到recastnavigation的大体设计思路，使用方式，Unity/服务器接入recastnavigation的完整流程。

**感谢@footman大佬在我学习过程中给予的支持！本文中部分内容也来自大佬的文章。**

## 前置内容

Premake认识：[https://blog.csdn.net/wuguyannian/article/details/92175725](https://blog.csdn.net/wuguyannian/article/details/92175725)

SDL认识：[https://baike.baidu.com/item/SDL/224181?fr=aladdin](https://baike.baidu.com/item/SDL/224181?fr=aladdin)

P/Invoke认识：[https://zhuanlan.zhihu.com/p/30746354](https://zhuanlan.zhihu.com/p/30746354)

## 环境

VS 2019以及完整的C++编译环境

Rider For Unreal Engine 2020.2（下面简称Rider）

Unity 2019.4.8 lts

.Net Core 2.2

recastnavigation master：[https://github.com/recastnavigation/recastnavigation/commit/9337e124182697de93acb656ef25766486738807](https://github.com/recastnavigation/recastnavigation/commit/9337e124182697de93acb656ef25766486738807)

## 正文

### 下载并运行recastnavigation Demo

先将下载的SDL库放到

> recastnavigation-master\RecastDemo\Contrib

并且需要改名为SDL，应该得到如下目录

> recastnavigation-master\RecastDemo\Contrib\SDL\lib\x64

然后将下载premake.exe放入

> recastnavigation-master\RecastDemo

然后通过命令行控制premake编译recastnavigation为sln工程

```bash
C:\Users\Administrator>f:

F:\>cd F:\Download\recastnavigation-master\recastnavigation-master\RecastDemo

F:\Download\recastnavigation-master\recastnavigation-master\RecastDemo>premake5 vs2019
Building configurations...
Running action 'vs2019'...
Generated Build/vs2019/recastnavigation.sln...
Generated Build/vs2019/DebugUtils.vcxproj...
Generated Build/vs2019/DebugUtils.vcxproj.filters...
Generated Build/vs2019/Detour.vcxproj...
Generated Build/vs2019/Detour.vcxproj.filters...
Generated Build/vs2019/DetourCrowd.vcxproj...
Generated Build/vs2019/DetourCrowd.vcxproj.filters...
Generated Build/vs2019/DetourTileCache.vcxproj...
Generated Build/vs2019/DetourTileCache.vcxproj.filters...
Generated Build/vs2019/Recast.vcxproj...
Generated Build/vs2019/Recast.vcxproj.filters...
Generated Build/vs2019/RecastDemo.vcxproj...
Generated Build/vs2019/RecastDemo.vcxproj.user...
Generated Build/vs2019/RecastDemo.vcxproj.filters...
Generated Build/vs2019/Tests.vcxproj...
Generated Build/vs2019/Tests.vcxproj.user...
Generated Build/vs2019/Tests.vcxproj.filters...
Done (150ms).
```

然后目录中会生成一个Build文件夹，里面是我们编译出来的sln工程

> recastnavigation-master\RecastDemo\Build\vs2019\recastnavigation.sln

我们用Rider打开新生成的sln文件，不出意外的话我们直接就可以构建并运行RecastDemo工程，也就是我们在它的Github首页看到的逼格超高的图片

![image-20201030231653682](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20201030231659.png)

好了，杯就先装到这，后面会详细介绍UI内容

### 构建用于P/Invoke的dll

让我们新建一个名为RecastNavDll的工程

包含CRecastHelper.cpp/h和RecastDll.cpp/h这四个文件，并编写其逻辑，主要是逻辑封装和为P/Invoke做准备。

然后添加项目Detour和Recast的引用。

最后编辑解决方案属性，准备将我们的RecastNavDll工程导出为x64的dll

![image-20201030233535743](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20201030233535.png)

#### 提前测试

如果按照我给出的源码来编译的话，不会出现逻辑问题，但如果有魔改的需要，可能就需要在C++这边快速测试，所以还会提供一个测试工程MyTest，其中包含MyTest.cpp/h和tools.cpp/h，直接编译并且用终端装载命令行即可进行测试

![image-20201030234043995](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20201030234044.png)

### Unity客户端/.Net Core服务端接入

如果是Unity接入recast很简单，直接将我们导出的RecastNavDll.dll放入Plugins目录下即可。

如果是服务端，我们需要新建子工程，并且添加RecastNavDll.dll链接，并且将csproj文件中CopyToOutputDirectory设置为Always。

```css
    <ItemGroup>
        <None Include="F:\Download\recastnavigation-master\recastnavigation-master\RecastDemo\Build\vs2019\x64\Release\RecastNavDll.dll">
            <Link>RecastNavDll.dll</Link>
            <CopyToOutputDirectory>Always</CopyToOutputDirectory>
        </None>
    </ItemGroup>
```

dll导入之后我们就可以在C#这边搭桥了，我们通过一个RecastInterface.cs文件作为P/Invoke的桥梁

### recastnavigation工作流程

我们可以通过官方自带的RecastDemo来跟踪源码得知工作方式

1. 初始化Recast引擎——recast_init
2. 加载地图——recast_loadmap(int id, const char* path)，id为地图的id，因为我们某些游戏中可能会有多个地图寻路实例，例如Moba游戏，每一场游戏中的地图寻路都是独立的，需要id来区分，path就是寻路数据的完整路径（包含文件名），这个寻路数据我们可以通过RecastDemo来得到
3. 寻路——recast_findpath(int id, const float* spos, const float* epos)，寻路的结果其实只是返回从起点到终点之间所有经过的凸多边形的序号，id为地图id，spos为起始点，epos为中点，我们可以把它们理解为C#中的Vector3
4. 计算实际路径——recast_smooth(int id, float step_size, float slop)，计算平滑路径，其实是根据findpath得到的【从起点到终点所经过的凸多边形的序号】，得到真正的路径（三维坐标），所以这一步是不可缺少的
5. 得到凸多边形id序列——recast_getpathpoly(int id)，得到pathfind以后，从起点到终点所经过的所有凸多边形id的序列
6. 得到寻路路径坐标序列——recast_getpathsmooth(int id)，得到smooth以后，路线的三维坐标序列
7. 释放地图——recast_freemap(int id)，游戏结束后记得释放地图寻路数据资源嗷
8. 释放Recast引擎——recast_fini()，如果我们在客户端使用，游戏流程结束要使用这个释放Recast引擎

### 性能测试

大家一定很关心recastnavigation库的性能如何，我做了个压测

以这张地图为例，进行随机寻路（包括合法点和非法点）

![image-20201030231653682](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20201031100628.png)

```cs
        public static void BenchmarkSample()
        {
            BenchmarkHelper.Profile("寻路100000次", BenchmarkRecast, 100000);
        }

        private static void BenchmarkRecast()
        {
            if (RecastInterface.FindPath(100,
                new System.Numerics.Vector3(-RandomHelper.RandomNumber(2, 50) - RandomHelper.RandFloat(),
                    RandomHelper.RandomNumber(-1, 5) + RandomHelper.RandFloat(), RandomHelper.RandomNumber(3, 20) + RandomHelper.RandFloat()),
                new System.Numerics.Vector3(-RandomHelper.RandomNumber(2, 50) - RandomHelper.RandFloat(),
                    RandomHelper.RandomNumber(-1, 5) + RandomHelper.RandFloat(), RandomHelper.RandomNumber(3, 20) + RandomHelper.RandFloat())))
            {
                RecastInterface.Smooth(100, 2f, 0.5f);
                {
                    int smoothCount = 0;
                    float[] smooths = RecastInterface.GetPathSmooth(100, out smoothCount);
                    List<Vector3> results = new List<Vector3>();
                    for (int i = 0; i < smoothCount; ++i)
                    {
                        Vector3 node = new Vector3(smooths[i * 3], smooths[i * 3 + 1], smooths[i * 3 + 2]);
                        results.Add(node);
                    }
                }
            }
        }
```

得到的结果是

| 寻路次数 | 耗时        |
| -------- | ----------- |
| 100      | 1.4293ms    |
| 10000    | 152.3692ms  |
| 100000   | 1365.5992ms |

完全可以满足服务器寻路需求，另外这还是单线程寻路，可以自己做多线程优化

### ET框架接入recastnavigation

我们要替换服务端的寻路组件，最后只需要RecastPathComponent和RecastPathProcessor两个文件即可完成寻路。

### 完整工作流

我们要先在Unity中使用NavMesh烘焙，然后使用NavMesh/Export Scene工具导出obj文件，然后在RecastDemo中进行读取并烘焙

1. Sample-Solo Mesh
2. Input Mesh-我们导出的obj文件，需要放到recastnavigation-master\RecastDemo\Bin\Meshes目录下
3. 调制参数，Build 
4. Save保存为nav文件，会导出为recastnavigation-master\RecastDemo\Bin\solo_navmesh.bin

**通过对比我们发现它与Unity中坐标点差别就是x轴坐标是相反的，所以我们进行寻路的时候对数据进行了处理。**

![image-20201031122521129](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20201031122521.png)

然后我们把导出的bin文件放到我们ET项目中，路径随意，只要在程序里定义好就行，比如我这里是 **项目根目录\Config\RecastNavData\solo_navmesh.bin**

服务端程序中读取就是读取的这个路径

```cs
/// <summary>
/// 5v5地图的Nav数据路径
/// </summary>
public const string Moba5V5MapNavDataPath = "../Config/RecastNavData/solo_navmesh.bin";
```

进行寻路即是

```cs
RecastPathComponent recastPathComponent = Game.Scene.GetComponent<RecastPathComponent>();
RecastPath recastPath = ReferencePool.Acquire<RecastPath>();
recastPath.StartPos = unit.Position;
recastPath.EndPos = new Vector3(target.x, target.y, target.z);
self.RecastPath = recastPath;
//TODO 因为目前阶段只有一张地图，所以默认mapId为10001
recastPathComponent.SearchPath(10001, self.RecastPath);
```

