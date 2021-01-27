---
title: Unity DOTS：ECS的核心部分
date:
updated:
tags: [Unity技术, 多线程, ECS, Unity DOTS]
categories:
  - - 游戏引擎
    - Unity
  - - GamePlay
    - ECS
keywords:
top_img:
cover: https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2020/05/nordeus_jobsystem_2.jpg
aplayer:
---
<meta name="referrer" content="no-referrer" />

# ECS的概念

实体组件系统（ECS）架构将身份（Entities，实体），数据（Components，组件）和行为（Systems，系统）分开。该架构专注于数据。Systems读取Components数据流，然后将数据从输入状态转换为输出状态，然后对这些实体进行索引。

下图说明了这三个基本部分如何协同工作：
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2020/05/QQ截图20200508230036.png)
在此图中，系统读取Translation和Rotation部分，将它们相乘，然后更新相应的LocalToWorld成分`（L2W = T*R）`。

`实体A和B具有Renderer组件，而实体C则没有，但是这不会影响系统，因为系统并不关心Renderer组件。`

您可以设置一个系统，使其需要一个Renderer组件，在这种情况下，系统将忽略实体C的组件。或者，您也可以设置系统以排除具有Renderer组件的实体，然后忽略实体A和B的组件。

## Archetypes(原型)
`组件类型的组合称为“原型”。`例如，一个3D对象可能具有一个用于其世界变换的组件，一个用于其线性运动的组件，一个用于旋转的组件和一个用于渲染的组件。这些3D对象的每个实例都对应一个实体，但是由于它们共享相同的组件集，因此ECS将它们归类为单个原型：
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2020/05/QQ截图20200508230637.png)
在此图中，实体A和B共享原型M，而实体C归属于原型N。

为了顺利更改实体的原型，可以在运行时添加或删除组件。例如，`如果您从实体B中删除Renderer组件，那么它将移至原型N。`
## Memory Chunks(内存块)
实体的原型决定ECS在何处存储该实体的组件。`ECS以“块”分配内存，每个块均由一个ArchetypeChunk对象表示。块始终包含单个原型的多个实体。当内存块已满时，ECS会为使用相同原型创建的任何新实体分配新的内存块。如果添加或删除组件，然后更改了实体原型，则ECS会将该实体的组件移动到其他块中。`

这种组织方案在原型和块之间提供了一对多的关系。这也意味着，`找到具有给定组件集的所有实体仅需要搜索通常数量很少的现有原型`，而不是搜索通常数量更大的所有实体。

ECS不会按特定顺序存储块中的实体。创建实体或将其更改为归属新的原型时，ECS会将其放入存储该原型且具有空闲空间的第一个块中。块仍然紧密排布。当从原型中移除实体时（可以是移除一个/多个Component，也可以直接移除Entity），ECS会将块中最后一个实体的组件移动到组件阵列中新腾出的插槽中（即被移除Entity的所在位置）。

注意：原型中共享组件的值还决定了哪些实体存储在哪个块中。指定块中的所有实体的共享组件都具有完全相同的值。`如果更改共享组件中任何字段的值，则修改后的实体将移至其他块，就像更改该实体的原型时一样。如有必要，将分配一个新块`。

使用共享组件对原型中的实体进行分组可以更有效地将它们一起处理。例如，混合渲染器定义其RenderMesh组件以达成此目的。

## Entity queries（实体查询）
`要确定系统应处理的实体，请使用EntityQuery`。实体查询在现有原型中搜索具有与您的需求相匹配的组件的原型。您可以通过查询指定以下组件要求：

- All-全部 ：原型必须包含“全部”类别中的所有组件类型。
- Any-任意 ：原型必须包含“任意”类别中的至少一种组件类型。
- None-无 ：原型在“无”类别中不得包含任何组件类型。

实体查询返回了我们所需要的组件类型的块的列表。然后，您可以直接使用IJobChunk遍历那些块中的组件。

## Jobs
要利用多个线程，可以使用`C＃Job system`。ECS提供SystemBase类`Entities.ForEach`以及`IJobChunk Schedule()`和`ScheduleParallel()`方法，以将数据转换到主线程之外。Entities.ForEach是最简单的使用方法，通常只需较少的代码行即可实现。您可以将IJobChunk用于Entities.ForEach无法处理的更复杂的情况。

`ECS按照你安排的那些Systems的顺序在主线程上调度Job（作业）。`在Schedule（调度）作业后，ECS会持续跟踪读取和写入组件的Job。读取组件的Job取决于写入同一组件的任何先前的预定Job，反之亦然。`作业调度使用作业的依赖性来确定哪些作业可以并行运行，哪些必须按顺序运行。`

## System organization（系统组织）
`ECS按World->Group的形式组织系统。`默认情况下，ECS将使用一组预定义的组来创建默认世界。它找到所有可用的系统，实例化它们，并将它们添加到默认世界中的预定义模拟组中。

`您可以指定同一组中系统的更新顺序。`群组是一种系统，因此您可以像其他系统一样将群组添加到另一个群组并指定其顺序。组中的所有系统在下一个系统或组之前进行更新。如果未指定顺序，则ECS将以不依赖于创建顺序的确定性方式将系统插入更新顺序。换句话说，即使您未明确指定顺序，同一组系统也始终以其组内的相同顺序进行更新。[实体组件缓冲系统](xref:Unity.Entities.EntityComponentBufferSystem "实体组件缓冲系统")

系统更新在主线程上进行。但是，系统可以使用作业将工作分流到其他线程。SystemBase提供了创建和调度作业的直接方法。

有关系统创建，更新顺序以及可用于组织系统的属性的更多信息，请参阅[系统更新顺序](https://docs.unity3d.com/Packages/com.unity.entities@0.10/manual/system_update_order.html "系统更新顺序")上的文档。

## ECS authoring（ECS创作）
在Unity编辑器中创建游戏或应用程序时，可以使用GameObjects和MonoBehaviours创建转换系统，以将这些UnityEngine对象和组件映射到实体。有关更多信息，请参见[创建游戏玩法](https://docs.unity3d.com/Packages/com.unity.entities@0.10/manual/gp_overview.html "创建游戏玩法")。
