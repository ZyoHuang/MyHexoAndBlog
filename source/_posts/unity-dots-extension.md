---
title: Unity DOTS：ECS拓展内容
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

# Sync points

synchronization poin（sync point）是程序执行中的一个点，它等待到目前为止已调度的所有job的完成。同步点会限制您一段时间内使用Job System中所有可用工作线程的能力。因此，通常应避免同步点。

## Structural changes(结构变化)

同步点是由 您在任何其他job正在操作components时不能安全执行自己的操作所引起的。ECS中数据的结构更改是引发Sync points的主要原因。以下所有都是结构上的变化：

- 创建entities
- 删除entities
- 向entity添加component
- 从entity中删除component
- 更改sharedcomponent的value

广义上讲，任何更改entity archetype或导致chunk中entities顺序更改的操作都是结构性更改。这些结构更改只能在主线程上执行。

结构更改不仅需要Sync points，而且还会使对任何component数据的所有直接引用无效。这包括[DynamicBuffer的](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.DynamicBuffer-1.html)实例以及提供对component（例如[ComponentSystemBase.GetComponentDataFromEntity）的](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.ComponentSystemBase.html#Unity_Entities_ComponentSystemBase_GetComponentDataFromEntity_)直接访问的方法的结果。

## 避免Sync points

您可以使用[entity command buffers](https://docs.unity3d.com/Packages/com.unity.entities@0.13/manual/entity_command_buffer.html)（ECB）将结构更改排入队列，而不是立即执行它们。存储在ECB中的命令可以在该帧的稍后一点执行。当执行ECB时，这会将跨帧分布的多个Sync points减少到单个Sync point。

每个标准[ComponentSystemGroup](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.ComponentSystemGroup.html)实例都提供一个[EntityCommandBufferSystem](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.EntityCommandBuffer.html)作为group中更新的第一个和最后一个system。通过从这些标准ECB系统之一获取[ECB](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.EntityCommandBuffer.html)对象，组内的所有结构更改都发生在帧中的同一点，从而只会有一个Sync point而不是多个Sync points。ECB还允许您记录job中的结构更改。如果没有ECB，则只能在主线程上进行结构更改。（即使在主线程上，通常也比使用[EntityManager](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.EntityManager.html)类本身对结构进行一个接一个的更改更快的在ECB中记录命令然后回放这些命令。）

如果您不能将[EntityCommandBufferSystem](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.EntityCommandBuffer.html)用于任务，请尝试将按system执行顺序进行结构更改的所有systems组合在一起。如果两个system都进行结构更改，则它们只有顺序更新才能产生一个同步点。

有关使用EntityCommandBuffer和EntityCommandBufferSystem的更多信息，请参见[Entity Command Buffers](https://docs.unity3d.com/Packages/com.unity.entities@0.13/manual/entity_command_buffer.html)。

# Write groups

常见的ECS模式是System读取一组**输入**components并写入另一components作为其**输出**。但是，在某些情况下，您可能要覆盖system的输出，并基于不同的输入集使用不同的system来更新输出components。Write groups为一个system提供了一种覆盖另一个system的机制，还不用更改另一个system。

目标component类型的write group由ECS应用[`WriteGroup`attribute](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.WriteGroupAttribute.html)的所有其他components类型组成，并以该目标component类型作为参数。作为system创建者，您可以使用write group，以便system用户可以排除system原本会选择和处理的entities。通过这种过滤机制，system用户可以根据自己的逻辑为排除的entities更新component，同时让system在其余components上正常运行。

要使用write groups，必须在system中的query上使用[write group filter option](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.EntityQueryOptions.html)。这会从query中排除所有具有某个component的write group中所有components的entities，这些components在query中被标记为可写。

要覆盖使用write group的system，请将您自己的component类型标记为该system的输出component类型的write group的一部分。原始system将忽略具有该component的任何entity，您可以使用自己的system更新这些entities的数据。

## Write groups示例

在此示例中，您使用外部程序包，他的功能是根据游戏中所有角色的健康状况为其进行着色。包中有两个components：`HealthComponent`和`ColorComponent`。

```csharp
public struct HealthComponent : IComponentData
{
   public int Value;
}

public struct ColorComponent : IComponentData
{
   public float4 Value;
}
```

此外，软件包中有两个systems：

1. `ComputeColorFromHealthSystem`，其读取`HealthComponent`写入`ColorComponent`
2. `RenderWithColorComponent`，读`ColorComponent`

为了表示玩家何时使用充能道具并且他们的角色变得无敌，您可以在角色的entity上附加一个`InvincibleTagComponent`。在这种情况下，角色的颜色应更改为单独的不同颜色，但是这个功能和上面的包中的功能不兼容。

您可以创建自己的system来重写该`ColorComponent`值，但理想`ComputeColorFromHealthSystem`情况下不会为您的entity计算这个充能之后的无敌颜色，因为它会忽略具有`InvincibleTagComponent`的任何entity。当屏幕上有成千上万的玩家时，这变得更加关键。这一切的缘由就是这个外来的功能包不知道`InvincibleTagComponent`的存在导致的。这是Write groups有用的时候。当您知道计算的值将被覆盖时，它使system可以忽略query中的entities。您需要做两件事：

1. `InvincibleTagComponent`必须标记为`ColorComponent`的write group的一部分：

   ```csharp
   [WriteGroup(typeof(ColorComponent))]
   struct InvincibleTagComponent : IComponentData {}
   ```

   `ColorComponent`的write group由所有具有`WriteGroup`attribute（参数为`typeof(ColorComponent)`）的component类型组成。

2. `ComputeColorFromHealthSystem`必须显式支持write groups。为此，system需要`EntityQueryOptions.FilterWriteGroup`为其所有queries指定options。

您可以这样实现`ComputeColorFromHealthSystem`：

```csharp
...
protected override void OnUpdate() {
   Entities
      .WithName("ComputeColor")
      .WithEntityQueryOptions(EntityQueryOptions.FilterWriteGroup) // support write groups
      .ForEach((ref ColorComponent color, in HealthComponent health) => {
         // compute color here
      }).ScheduleParallel();
}
...
```

执行此job时，将发生以下情况：

1. system检测到您写入`ColorComponent`的内容，因为它是一个按引用传递的参数
2. 它查找`ColorComponent`的write group并在其中找到`InvincibleTagComponent`
3. 它不会包含所有具有`InvincibleTagComponent`的entities

好处是，这允许system基于其未知的类型排除entities。

**注意：**有关更多示例，请参见`Unity.Transforms`代码，该代码对其更新的每个component（包括）使用write group`LocalToWorld`。

## 创建Write groups

要创建write group，请将`WriteGroup`attribute添加到write group中每个component类型的声明中。该`WriteGroup`attribute有一个参数，这是group中components用来更新的component类型。单个component可以是多个write group的成员。

例如，如果您有一个只要有component`A`或`B`在entity上就写入component`W`的system，则`W`可以按以下方式定义write group：

```csharp
public struct W : IComponentData
{
   public int Value;
}

[WriteGroup(typeof(W))]
public struct A : IComponentData
{
   public int Value;
}

[WriteGroup(typeof(W))]
public struct B : IComponentData
{
   public int Value;
}
```

**注意：**请勿将write group（上面示例中的component`W`）的目标添加到其本身的write group中。

## 启用Write groups过滤

要启用write group过滤，请在job上设置`FilterWriteGroups`标志：

```csharp
public class AddingSystem : SystemBase
{
   protected override void OnUpdate() {
      Entities
         .WithEntityQueryOptions(EntityQueryOptions.FilterWriteGroup) 
         .ForEach((ref W w, in B b) => {
            // 在这里执行一些计算
         }).ScheduleParallel();}
}
```

对于EntityQueryDesc，在创建query时设置标志：

```csharp
public class AddingSystem : SystemBase
{
   private EntityQuery m_Query;

   protected override void OnCreate()
   {
       var queryDescription = new EntityQueryDesc
       {
           All = new ComponentType[] {
              ComponentType.ReadWrite<W>(),
              ComponentType.ReadOnly<B>()
           },
           Options = EntityQueryOptions.FilterWriteGroup
       };
       m_Query = GetEntityQuery(queryDescription);
   }
   // Define IJobChunk struct and schedule...
}
```

在query中启用 write group filtering时，query会将可写组件的writegroup中的所有components添加到`None`查询列表中，除非您明确将它们添加到`All`或`Any`列表中。结果就是，query仅在明确要求特定write group中每个component的情况下才选择该entity。否则，如果entity具有该write group中的一个或多个其他components，则query将不再包含他。

在上面的示例代码中，query：

- 排除任何具有component`A`的实体，因为它`W`是可写的，并且`A`是的write group的一部分`W`。
- 不排除任何具有component`B`的实体。即使`B`是`W` writegroup的一部分，但它在`All`列表中显式指定了。

## 覆盖另一个使用Write groups的system

如果system在其query中使用write group filter，则可以使用自己的system覆盖该system并写入那些components。要覆盖system，请将您自己的component添加到其他system要写入的component的write group中。由于write group filter排除了query中没有明确要求的write group中的任何component，因此其他system将忽略具有您自定义component的任何entity。

例如，如果要通过指定旋转的角度和轴来设置entity的方向，则可以创建一个component和一个system，以将角度和轴的值转换为四元数并将其写入到`Unity.Transforms.Rotation`component中。为了防止`Unity.Transforms`system更新`Rotation`，无论您是否拥有其他components，都可以将`Rotation`component放入以下write group中：

```csharp
using System;
using Unity.Collections;
using Unity.Entities;
using Unity.Transforms;
using Unity.Mathematics;

[Serializable]
[WriteGroup(typeof(Rotation))]
public struct RotationAngleAxis : IComponentData
{
   public float Angle;
   public float3 Axis;
}
```

然后，您可以使用该`RotationAngleAxis`component更新任何entity而不会发生冲突：

```csharp
using Unity.Burst;
using Unity.Entities;
using Unity.Jobs;
using Unity.Collections;
using Unity.Mathematics;
using Unity.Transforms;

public class RotationAngleAxisSystem : SystemBase
{
   protected override void OnUpdate()
   {
      Entities.ForEach((ref Rotation destination, in RotationAngleAxis source) =>
      {
         destination.Value 
             = quaternion.AxisAngle(math.normalize(source.Axis), source.Angle);
      }).ScheduleParallel();
   }
}
```

## 扩展使用Write groups的另一个System

如果要扩展另一个System而不是覆盖它，或者要允许将来的system覆盖或扩展系统，则可以在自己的system上启用write group filter。但是，执行此操作时，默认情况下，两个system都不会处理component的任何组合。您必须显式查询和处理每个组合。

在前面的示例中，它定义了一个write group，其中包含component`A`，`B`并且目标组是component `W`。如果将一个名为`C`的新component添加到write group，则新system可以查询包含`C`的entities，它并不关心这些entities是否也具有component`A`或`B`。但是，如果新system还启用了write group filter功能，那就不再成立了。如果只需要component`C`，则write group filter将排除带有`A`或`B`的任何entity。您必须显式查询有意义的每种component组合。**注意：**您可以在适当的时候使用查询`Any`的子句。

```csharp
var query = new EntityQueryDesc
{
    All = new ComponentType[] {
       ComponentType.ReadOnly<C>(), 
       ComponentType.ReadWrite<W>()
    },
    Any = new ComponentType[] {
       ComponentType.ReadOnly<A>(), 
       ComponentType.ReadOnly<B>()
    },
    Options = EntityQueryOptions.FilterWriteGroup
};
```

如果有任何entity包含未明确提及的write group中components的组合，则写入write group目标的system及其过滤器将无法处理它们。但是，如果您拥有任何这些类型的entities，则很可能是程序中的逻辑错误，因此它们不应该存在。

## Write Group总结

Write Group对于ECS来说，其实就是一个OOP里的虚函数重写时要不要调用base.XXX()的功能

# 版本控制

版本号检测潜在的更改。您可以使用它们来实施有效的优化策略，例如，自从应用程序的最后一帧以来数据没有更改时，跳过处理。对entity执行快速版本检查以提高应用程序的性能非常有用。

本页概述了ECS使用的所有不同版本号，以及导致它们更改的条件。

所有版本号都是32位有符号整数。它们总是增加，除非它们溢出：有符号整数溢出是C＃中定义的行为。这意味着要比较版本号，应使用*等式/不等式运算符，而不是关系运算符(我咋感觉这句话说反了)。*

例如，检查VersionB是否比VersionA更新的正确方法是使用以下方法：

```c#
bool VersionBIsMoreRecent = (VersionB - VersionA) > 0;
```

通常无法保证版本号增加多少。

## EntityId.Version

一个`EntityId`由索引和版本号组成。由于ECS会回收索引，因此`EntityManager`在entity被破坏时它的版本号都会增加。如果在`EntityManager`中查找`EntityId`时版本号不匹配，则意味着所引用的entity不再存在。

例如，在您通过`EntityId`获取部队正在跟踪的敌人的位置之前，您可以调用`ComponentDataFromEntity.Exists`。这使用版本号来检查entity是否仍然存在。

## World.Version

ECS每次创建或销毁管理器（比如system）时，都会增加World的版本号。

## EntityDataManager.GlobalVersion

`EntityDataManager.GlobalVersion` 在每个job component system更新之前增加。

您应该将此版本号与`System.LastSystemVersion`结合使用。

## System.LastSystemVersion

`System.LastSystemVersion``EntityDataManager.GlobalVersion`每次job component system更新后，取的值。

您应该将此版本号与`Chunk.ChangeVersion[]`结合使用。

## Chunk.ChangeVersion

对于archetype中的每种component类型，此数组包含该chunk中该component最后一次被访问时可写`EntityDataManager.GlobalVersion`的值。这并不意味任何内容都已更改，只是说可能已更改。

即使存储了sharedcomponent的版本号，您也永远无法以可写方式访问它们：所以这并没有什么卵用。

当您`WithChangeFilter()`在`Entities.ForEach`构造中使用该功能时，ECS会将特定component的`Chunk.ChangeVersion`与`System.LastSystemVersion`进行比较，并且它仅处理其component数组在system上次开始运行后被访问为可写的chunk。

例如，如果确保自上一帧以来一组单位的生命值未发生变化，则可以跳过检查这些单位是否应更新其损坏模型的步骤。

## EntityManager.m_ComponentTypeOrderVersion []

对于每种非shared component类型，每当涉及该类型的迭代器无效时，ECS都会增加版本号。也就是，任何可能修改该类型数组的内容（而不是实例）。

例如，如果您具有特定component标识的静态对象和每个chunk的边界框，则仅当该component的类型顺序版本发生更改时，才需要更新这些边界框。

## SharedComponentDataManager.m_SharedComponentVersion []

当存储在引用sharedcomponent的chunk中的entity发生任何结构更改时，这些版本号会增加。

例如，如果您保留每个sharecomponents的entity计数，则可以使用该版本号仅在相应版本号发生更改时重计算每个计数。

# Job extensions

Unity C＃ Jobs System使您可以在多个线程上运行代码。该系统提供调度，并行处理和多线程安全性。Jobs System是Unity的核心模块，提供用于创建和运行job的通用接口和类（无论您是否使用ECS）。

这些接口包括：

- [IJob](https://docs.unity3d.com/ScriptReference/Unity.Jobs.IJob.html)：创建一个可以在由Jobs System scheduler决定的任何线程或核心上运行的job。
- [IJobParallelFor](https://docs.unity3d.com/ScriptReference/Unity.Jobs.IJobParallelFor.html)：创建一个可以在多个线程上并行运行的作业，以处理[NativeContainer](https://docs.unity3d.com/Manual/JobSystemNativeContainer.html)的元素。
- [IJobExtensions](https://docs.unity3d.com/ScriptReference/Unity.Jobs.IJobExtensions.html)：提供扩展方法以运行IJobs。
- [IJobParalllelForExtensions](https://docs.unity3d.com/ScriptReference/Unity.Jobs.IJobParallelForExtensions.html)：提供扩展方法以运行IJobParallelFor job。
- [JobHandle](https://docs.unity3d.com/ScriptReference/Unity.Jobs.JobHandle.html)：访问job的句柄。您还可以使用`JobHandle`实例来指定job之间的依赖关系。

有关Job System的概述，请参见《 Unity用户手册》中的[C＃Jobs System](https://docs.unity3d.com/Manual/JobSystemSafetySystem.html)。

该[Job Package](https://docs.unity3d.com/Packages/com.unity.jobs@latest)扩展了Jobs System，以支持ECS。它包含了：

- [IJobParallelForDeferExtensions](https://docs.unity3d.com/Packages/com.unity.jobs@latest?preview=1&subfolder=/api/Unity.Jobs.IJobParallelForDeferExtensions.html)
- [IJobParallelForFilter](https://docs.unity3d.com/Packages/com.unity.jobs@latest?preview=1&subfolder=/api/Unity.Jobs.IJobParallelForFilter.html)
- [JobParallelIndexListExtensions](https://docs.unity3d.com/Packages/com.unity.jobs@latest?preview=1&subfolder=/api/Unity.Jobs.JobParallelIndexListExtensions.html)
- [`JobStructProduce<T>`](https://docs.unity3d.com/Packages/com.unity.jobs@latest?preview=1&subfolder=/api/Unity.Jobs.JobParallelIndexListExtensions.JobStructProduce-1.html)
