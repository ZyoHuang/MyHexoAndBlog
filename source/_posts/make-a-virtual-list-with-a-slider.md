---
title: 制作一个带有滑块的虚拟列表
date:
updated:
tags: Unity技术
categories:
  - - GamePlay
    - 实用工具
  - - 游戏引擎
    - Unity
keywords:
top_img:
cover: https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2020/02/QQ截图20200218165835.png
aplayer:
---
<meta name="referrer" content="no-referrer" />

# 前言
昨天面试被问到带有滑块的虚拟列表同步问题，一时间没Get到点，因为一直用的FGUI，今天用UGUI一试，果真有面试官所说的问题——只要对象Active为False，滑块就会出现问题，与当前真实位置出现偏差。
所以今天就来详细的了解，解决一下这个问题。
## 什么是虚拟列表
如果列表的item数量特别多时，例如几百上千，为每一条项目创建实体的显示对象将非常消耗时间和资源。只为显示范围内的item创建实体对象，并通过动态设置数据的方式实现大容量列表，这就是虚拟列表。
总而言之是一种优化手段，它只会渲染当前可见范围的数据，所以同时`最多只有当前列表所能显示容量+2个对象`，因为当列表滑动时，会出现一些元素并没有完全显现或者完全消失的情况。
# 正文
## 思路
我先去看了下FGUI对于这块处理的源码，主要思路如下：
## 准备工作
假如当前虚拟列表容量为1000，就需要创建1000个内部ItemInfo对象，这是必须的，因为我们需要记录下每一个游戏对象的渲染状态和内容。
```csharp
public class ItemInfo
{
	//元素位置
    public Vector2 Pos;
	//具体元素游戏对象（渲染对象）
    public GameObject obj;
}

public List<ItemInfo> ItemInfos = new List<ItemInfo>();
```
同样的，我们需要创建1000个渲染对象所需要的数据对象，这里为了方便演示，直接用`List<string>`代替。
```csharp
/// <summary>
/// 所有的string内容，用于给元素内容赋值，并保存内容
/// </summary>
public List<string> AllContents = new List<string>();
```
然后就是我们具体的渲染对象，他只需要`当前列表所能显示容量+2`个。
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2020/02/QQ截图20200218162548.png)
然后是滑块的处理，滑块本身是依据ContentView的大小所改变，所以有两个选择：

1. 使用ScrollRect索引滑块，提前计算ContentView的Rect大小
2. 单独做一个滑块，自己运行时处理ContentView大小

## 算法原理
### 我的弱智算法

首先是确认渲染元素消失，出现的临界点，那就是ContentView的顶部和底部
即：`如果此时列表中的每个元素都往下运动，每当一个渲染元素自身的顶部小于等于ContentView底部，说明他该被隐藏了，每当一个渲染元素自身的底部大于等于ContentView顶部，说明他该被显示了，如果方向相反，则规则相反`。
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2020/02/QQ截图20200218165835.png)
当然了，这个算法缺陷很明显了，如果一次性拖动太多进度，会出现大片空白，甚至是需要延迟1,2s数据才会刷新出来，虽然可以通过一些手段优化，但总感觉没内味。。。

### 钱康来大佬的算法
其实有一个更好的算法，就是直接轮询所有的ItemInfo，依据他们的Pos判断是否在可视范围内，如果在，且obj为空，需要从对象池拿到obj然后赋值，如果不在，且obj不为空，需要把它回收到对象池。伪代码如下
```csharp
for(int i=0;i<ItemInfos.Length;i++)
{
	if(ItemInfos[i].Pos在可视范围内)
	{
		if(ItemInfos[i].obj==null)
		{
			从对象池取出一个GameObject，并将其赋值给ItemInfos[i].obj;
		}
	}
	else//如果不在
	{
		if(ItemInfos[i].obj!=null)
		{
			将ItemInfos[i].obj回收到对象池中，并将ItemInfos[i].obj设置为null;
		}
	}
}
```
其中最重要的一点就是如何判断在不在当前视角范围内，大佬的做法是使用了一个个Bound做对比。
### 最后
当然了，这只是虚拟列表最基本的功能，还有很多譬如立即滑动，定位到某个元素，实时增删改查列表，这些都是挑战。

## 一些坑
如果使用Unity的LayoutGroup组件，需要注意Rebuild问题，不然会有一些问题（鬼畜）

## 拓展
钱康来大佬做了一个非常完善的无限循环滚动列表，所以我就不献丑了，下面是链接：
[https://github.com/qiankanglai/LoopScrollRect](https://github.com/qiankanglai/LoopScrollRect "https://github.com/qiankanglai/LoopScrollRect")
