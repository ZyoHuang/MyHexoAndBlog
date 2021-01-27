---
title: Unity DOTS：Components部分
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

# 组件

Components是ECS体系结构的三个主要元素之一。它们代表您的游戏或应用程序的数据。[Entities](https://docs.unity3d.com/Packages/com.unity.entities@0.13/manual/ecs_entities.html)是索引您的components集合的标识符，而systems提供了行为。

ECS中的Components是具有以下“标记接口”之一的结构体类型：

- [IComponentData](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.IComponentData.html) —用于[通用](https://docs.unity3d.com/Packages/com.unity.entities@0.13/manual/component_data.html)和[chunk components]。
- [IBufferElementData](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.IBufferElementData.html) —将[动态缓冲区（dynamic buffers）]与entities相关联。
- [ISharedComponentData](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.ISharedComponentData.html) —按archetype中的值对entities进行分类或分组。有关更多信息，请参见[Shared Component Data]。
- [ISystemStateComponentData](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.ISystemStateComponentData.html) —将特定system的状态与entity相关联，并检测何时创建或销毁单个实体。有关更多信息，请参见[System State Component](https://docs.unity3d.com/Packages/com.unity.entities@0.13/manual/system_state_components.html)。
- [ISharedSystemStateComponentData](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.ISystemStateSharedComponentData.html) —共享状态和System状态 数据的组合。请参阅[System State Component](https://docs.unity3d.com/Packages/com.unity.entities@0.13/manual/system_state_components.html)。
- [Blob assets](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.BlobBuilder.html) –从技术上讲，它不是“component”，但您可以使用Blob assets来存储数据。Blob assets可以由一个或多个component使用[BlobAssetReference](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.BlobAssetReference-1.html)进行引用，并且他是不可变的。您可以使用Blob assets在资产之间共享数据并访问C＃ jobs中的数据。

EntityManager将components的独特组合组织到**archetypes**。archetypes将归属相同的archetypes的的components一起存储在称为*chunks*的内存块中。给定chunk中的entities均具有相同的components archetypes。



![image-20200811174536423](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200811174536423.png)



该图说明了ECS如何通过存archetypes存储components数据的chunk。*shared components和chunk components是例外*，因为ECS将它们存储在chunk外部。这些component类型的单个实例适用于他所适用的chunks中的所有entities。另外，您可以选择将dynamic buffers存储在chunk之外。即使ECS不在chunk内存储这些类型的components，您在查询entities时通常也可以将它们与其他components类型相同对待。

# 通用组件

`ComponentData`在标准ECS术语中称为组件数据，是仅包含[entities](https://docs.unity3d.com/Packages/com.unity.entities@0.13/manual/ecs_entities.html)的实例数据的结构。`ComponentData`不应包含除Utility（来访问结构中的数据的方法）功能之外的方法。您应该在systems中实现所有游戏逻辑和行为。用面向对象的Unity系统来讲，这有点类似于Component（经典挂载MonoBehaviour）类，但是**只包含变量，不包含逻辑**。

Unity ECS API提供了一个名为[IComponentData的接口](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.IComponentData.html)，您可以在代码中实现该接口以表示他是general components类型。

## IComponentData

传统的Unity Component是[面向对象的](https://en.wikipedia.org/wiki/Object-oriented_programming)类，其中包含数据和方法。`IComponentData`是纯ECS的组件，这意味着它没有定义任何行为，仅定义了数据。您应该实现`IComponentData`为struct而不是class，这意味着默认情况下，它是[通过值而不是通过引用](https://stackoverflow.com/questions/373419/whats-the-difference-between-passing-by-reference-vs-passing-by-value?answertab=votes#tab-top)复制的。通常，您需要使用以下模式来修改数据：

```CSharp
var transform = group.transform[index]; // 读

transform.heading = playerInput.move; // 改
transform.position += deltaTime * playerInput.move * settings.playerMoveSpeed;

group.transform[index] = transform; // 写
```

*`IComponentData`不得包含对托管对象的引用。*这是因为`ComponentData`存在于在没有被GC跟踪的[块内存中](https://docs.unity3d.com/Packages/com.unity.entities@0.13/manual/chunk_iteration.html)，所以具有许多性能优势。

### 托管的IComponentData

使用托管形式的`IComponentData`（即，`IComponentData`声明为`class`而不是声明为`struct`）有助于将现有代码以一把梭的方式移植到ECS，`ISharedComponentData`不适合与托管数据进行交互操作或为数据布局提供原型。

这些托管的组件的使用方式与值类型的`IComponentData`相同。但是，ECS在内部以完全不同（且较慢）的方式来处理它们。如果不需要托管组件支持，请在应用程序的**Player Settings** (菜单: **Edit > Project Settings > Player > Scripting Define Symbols**)）添加`UNITY_DISABLE_MANAGED_COMPONENTS`宏，以防止意外使用。

因为托管`IComponentData`是托管类型，所以与值类型的`IComponentData`相比，它具有以下性能缺点：

- 不能与Burst编译器一起使用
- 不能在job结构中使用
- 它不能使用[chunk memory](https://docs.unity3d.com/Packages/com.unity.entities@0.13/manual/chunk_iteration.html)
- 需要被GC

您应该尽量限制托管组件的数目，并且尽可能使用[blittable](https://docs.microsoft.com/en-us/dotnet/framework/interop/blittable-and-non-blittable-types)类型

托管对象`IComponentData`必须实现`IEquatable<T>`接口并重写`Object.GetHashCode()`。此外，出于序列化的目的，托管组件必须是默认可构造的。

您必须在主线程上设置托管component的值。为此，请使用 `EntityManager`或`EntityCommandBuffer`。由于托管component是引用类型，因此与[ISharedComponentData](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.ISharedComponentData.html)不同，您可以更改组件的值而无需在chunk之间移动entities。但是这不会创建同步点。

但是，尽管在逻辑上将托管组件与值类型的组件分开存储，但它们仍然依托于实体的`EntityArchetype`定义。这样，向实体添加新的托管组件仍然会导致ECS创建新的archetype（如果尚不存在匹配的archetype），并将实体移至新的Chunk。

有关示例，请参见文件：`/Packages/com.unity.entities/Unity.Entities/IComponentData.cs`。

# Shared component data

shared components是一种特殊的数据组件，您可以根据shared components中的特定值（但不包括他们archetype的）对entities进行细分。将shared component添加到一个entity时，EntityManager会将具有相同shared component的（struct意义上的相等）所有entities放入同一chunk中。

shared components使您的systems可以像entities一样一起处理。例如，shared components的`Rendering.RenderMesh`是Hybrid.rendering包的一部分，它定义了几个字段，包括**mesh**，**material**和**receiveShadows**。当您的应用程序渲染时，最有效的方法是一起处理所有具有相同字段值的3D对象。因为shared components指定了这些属性，所以EntityManager将与其匹配的所有entities放置在内存中，以便渲染系统可以高效地对其进行迭代。

**注意：**如果您过度使用共享组件，则可能导致不佳的chunk利用率。这是因为当您使用shared components时，它将触发基于archetype和每个shared components的特定字段的组合扩张。因此，请避免添加任何将实体分类到shared components中不需要的字段。要查看chunk块利用率情况，请使用[Entity Debugger](https://docs.unity3d.com/Packages/com.unity.entities@0.13/manual/ecs_debugging.html)。

如果您从entity中添加或删除component，或更改shared component的值，则EntityManager会将entity移至其他chunk，并在必要时创建新chunk（但是和这个entity同chunk的其余entities不会变动）。

您应该将[IComponentData](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.IComponentData.html)用于存储分类不同entities的数据，例如存储世界位置，对象生命值或粒子生存时间。当许多entities共享某些共同点时，应使用[ISharedComponentData](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.ISharedComponentData.html)。例如，在DOTS软件包的Boids演示中，许多entities都从同一[Prefab](https://docs.unity3d.com/Manual/Prefabs.html)实例化，结果，许多`Boid`实体之间`RenderMesh`完全相同。

```cs
[System.Serializable]
public struct RenderMesh : ISharedComponentData
{
    public Mesh                 mesh;
    public Material             material;

    public ShadowCastingMode    castShadows;
    public bool                 receiveShadows;
}
```

`ISharedComponentData`在每个entity的内存成本为零（因为他是凌驾于chunk外的）。您可以使用`ISharedComponentData`将具有相同`InstanceRenderer`数据的所有entities分组在一起，然后高效地提取所有矩阵以进行渲染。代码简单有效，因为在ECS访问数据时对数据进行了布局。

有关此示例，请参阅`RenderMeshSystemV2`文件`Packages/com.unity.entities/Unity.Rendering.Hybrid/RenderMeshSystemV2.cs`。

## 有关SharedComponentData的重要说明：

- ECS将具有相同`SharedComponentData`的entities按相同的[chunk](https://docs.unity3d.com/Packages/com.unity.entities@0.13/manual/chunk_iteration.html)分组在一起。它将`SharedComponentData`的索引存储到每个chunk，而不是每个entity一次。结果就是，`SharedComponentData`对于每个entity的内存开销为零。
- 您可以使用[EntityQuery](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.EntityQuery.html)遍历相同类型（components）的所有entities。您还可以使用[EntityQuery.SetFilter（）](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.EntityQuery.html#Unity_Entities_EntityQuery_SetSharedComponentFilter_)专门对具有特定`SharedComponentData`值的entities进行迭代。由于数据布局的原因，这种迭代的开销很低。
- 您可以用`EntityManager.GetAllUniqueSharedComponents`用来检索`SharedComponentData`添加到任何活动实体的所有唯一性。
- ECS会对`SharedComponentData`进行自动[引用计数](https://en.wikipedia.org/wiki/Reference_counting) 。
- `SharedComponentData`应该很少被改变。如果更改一个entity的`SharedComponentData`，则需要使用[memcpy](https://msdn.microsoft.com/en-us/library/aa246468(v=vs.60).aspx)将该entity的所有`ComponentData`内容复制到另一个chunk中（或者新建的一个chunk中）。

更加详细的细节，可以参照：https://gametorrahod.com/everything-about-isharedcomponentdata/

# System State Components

您可以使用[SystemStateComponentData](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.ISystemStateComponentData.html)跟踪一个system内部的资源，并根据需要创建和销毁这些资源，而无需依赖各个回调。

`SystemStateComponentData`和[SystemStateSharedComponentData](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.ISystemStateSharedComponentData.html)与`ComponentData`和`SharedComponentData`相似，但是ECS在销毁实体时不会删除`SystemStateComponentData`。

当entity被销毁时，ECS通常：

1. 查找引用特定entity的ID的所有components。
2. 删除那些components。
3. 回收entity的ID以供重用。

但是，如果`SystemStateComponentData`存在，则ECS不会回收ID。这使system有机会清理与entity的ID相关联的任何资源或状态。ECS仅在`SystemStateComponentData`被移除后才复用实体ID。

## 何时使用System State Components

systems可能需要保持一个基于`ComponentData`的内部状态。例如，可以分配的资源们。

systems还需要能够将状态作为值进行管理，其他systems可能会进行状态的更改。例如，当components中的值更改，或添加或删除相关components时。

`No callbacks`是ECS设计规则的重要组成部分。

`SystemStateComponentData`设计初衷是 对应一个用户组件，从而提供其内部状态。

例如，给定：

- FooComponent（`ComponentData`，用户指定）
- FooStateComponent（`SystemComponentData`，system指定）

### 当一个component被添加时检测

创建一个component时，system state component并不存在。system更新查询components（并没有system state component），并可以推断components已被添加的时刻。此时，system将添加system state component和任何所需的内部状态。

### 当一个component被移除时检测

移除component时，system state component仍然存在。system更新查询system state component（并没有components），并可以推断components已被删除的时刻。此时，system将移除system state component并修正任何需要的内部状态。

### 当一个entity被销毁时检测

`DestroyEntity` 是以下用途的简写程序：

- 查找引用了指定entity ID的components。
- 删除找到的components。
- 回收entity ID。

但是，在`DestroyEntity`移除最后一个component之前，都不会删除`SystemStateComponentData`，并且不会回收entity ID。这使system有机会用移除component完全相同的方式清理内部状态。

## SystemStateComponent

一个 `SystemStateComponentData`与`ComponentData`相似。

```CSharp
struct FooStateComponent : ISystemStateComponentData
{
}
```

一个`SystemStateComponentData`对于创建它的system之外的都是`只读的`。

## SystemStateSharedComponent

一个 `SystemStateSharedComponentData`与`SharedComponentData`相似。

```CSharp
struct FooStateSharedComponent : ISystemStateSharedComponentData
{
  public int Value;
}
```

## 一个使用state components的示例

以下示例展示了一个简单地system，该system说明了如何使用system state components来管理entities。该示例定义了通用IComponentData实例和系统状态ISystemStateComponentData实例。它还基于这些实体定义了三个queries：

- `m_newEntities`会选择具有一般的component但不具有system state component的entities。该query查找system之前未见过的新enitites（因为新加了component嘛，所以就是新entity了）。system会运行一个job会对查询到的entitites添加system state component。
- `m_activeEntities`选择同时具有一般component和system state component的entities。在实际的应用程序中，其内容可能是处理或销毁entities。
- `m_destroyedEntities`选择具有system state component但不具有一般component的entities。由于system state component永远不会单独添加到entity，因此此system或其他system必须删除此query选择的entities。system重用销毁的entities query以运行job并从entities中删除system state component件，这使ECS可以回收entity ID。

**注意：**此简化示例不维护系统内的任何状态。系统状态组件的目的之一是跟踪何时需要分配或清除持久性资源。

```cs
using Unity.Entities;
using Unity.Jobs;
using Unity.Collections;

public struct GeneralPurposeComponentA : IComponentData
{
    public int Lifetime;
}

public struct StateComponentB : ISystemStateComponentData
{
    public int State;
}

public class StatefulSystem : SystemBase
{
    private EntityCommandBufferSystem ecbSource;

    protected override void OnCreate()
    {
        ecbSource = World.GetExistingSystem<EndSimulationEntityCommandBufferSystem>();

        // 创建测试用的entities
        // 这跑在主线程，但因为使用一个了命令缓冲区，所以还是很快
        // This runs on the main thread, but it is still faster to use a command buffer
        EntityCommandBuffer creationBuffer = new EntityCommandBuffer(Allocator.Temp);
        EntityArchetype archetype = EntityManager.CreateArchetype(typeof(GeneralPurposeComponentA));
        for (int i = 0; i < 10000; i++)
        {
            Entity newEntity = creationBuffer.CreateEntity(archetype);
            creationBuffer.SetComponent<GeneralPurposeComponentA>
            (
                newEntity,
                new GeneralPurposeComponentA() { Lifetime = i }
            );
        }
        //执行命令缓冲区内容
        creationBuffer.Playback(EntityManager);
    }

    protected override void OnUpdate()
    {
        EntityCommandBuffer.ParallelWriter parallelWriterECB = ecbSource.CreateCommandBuffer().AsParallelWriter();

        // 有GeneralPurposeComponentA但没有StateComponentB的Entities
        // Entities with GeneralPurposeComponentA but not StateComponentB
        Entities
            .WithNone<StateComponentB>()
            .ForEach(
                (Entity entity, int entityInQueryIndex, in GeneralPurposeComponentA gpA) =>
                {
                // 每个entity添加一个系统状态组件实例
                // Add an ISystemStateComponentData instance
                parallelWriterECB.AddComponent<StateComponentB>
                    (
                        entityInQueryIndex,
                        entity,
                        new StateComponentB() { State = 1 }
                    );
                })
            .ScheduleParallel();
        ecbSource.AddJobHandleForProducer(this.Dependency);

        // 创建一个新的命令缓冲区
        parallelWriterECB = ecbSource.CreateCommandBuffer().AsParallelWriter();

        // 同时拥有GeneralPurposeComponentA和StateComponentB的entities
        // Entities with both GeneralPurposeComponentA and StateComponentB
        Entities
            .WithAll<StateComponentB>()
            .ForEach(
                (Entity entity,
                 int entityInQueryIndex,
                 ref GeneralPurposeComponentA gpA) =>
                {
                // 处理entity，在这个例子里是减少生命时间
                // Process entity, in this case by decrementing the Lifetime count
                gpA.Lifetime--;

                // 如果超过时间限制，就移除entity
                // If out of time, destroy the entity
                if (gpA.Lifetime <= 0)
                    {
                        parallelWriterECB.DestroyEntity(entityInQueryIndex, entity);
                    }
                })
            .ScheduleParallel();
        ecbSource.AddJobHandleForProducer(this.Dependency);

        // 创建一个新的命令缓冲区
        // Create new command buffer
        parallelWriterECB = ecbSource.CreateCommandBuffer().AsParallelWriter();

        // 有StateComponentB但没有GeneralPurposeComponentA的entities
        // Entities with StateComponentB but not GeneralPurposeComponentA
        Entities
            .WithAll<StateComponentB>()
            .WithNone<GeneralPurposeComponentA>()
            .ForEach(
                (Entity entity, int entityInQueryIndex) =>
                {
                // 这个系统有责任去移除他所添加的ISystemStateComponentData实例，否则这些entity永远不会真正的被销毁
                // This system is responsible for removing any ISystemStateComponentData instances it adds
                // Otherwise, the entity is never truly destroyed.
                parallelWriterECB.RemoveComponent<StateComponentB>(entityInQueryIndex, entity);
                })
            .ScheduleParallel();
        ecbSource.AddJobHandleForProducer(this.Dependency);

    }

    protected override void OnDestroy()
    {
        // 实现OnDestroy来清理被system分配的资源
        // Implement OnDestroy to cleanup any resources allocated by this system.
        // (This simplified example does not allocate any resources, so there is nothing to clean up.)
    }
}
```

# Dynamic buffer components

使用dynamic buffer components（动态缓冲区组件）将类似数组的数据与entity相关联。Dynamic buffers是ECS组件，可以容纳可变数量的元素，并根据需要自动调整大小。

要创建Dynamic buffers，请首先声明一个实现[IBufferElementData](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.IBufferElementData.html)的结构，并定义存储在缓冲区中的元素。例如，可以对存储整数的dynamic buffer component使用以下结构：

```cs
public struct IntBufferElement : IBufferElementData
{
    public int Value;
}
```

要将dynamic buffer与entity相关联，请直接向entity添加[IBufferElementData](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.IBufferElementData.html)组件，而不要添加[dynamic buffer container](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.DynamicBuffer-1.html)本身。

ECS管理container。对于大多数用途，您可以使用声明的`IBufferElementData`类型将dynamic buffer与其他任何ECS component相同对待。例如，您可以`IBufferElementData`在[entity queries](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.EntityQuery.html)中以及在添加或删除dynamic buffer component时使用该类型。但是，必须使用不同的函数来访问dynamic buffer component，并且这些函数提供了[DynamicBuffer](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.DynamicBuffer-1.html)实例，该实例为缓冲区数据提供了类似于数组的接口。

要为dynamic buffer component指定“internal capacity（内部容量）”，请使用[InternalBufferCapacity Attribute](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.InternalBufferCapacityAttribute.html)。内部容量定义了dynamic buffer与entity的其他component一起存储在[ArchetypeChunk](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.ArchetypeChunk.html)中的元素数。除了内部容量之外，这个缓冲区还会在当chunk块之外分配一个堆内存块，并将所有现有元素移动到其中，ECS自动管理该外部缓冲区内存（Heap memory block），并在移除dynamic buffer component时释放他的内存。

**注意：**如果缓冲区中的数据不是动态的，则可以使用[Blob资产](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.BlobBuilder.html)代替动态缓冲区。Blob资产可以存储结构化数据，包括数组。多个entities可以共享Blob资产。

## 声明缓冲区元素类型

要声明缓冲区，请声明一个结构，该结构定义了要放入缓冲区的元素的类型。该结构必须实现[IBufferElementData](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.IBufferElementData.html)，如下所示：

```cs
// 内部容量表示一个缓冲区在其本身被移出chunk之前可以容纳多少个元素（其实就是原始最大容量）
// InternalBufferCapacity specifies how many elements a buffer can have before
// the buffer storage is moved outside the chunk.
[InternalBufferCapacity(8)]
public struct MyBufferElement : IBufferElementData
{
    // Actual value each buffer element will store.
    // 每个缓冲区内的元素
    public int Value;

    // The following implicit conversions are optional, but can be convenient.
    // 以下隐式实现可选，但很方便
    public static implicit operator int(MyBufferElement e)
    {
        return e.Value;
    }

    public static implicit operator MyBufferElement(int e)
    {
        return new MyBufferElement { Value = e };
    }
}
```

## 向entity添加缓冲区类型

要将缓冲区添加到实体，请添加`IBufferElementData`定义缓冲区元素数据类型的结构，然后将该类型直接添加到entity或[archetype](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.EntityArchetype.html)：

### 使用EntityManager.AddBuffer()

有关更多信息，请参见[EntityManager.AddBuffer()](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.EntityManager.html#Unity_Entities_EntityManager_AddBuffer_)上的文档。

```cs
EntityManager.AddBuffer<MyBufferElement>(entity);
```

### 使用archetype

```cs
Entity e = EntityManager.CreateEntity(typeof(MyBufferElement));
```

### 使用`[GenerateAuthoringComponent]`attribute

您可以用`[GenerateAuthoringComponent]`标识只包含一个字段的简单得IBufferElementData。设置此属性后，您可以将ECS IBufferElementData组件添加到GameObject，以便可以在编辑器中设置缓冲区元素。

例如，如果声明以下类型，则可以将其直接添加到编辑器中的GameObject中：

```CSharp
[GenerateAuthoringComponent]
public struct IntBufferElement: IBufferElementData
{
    public int Value;
}
```

Unity在幕后生成了一个名为`IntBufferElementAuthoring`（继承自`MonoBehaviour`）的类，该类公开了一个公共`List<int>`类型的字段。将包含此GenerateAuthoringComponent的GameObject转换为entity时，该列表将转换为`DynamicBuffer<IntBufferElement>`，然后添加到转换后的entity中。

请注意以下限制：

- 单个C＃文件中只能有一个generated authoring component，并且C＃文件中不能包含另一个MonoBehaviour。
- `IBufferElementData` 对于包含多个字段的类型，无法自动生成GenerateAuthoringComponent。
- `IBufferElementData` 无法为具有显式布局的类型自动生成GenerateAuthoringComponent。

### 使用[EntityCommandBuffer](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.EntityCommandBuffer.html)

将命令添加到entity command buffer时，可以添加或设置buffer component。

使用[AddBuffer](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.EntityCommandBuffer.html#Unity_Entities_EntityCommandBuffer_AddBuffer__1_Unity_Entities_Entity_)为entity创建一个新的缓冲区，这将更改entity的archetype。使用[SetBuffer](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.EntityCommandBuffer.html#Unity_Entities_EntityCommandBuffer_SetBuffer__1_Unity_Entities_Entity_)清除现有缓冲区（必须是已存在的）并在其位置创建一个新的空缓冲区。这两个函数都返回一个[DynamicBuffer](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.DynamicBuffer-1.html)实例，您可以使用该实例来填充新缓冲区。您可以立即将元素添加到缓冲区，但是在执行命令缓冲区时需要将缓冲区添加到entity，否则无法访问它们。

以下job使用command buffer创建一个新entity，然后使用[EntityCommandBuffer.AddBuffer](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.EntityCommandBuffer.html#Unity_Entities_EntityCommandBuffer_AddBuffer__1_Unity_Entities_Entity_)添加一个动态缓冲区组件。job还向动态缓冲区添加了许多元素。

```cs
using Unity.Entities;
using Unity.Jobs;

public class CreateEntitiesWithBuffers : SystemBase
{
    // A command buffer system executes command buffers in its own OnUpdate
    // 一个命令缓冲区system执行命令缓冲区在他自己的OnUpdate函数
    public EntityCommandBufferSystem CommandBufferSystem;

    protected override void OnCreate()
    {
        // Get the command buffer system
        // 获取命令缓冲区system
        CommandBufferSystem
            = World.DefaultGameObjectInjectionWorld.GetExistingSystem<EndSimulationEntityCommandBufferSystem>();
    }

    protected override void OnUpdate()
    {
        // The command buffer to record commands,
        // 命令缓冲区，在一帧的晚些时候被command buffer system执行
        // which are executed by the command buffer system later in the frame
        EntityCommandBuffer.ParallelWriter commandBuffer
            = CommandBufferSystem.CreateCommandBuffer().AsParallelWriter();
        //The DataToSpawn component tells us how many entities with buffers to create
        //DataToSpawn component告诉我们在缓冲区有多少entities将会被创建
        Entities.ForEach((Entity spawnEntity, int entityInQueryIndex, in DataToSpawn data) =>
        {
            for (int e = 0; e < data.EntityCount; e++)
            {
                //Create a new entity for the command buffer
                //为命令缓冲区创建一个Entity
                Entity newEntity = commandBuffer.CreateEntity(entityInQueryIndex);

                //Create the dynamic buffer and add it to the new entity
                //创建动态缓冲区，并把它加入刚刚创建的entity身上
                DynamicBuffer<MyBufferElement> buffer =
                    commandBuffer.AddBuffer<MyBufferElement>(entityInQueryIndex, newEntity);

                //Reinterpret to plain int buffer
                //将动态缓冲区重定义为int元素的缓冲区
                DynamicBuffer<int> intBuffer = buffer.Reinterpret<int>();

                //Optionally, populate the dynamic buffer
                //可选的，填充动态缓冲区
                for (int j = 0; j < data.ElementCount; j++)
                {
                    intBuffer.Add(j);
                }
            }

            //Destroy the DataToSpawn entity since it has done its job
            //在DataToSpawn entity完成他的job时，销毁他 
            commandBuffer.DestroyEntity(entityInQueryIndex, spawnEntity);
        }).ScheduleParallel();

        CommandBufferSystem.AddJobHandleForProducer(this.Dependency);
    }
}
```

**注意：**不需要立即将数据添加到动态缓冲区。但是，直到执行了您正在使用的entity命令缓冲区后，您才能再次访问该缓冲区。

## 访问缓冲区

您可以使用[EntityManager](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.EntityManager.html)，[systems](https://docs.unity3d.com/Packages/com.unity.entities@0.13/manual/ecs_systems.html)和[job](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.DynamicBuffer-1.html)来访问[DynamicBuffer](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.DynamicBuffer-1.html)实例，就像访问实体的其他组件类型一样。

### EntityManager访问缓冲区

您可以使用[EntityManager的实例](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.EntityManager.html)来访问动态缓冲区：

```cs
DynamicBuffer<MyBufferElement> dynamicBuffer
    = EntityManager.GetBuffer<MyBufferElement>(entity);
```

### 查找另一个entity的缓冲区

当您需要查找job中另一个entity的缓冲区数据时，可以将[BufferFromEntity](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.BufferFromEntity-1.html)变量传递给作业。

```cs
BufferFromEntity<MyBufferElement> lookup = GetBufferFromEntity<MyBufferElement>();
var buffer = lookup[entity];
buffer.Add(17);
buffer.RemoveAt(0);
```

### SystemBase Entities.ForEach访问缓冲区

通过将缓冲区作为lambda函数参数之一传递，可以访问与使用Entities.ForEach处理的entity相关联的动态缓冲区。以下示例将所有存储在类型为`MyBufferElement`缓冲区中的值相加：

```cs
public class DynamicBufferSystem : SystemBase
{
    protected override void OnUpdate()
    {
        var sum = 0;

        Entities.ForEach((DynamicBuffer<MyBufferElement> buffer) =>
        {
            for(int i = 0; i < buffer.Length; i++)
            {
                sum += buffer[i].Value;
            }
        }).Run();

        Debug.Log("Sum of all buffers: " + sum);
    }
}
```

请注意，在此示例中我们可以直接对`sum`写入捕获的变量，因为我们使用`Run()`来执行代码。如果我们将函数安排为在job中运行，那么即使结果为单个值，我们也只能写入本地化的容器（例如NativeArray）。

### IJobChunk访问缓冲区

要访问`IJobChunk`job中的单个缓冲区，请将缓冲区数据类型传递给job，然后使用该数据类型获取[BufferAccessor](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.BufferAccessor-1.html)。缓冲区访问器是一种类似于数组的结构，可提供对当前块中所有动态缓冲区的访问。

与前面的示例类似，以下示例将所有包含类型为`MyBufferElement`的元素的动态缓冲区的内容相加。`IJobChunk`job还可以在每个chunk上并行运行，因此在示例中，它首先将每个缓冲区的中间和存储在本地化数组中，然后使用第二个job来计算最终和。在这种情况下，中间数组为每个chunk保存一个结果，而不是为每个entity保存一个结果。

```cs
public class DynamicBufferJobSystem : SystemBase
{
    private EntityQuery query;

    protected override void OnCreate()
    {
        //Create a query to find all entities with a dynamic buffer
        // containing MyBufferElement
        // 创建一个用于查找所有带有一个动态缓冲区组件（包含 MyBufferElement）的entities的query
        EntityQueryDesc queryDescription = new EntityQueryDesc();
        queryDescription.All = new[] {ComponentType.ReadOnly<MyBufferElement>()};
        query = GetEntityQuery(queryDescription);
    }

    public struct BuffersInChunks : IJobChunk
    {
        //The data type and safety object
        //数据类型和安全对象
        public BufferTypeHandle<MyBufferElement> BufferTypeHandle;

        //An array to hold the output, intermediate sums
        //用于保存输出中间结果的数组
        public NativeArray<int> sums;

        public void Execute(ArchetypeChunk chunk,
            int chunkIndex,
            int firstEntityIndex)
        {
            //A buffer accessor is a list of all the buffers in the chunk
            //一个缓冲区的获取者是在一个chunk中的所有缓冲区的列表
            BufferAccessor<MyBufferElement> buffers
                = chunk.GetBufferAccessor(BufferTypeHandle);

            for (int c = 0; c < chunk.Count; c++)
            {
                //An individual dynamic buffer for a specific entity
                //指定entity的单个动态缓冲区
                DynamicBuffer<MyBufferElement> buffer = buffers[c];
                for(int i = 0; i < buffer.Length; i++)
                {
                    sums[chunkIndex] += buffer[i].Value;
                }
            }
        }
    }

    //Sums the intermediate results into the final total
    //将所有中间结果相加得到最后的总值
    public struct SumResult : IJob
    {
        [DeallocateOnJobCompletion] public NativeArray<int> sums;
        public NativeArray<int> result;
        public void Execute()
        {
            for(int i  = 0; i < sums.Length; i++)
            {
                result[0] += sums[i];
            }
        }
    }

    protected override void OnUpdate()
    {
        //Create a native array to hold the intermediate sums
        //创建一个本地化数组来保存中间值
        int chunksInQuery = query.CalculateChunkCount();
        NativeArray<int> intermediateSums
            = new NativeArray<int>(chunksInQuery, Allocator.TempJob);

        //Schedule the first job to add all the buffer elements
        //安排第一个job将所有缓冲区元素加起来
        BuffersInChunks bufferJob = new BuffersInChunks();
        bufferJob.BufferTypeHandle = GetBufferTypeHandle<MyBufferElement>();
        bufferJob.sums = intermediateSums;
        this.Dependency = bufferJob.ScheduleParallel(query, this.Dependency);

        //Schedule the second job, which depends on the first
        //安排第二个job，他依赖第一个job，用来得到最终结果
        SumResult finalSumJob = new SumResult();
        finalSumJob.sums = intermediateSums;
        NativeArray<int> finalSum = new NativeArray<int>(1, Allocator.Temp);
        finalSumJob.result = finalSum;
        this.Dependency = finalSumJob.Schedule(this.Dependency);

        this.CompleteDependency();
        Debug.Log("Sum of all buffers: " + finalSum[0]);
        finalSum.Dispose();
    }
}
```

## 重新解释缓冲区

缓冲区可以重新解释为相同大小的类型。目的是允许进行受控的类型合并，并在转换元素类型时不感到蛋疼。要重新解释，请调用[Reinterpret ](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.DynamicBuffer-1.html#Unity_Entities_DynamicBuffer_1_Reinterpret_)：

```cs
DynamicBuffer<int> intBuffer
    = EntityManager.GetBuffer<MyBufferElement>(entity).Reinterpret<int>();
```

重新解释的缓冲区实例保留了原始缓冲区的安全性，并且可以安全使用。*重新解释的缓冲区引用原始数据*，因此对一个重新解释的缓冲区的修改会立即反映在其他缓冲区中。

**注意：**重新解释函数仅强制所涉及的类型具有相同的长度。例如，您可以为一个`uint`和`float`buffer 加上别名而不引起错误，因为这两种类型均为32位长。您必须确保重新解释在逻辑上有意义（别乱搞）。

## 缓冲区引用无效

每次[结构更改都会](https://docs.unity3d.com/Packages/com.unity.entities@0.13/manual/sync_points.html#structural-changes)使对动态缓冲区的所有引用无效。结构变化通常会导致实体从一个chunk移动到另一个chunk。小型动态缓冲区可以引用块内的内存（而不是主内存中的内存），因此，在结构更改后需要重新获取它们。

```cs
var entity1 = EntityManager.CreateEntity();
var entity2 = EntityManager.CreateEntity();

DynamicBuffer<MyBufferElement> buffer1
    = EntityManager.AddBuffer<MyBufferElement>(entity1);
// This line causes a structural change and invalidates
// the previously acquired dynamic buffer
// 这一行会导致一次结构癌变，并且无效化先前的缓冲区
DynamicBuffer<MyBufferElement> buffer2
    = EntityManager.AddBuffer<MyBufferElement>(entity1);
// This line will cause an error:
// 这一行将会导致错误
buffer1.Add(17);
```

# Chunk component data

使用chunk components将数据与特定[chunk](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.ArchetypeChunk.html)关联。

chunk component包含适用于特定chunk中所有entities的数据。例如，如果您有代表紧密排布的3D对象的entities chunks，则可以使用chunk component为它们存储集合边界框。chunk component使用接口类型[IComponentData](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.IComponentData.html)。

## 添加并设置chunk component的值

尽管chunk component可以单个块具有唯一的值，但它们仍然是该chunk中entity archetype的一部分。因此，如果您从实体中删除了一个chunk component，ECS会将该entity移动到另一个chunk（可能是一个新的chunk）。同样，如果将chunk component添加到entity，则ECS会将该entity移至其他chunk，因为其archetype会更改；chunk component的添加不会影响原始chunk中的其余entities。

如果您在chunk中使用entity来更改chunk component的值，则它将更改该chunk中所有entities所共有的chunk component的值*(这一点和SCB不一致)*。如果更改entity的archetype，以使其移动到具有相同类型的chunk component的新chunk中，那么目标chunk中的现有值将不受影响*(这一点和SCB一致)*。**注意：**如果将entity移至新创建的chunk，则ECS会为该chunk创建一个新的chunk component并分配其默认值。

使用chunk component和通用component之间的主要区别在于，您使用不同的功能来添加，设置和删除它们。

**相关API**

| **目的**                                                     | **功能**                                                     |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| 介绍                                                         | [IComponentData](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.IComponentData.html) |
|                                                              |                                                              |
| **[ArchetypeChunk方法](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.ArchetypeChunk.html)** |                                                              |
| 读                                                           | [GetChunkComponentData （ArchetypeChunkComponentType ）](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.ArchetypeChunk.html#Unity_Entities_ArchetypeChunk_GetChunkComponentData_) |
| 检查                                                         | [HasChunkComponent <T>（ArchetypeChunkComponentType <T>）]   |
| 写                                                           | [SetChunkComponentData （ArchetypeChunkComponentType ，T）](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.ArchetypeChunk.html#Unity_Entities_ArchetypeChunk_SetChunkComponentData_) |
|                                                              |                                                              |
| **[EntityManager方法](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.EntityManager.html)** |                                                              |
| 创建                                                         | [AddChunkComponentData （Entity）](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.EntityManager.html#Unity_Entities_EntityManager_AddChunkComponentData__1_Unity_Entities_Entity_) |
| 创建                                                         | [AddChunkComponentData （EntityQuery，T）](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.EntityManager.html#Unity_Entities_EntityManager_AddChunkComponentData__1_Unity_Entities_EntityQuery___0_) |
| 创建                                                         | [AddComponents（Entity，ComponentTypes）](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.EntityManager.html#Unity_Entities_EntityManager_AddComponents_Unity_Entities_Entity_Unity_Entities_ComponentTypes_) |
| 获取类型信息                                                 | [GetComponentTypeHandle]                                     |
| 读                                                           | [GetChunkComponentData <T>（ArchetypeChunk）]                |
| 读                                                           | [GetChunkComponentData （Entity）](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.EntityManager.html#Unity_Entities_EntityManager_GetChunkComponentData__1_Unity_Entities_Entity_) |
| 检查                                                         | [HasChunkComponent （Entity）](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.EntityManager.html#Unity_Entities_EntityManager_HasChunkComponent_) |
| 删除                                                         | [RemoveChunkComponent （Entity）](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.EntityManager.html#Unity_Entities_EntityManager_RemoveChunkComponent__1_Unity_Entities_Entity_) |
| 删除                                                         | [RemoveChunkComponentData （EntityQuery）](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.EntityManager.html#Unity_Entities_EntityManager_RemoveChunkComponentData_) |
| 写                                                           | [EntityManager.SetChunkComponentData （ArchetypeChunk，T）](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.EntityManager.html#Unity_Entities_EntityManager_SetChunkComponentData_) |



## 声明chunk component

chunk component使用接口类型[IComponentData](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.IComponentData.html)。

```cs
public struct ChunkComponentA : IComponentData
{
    public float Value;
}
```



## 创建一个chunk component

要直接添加chunk component，请确保目标chunk中至少存在一个实体，或使用选择一组目标chunks的entity query。您不能在job内添加chunk component，也不能使用`EntityCommandBuffer`来添加chunk component。

您还可以将chunk component作为[EntityArchetype](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.EntityArchetype.html)的一部分或ECS用于创建entity的[ComponentType]对象列表的一部分。*ECS为每个chunk创建chunk component，并存储具有该archetype的实体*。

通过这些方法使用[ComponentType.ChunkComponent ](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.ComponentType.html#Unity_Entities_ComponentType_ChunkComponent_)或[ComponentType.ChunkComponentReadOnly <T>]。否则，ECS将该组件视为通用组件，而不是chunk component。

**使用在一个chunk里的entity**

给定目标chunk中的entity，您可以使用[EntityManager.AddChunkComponentData （）](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.EntityManager.html#Unity_Entities_EntityManager_AddChunkComponentData__1_Unity_Entities_EntityQuery___0_)函数将chunk component添加到块中：

```cs
EntityManager.AddChunkComponentData<ChunkComponentA>(entity);
```

使用此方法时，不能立即为chunk component设置值。

**使用[EntityQuery](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.EntityQuery.html)**

给定一个entity query，该query选择了要添加chunk component的所有chunks，您可以使用[EntityManager.AddChunkComponentData （）](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.EntityManager.html#Unity_Entities_EntityManager_AddChunkComponentData__1_Unity_Entities_EntityQuery___0_)函数来添加和设置component：

```cs
EntityQueryDesc ChunksWithoutComponentADesc = new EntityQueryDesc()
{
    None = new ComponentType[] { ComponentType.ChunkComponent<ChunkComponentA>() }
};
EntityQuery ChunksWithoutChunkComponentA = GetEntityQuery(ChunksWithoutComponentADesc);

EntityManager.AddChunkComponentData<ChunkComponentA>(ChunksWithoutChunkComponentA,
    new ChunkComponentA() { Value = 4 });
```

使用此方法时，可以为所有新chunk component设置相同的初始值。

**使用[EntityArchetype](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.EntityArchetype.html)**

当您创建具有archetype或具有多个components类型的entity时，请在archetype中包含chunk component类型：

```cs
EntityArchetype ArchetypeWithChunkComponent = EntityManager.CreateArchetype(
    ComponentType.ChunkComponent(typeof(ChunkComponentA)),
    ComponentType.ReadWrite<GeneralPurposeComponentA>());
Entity newEntity = EntityManager.CreateEntity(ArchetypeWithChunkComponent);
```

或具有多个components：

```cs
ComponentType[] compTypes = {ComponentType.ChunkComponent<ChunkComponentA>(),
                             ComponentType.ReadOnly<GeneralPurposeComponentA>()};
Entity entity = EntityManager.CreateEntity(compTypes);
```

使用这些方法时，ECS新建的chunks的chunk components作为entity构造的一部分将接收默认结构值。ECS不会更改现有chunk中的chunk component。请参阅[更新块组件，](https://docs.unity3d.com/Packages/com.unity.entities@0.13/manual/ecs_chunk_component.html#update)以了解如何在给定entity引用的情况下设置chunk component值。



## 读取chunk component

要读取chunk component，可以使用代表chunk的[ArchetypeChunk](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.ArchetypeChunk.html)对象，或在目标chunk中使用entity。

**使用ArchetypeChunk实例**

给定一个chunk，您可以使用[EntityManager.GetChunkComponentData ](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.EntityManager.html#Unity_Entities_EntityManager_GetChunkComponentData__1_Unity_Entities_ArchetypeChunk_)函数读取其chunk component。以下代码遍历与查询匹配的所有chunks，并访问他们的ChunkComponentA`：

```cs
NativeArray<ArchetypeChunk> chunks = ChunksWithChunkComponentA.CreateArchetypeChunkArray(Allocator.TempJob);
foreach (var chunk in chunks)
{
    var compValue = EntityManager.GetChunkComponentData<ChunkComponentA>(chunk);
    //..
}
chunks.Dispose();
```

**使用在一个chunk里的entity**

给定一个entity，您可以使用[EntityManager.GetChunkComponentData ](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.EntityManager.html#Unity_Entities_EntityManager_GetChunkComponentData__1_Unity_Entities_ArchetypeChunk_)访问包含该entity的chunk中的chunk component：

```cs
if (EntityManager.HasChunkComponent<ChunkComponentA>(entity))
{
    ChunkComponentA chunkComponentValue = EntityManager.GetChunkComponentData<ChunkComponentA>(entity);
}
```



## 更新chunk component

您可以在引用其所属的[chunk](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.ArchetypeChunk.html)情况下更新chunk component。在`IJobChunk`job中，可以调用[ArchetypeChunk.SetChunkComponentData]。在主线程上，可以使用EntityManager版本：[EntityManager.SetChunkComponentData]。**注意：**您无法使用SystemBase Entities.ForEach访问chunk component，因为您无权访问`ArchetypeChunk`对象或EntityManager。

**使用ArchetypeChunk实例**

要在job中更新chunk component，请参阅 [Reading and writing in a system](https://docs.unity3d.com/Packages/com.unity.entities@0.13/manual/ecs_chunk_component.html#update).。

要在主线程上更新块组件，请使用EntityManager：

```cs
EntityManager.SetChunkComponentData<ChunkComponentA>(chunk, new ChunkComponentA() { Value = 7 });
```

**使用entity实例**

如果chunk中除本身外有一个entity，则还可以使用EntityManger来获取包含该entity的chunk：

**注意：**如果只想读取chunk component而不写入，则在定义实体查询时应使用[ComponentType.ChunkComponentReadOnly](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.ComponentType.html#Unity_Entities_ComponentType_ChunkComponentReadOnly_)，以避免创建不必要的job scheduling 约束。



## 删除chunk component

使用[EntityManager.RemoveChunkComponent](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.EntityManager.html#Unity_Entities_EntityManager_RemoveChunkComponent_)函数删除chunk component。您可以删除目标chunk中给定entity的chunk component，也可以从entity query选择的所有chunk中删除给定类型的所有chunk component。

如果从单个entity中删除chunk component，则该entity将移至其他chunk，因为该实体的archetype会更改。只要该chunk中还有其他entities，该chunk就会保留未更改的chunk component。



## 在query中使用chunk component

要在entity query中使用chunk component，必须使用[ComponentType.ChunkComponent ](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.ComponentType.html#Unity_Entities_ComponentType_ChunkComponent_)或[ComponentType.ChunkComponentReadOnly <T>]函数来指定类型。否则，ECS将该组件视为通用component，而不是Chunk component。

**使用[EntityQueryDesc](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.EntityQueryDesc.html)**

您可以使用以下query描述创建一个entity query，该query选择所有chunks以及这些chunks中具有类型为*ChunkComponentA*的chunk component的*entity*：

```cs
EntityQueryDesc ChunksWithChunkComponentADesc = new EntityQueryDesc()
{
    All = new ComponentType[] { ComponentType.ChunkComponent<ChunkComponentA>() }
};
```



## 遍历chunk以设置chunk component

要遍历要为其设置chunk components的所有chunks，可以创建一个entity query，该entity query选择正确的chunk，然后使用EntityQuery对象获取ArchetypeChunk实例的列表作为本地化数组。ArchetypeChunk对象允许您将新值写入组chunk component。

```cs
public class ChunkComponentExamples : SystemBase
{
    private EntityQuery ChunksWithChunkComponentA;
    protected override void OnCreate()
    {
        EntityQueryDesc ChunksWithComponentADesc = new EntityQueryDesc()
        {
            All = new ComponentType[] { ComponentType.ChunkComponent<ChunkComponentA>() }
        };
        ChunksWithChunkComponentA = GetEntityQuery(ChunksWithComponentADesc);
    }

    [BurstCompile]
    struct ChunkComponentCheckerJob : IJobChunk
    {
        public ComponentTypeHandle<ChunkComponentA> ChunkComponentATypeHandle;
        public void Execute(ArchetypeChunk chunk, int chunkIndex, int firstEntityIndex)
        {
            var compValue = chunk.GetChunkComponentData(ChunkComponentATypeHandle);
            //...
            var squared = compValue.Value * compValue.Value;
            chunk.SetChunkComponentData(ChunkComponentATypeHandle,
                new ChunkComponentA() { Value = squared });
        }
    }

    protected override void OnUpdate()
    {
        var job = new ChunkComponentCheckerJob()
        {
            ChunkComponentATypeHandle = GetComponentTypeHandle<ChunkComponentA>()
        };
        this.Dependency = job.Schedule(ChunksWithChunkComponentA, this.Dependency);
    }
}
```

请注意，如果需要读取chunk中的component以确定chunk component的正确值，则应使用[IJobEntityBatch](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.IJobEntityBatch.html)。例如，以下代码为包含具有LocalToWorld组件的实体的所有块计算出与轴对齐的边界框：

```cs
public struct ChunkAABB : IComponentData
{
    public AABB Value;
}

[UpdateInGroup(typeof(PresentationSystemGroup))]
[UpdateBefore(typeof(UpdateAABBSystem))]
public class AddAABBSystem : SystemBase
{
    EntityQuery queryWithoutChunkComponent;
    protected override void OnCreate()
    {
        queryWithoutChunkComponent = GetEntityQuery(new EntityQueryDesc()
        {
            All = new ComponentType[]  { ComponentType.ReadOnly<LocalToWorld>() },
            None = new ComponentType[] { ComponentType.ChunkComponent<ChunkAABB>() }
        });
    }

    protected override void OnUpdate()
    {
        // This is a structural change and a sync point
        // 这是一个结构改变和一个同步点
        EntityManager.AddChunkComponentData<ChunkAABB>(queryWithoutChunkComponent, new ChunkAABB());
    }
}

[UpdateInGroup(typeof(PresentationSystemGroup))]
public class UpdateAABBSystem : SystemBase
{
    EntityQuery queryWithChunkComponent;
    protected override void OnCreate()
    {
        queryWithChunkComponent = GetEntityQuery(new EntityQueryDesc()
        {
            All = new ComponentType[] { ComponentType.ReadOnly<LocalToWorld>(),
                                        ComponentType.ChunkComponent<ChunkAABB>()}
        });
    }

    [BurstCompile]
    struct AABBJob : IJobChunk
    {
        [ReadOnly] public ComponentTypeHandle<LocalToWorld> LocalToWorldTypeHandleInfo;
        public ComponentTypeHandle<ChunkAABB> ChunkAabbTypeHandleInfo;
        public uint L2WChangeVersion;
        public void Execute(ArchetypeChunk chunk, int chunkIndex, int firstEntityIndex)
        {
            bool chunkHasChanges = chunk.DidChange(LocalToWorldTypeHandleInfo, L2WChangeVersion);

            if (!chunkHasChanges)
                return; // early out if the chunk transforms haven't changed

            NativeArray<LocalToWorld> transforms = chunk.GetNativeArray<LocalToWorld>(LocalToWorldTypeHandleInfo);
            UnityEngine.Bounds bounds = new UnityEngine.Bounds();
            bounds.center = transforms[0].Position;
            for (int i = 1; i < transforms.Length; i++)
            {
                bounds.Encapsulate(transforms[i].Position);
            }
            chunk.SetChunkComponentData(ChunkAabbTypeHandleInfo, new ChunkAABB() { Value = bounds.ToAABB() });
        }
    }

    protected override void OnUpdate()
    {
        var job = new AABBJob()
        {
            LocalToWorldTypeHandleInfo = GetComponentTypeHandle<LocalToWorld>(true),
            ChunkAabbTypeHandleInfo = GetComponentTypeHandle<ChunkAABB>(false),
            L2WChangeVersion = this.LastSystemVersion
        };
        this.Dependency = job.Schedule(queryWithChunkComponent, this.Dependency);
    }
}
```
