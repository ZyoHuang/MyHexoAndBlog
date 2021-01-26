---
title: Unity DOTS：入门简介（ECS，Burst Complier，JobSystem）
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
终于可以从更深层次上学习ECS了，之前一直停留在浅层次的编码模式（即ECS意识流），没有真正的去了解ECS的内部原理，Unity目前在维护一套以ECS为架构开发的DOTS技术栈，非常值得学习。
# ECS
## 什么是ECS
ECS即实体（Entity），组件（Component），系统（System），其中Entity，Component皆为纯数据向的类，System负责操控他们，这种模式会一定程度上优化我们的代码速度。

- Entities：游戏中的事物
- Components：与Entity相关的数据，但是应该有数据本身而不是实体来组织。（这种组织上的差异正是面向对象和面向数据的设计之间的关键差异之一）
- Systems：Systems是把Components的数据从当前状态转换为下一个状态的逻辑，例如，一个system可能会通过他们的速度乘以从前一帧到这一帧的时间间隔来更新所有的移动中的entities的位置。

## ECS为什么会快
### 计算机组成原理前置知识
首先明确几个知识点

- CPU处理数据的速度非常快，往往会出现CPU处理完数据在那干等着的情况，所以需要设计能跟上CPU的高速缓存区来尽量保证CPU有事干，同时也提高了数据访问效率。
- CPU自身有三级缓存，第一级最快，容量最小，第三级最慢，容量最大。
- 我们常说的内存是指CPU拿取数据的起源点，CPU访问内存的速率远小于三级缓存速率。
- CPU操作数据会先从一，二，三级缓存中取得数据，速度非常快，尤其在一级缓存处速率基本可以满足CPU的需求（即不让CPU歇着），但是有些情况下我们请求的数据不在这三级缓存中（缺页中断），就需要寻址到内存中的数据（包含这个数据的一整块数据都将被存入缓存），并且把目标数据放到三级缓存中，提高下一次的访问速度（因为这一次调用的数据块往往在不久的将来还会用到）。

### ECS的数据组织与使用形式
ECS架构在执行逻辑时，只会操作需要操作的数据，而E和C这两者的配合把相关数据紧密的排列在一起，并且通过Fliter组件过滤掉不需要的数据，这样就减少了`缺页中断`次数，整体上提高了程序效率。
此外现代CPU中的使用数据对齐的先进技术（自动矢量化 即：SIMD）与这种数据密集的架构相性极好，可以进一步提高性能，但是需要有一定的SIMD编程经验（虽然C#编译器内部本来就有做这种SIMD优化）来处理具体的业务逻辑。

## ECS有什么优势
对比传统的面向对象编程，ECS模式无疑更加适合现代CPU架构，因为它可以做到高效的处理数据而不用把多余的数据字段存入宝贵的缓存从而导致多次缺页中断。
举个例子就是传统模式下我们操作Unity对象的Position属性，他会把GameObject所有相关数据都加入缓存，浪费了宝贵的缓存空间。
而如果在ECS模式下，将只会把Position属性集放入内存，节省了缓存空间，也一定程度上减少了缺页频率，即常说的`提高缓存命中率`。

## ECS真有那么神吗
很遗憾，答案是否定的，ECS在正常情景应用下性能和传统模式不分伯仲，有时甚至因为我们额外的管理分类操作而导致反而不如传统模式性能好的情况出现，它只适用于`超多对象的统一管理与操作`情形，这也是我们经常看到的ECS的Demo很震撼的原因之一了。
并且真正启用ECS还是有一定技术门槛的，需要有一定的SIMD编程经验，才能把良好数据管理的优势进一步的发挥出来。

# Unity DOTS
## 什么是Unity DOTS
Unity DOTS就是Unity官方基于ECS架构开发的一套包含Burst Complier技术和JobSystem技术面向数据的技术栈，它旨在充分利用SIMD，多线程操作充分发挥ECS的优势。

## Burst Complier

Burst是使用LLVM从IL/.NET字节码转换为高度优化的本机代码的编译器。它作为Unity package发布，并使用Unity Package Manager集成到Unity中。
它全盘接管了我们编写的新C#编译工作，可以让我们在特定模式下无痛写出高性能代码。
## JobSystem
它可以让我们无痛写出多线程并行处理的代码，并且内部配合Burst Complier进行SIMD优化。
你可以把JobSystem和Unity的ECS一起用，两者配合可以让为所有平台生成高性能机器代码变得简单。

### JobSystem是如何工作的
编写多线程代码可以带来高性能的收益，包括帧率的显著提高，将Burst Compiler和C# JobSystem一起用可以提高生成代码的质量，`这可以大大减少移动设备上的电池消耗`。

C# JobsSystem另一个重要的点是，他和Unity的native jobsystem整合在一起，用户编写的代码和Unity共享线程，这种合作形式避免了创建多于CPU核心数的线程（会引起CPU资源竞争）

## Unity.Mathematics
一个C＃数学库提供矢量类型和数学函数（类似Shader里的语法）。由Burst编译器用来将C＃/IL编译为高效的本机代码。

这个库的主要目标是提供一个友好的数学API（对于熟悉SIMD和图形/着色器的开发者们来说），使用常说的float4，float3类型...等等。带有由静态类math提供的所有内在函数，可以使用轻松将其导入到C＃程序中然后using static Unity.Mathematics.math来使用它。

除此之外，Burst编译器还可以识别这些类型，并为所有受支持的平台（x64，ARMv7a ...等）上为正在运行的CPU提供优化的SIMD类型。

`注意：该API尚在开发中，我们可能会引入重大更改（API和基本行为）`

# 推荐阅读
- 官方的ECSSample开源库：[https://github.com/Unity-Technologies/EntityComponentSystemSamples](https://github.com/Unity-Technologies/EntityComponentSystemSamples "https://github.com/Unity-Technologies/EntityComponentSystemSamples") 
- 官方的ECS文档：[https://docs.unity3d.com/Packages/com.unity.entities@latest/index.html](https://docs.unity3d.com/Packages/com.unity.entities@latest/index.html "https://docs.unity3d.com/Packages/com.unity.entities@latest/index.html")
- 官方的Burst文档：[https://docs.unity3d.com/Packages/com.unity.burst@latest/index.html](https://docs.unity3d.com/Packages/com.unity.burst@latest/index.html "https://docs.unity3d.com/Packages/com.unity.burst@latest/index.html")
- 官方的JobSystem文档：[https://docs.unity3d.com/Manual/JobSystem.html](https://docs.unity3d.com/Manual/JobSystem.html "https://docs.unity3d.com/Manual/JobSystem.html")
- 官方的Unity.Mathematics简介：[https://docs.unity3d.com/Packages/com.unity.mathematics@latest/index.html](https://docs.unity3d.com/Packages/com.unity.mathematics@1.1/manual/index.html "https://docs.unity3d.com/Packages/com.unity.mathematics@latest/index.html")
- 木头哥的ECS教程：[http://www.benmutou.com/archives/category/unity3d/ecs/unityecs_beginner/page/2](http://www.benmutou.com/archives/category/unity3d/ecs/unityecs_beginner/page/2 "http://www.benmutou.com/archives/category/unity3d/ecs/unityecs_beginner/page/2")
- EntherVarope的教程：[https://connect.unity.com/u/enthervarope](https://connect.unity.com/u/enthervarope "https://connect.unity.com/u/enthervarope")
- 十分深入的ECS好文：https://gametorrahod.com/
