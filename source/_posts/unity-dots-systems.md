---
title: Unity DOTS：Systems部分
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

# Systems

一个**system**，也就是ECS里的S，提供了将component的数据从其当前状态变换到其下一个状态的逻辑-例如，一个system可以通过velocity乘以Time.deltaime来更新所有可移动entities的位置。

![image-20200813144636072](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200813144636072.png)



## Instantiating systems

Unity ECS自动在您的项目中发现system类型，并在运行时实例化它们。它将每个发现的system添加到默认system groups之一中。您可以使用[system attributes](https://docs.unity3d.com/Packages/com.unity.entities@0.13/manual/system_update_order.html#attributes)来指定system的父组以及该system在该group中的顺序。如果未指定父项，则Unity将以确定性的，但并未指定顺序的将system添加到默认世界的Simulation system group中。您也可以使用attribute禁用自动创建。

system的更新循环由其父[ComponentSystemGroup](https://docs.unity3d.com/Packages/com.unity.entities@0.13/manual/system_update_order.html)驱动。ComponentSystemGroup本身是一种特殊的system，负责更新其child systems。group可以嵌套。system从运行的[world](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.World.html)获取[time](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Core.TimeData.html)数据；time由[UpdateWorldTimeSystem](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.UpdateWorldTimeSystem.html)更新。

您可以使用*Entity Debugger window*（menu: **Window** > **Analysis** > **Entity Debugger**）查看system configuration。



## System类型

Unity ECS提供了几种类型的systems。通常，为实现游戏行为和数据转换而编写的system将扩展[SystemBase](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html)。其他system类具有特殊目的。比如，通常情况下，您使用[EntityCommandBufferSystem](https://docs.unity3d.com/Packages/com.unity.entities@0.13/manual/entity_command_buffer.html)和[ComponentSystemGroup](https://docs.unity3d.com/Packages/com.unity.entities@0.13/manual/system_update_order.html)类的现有实例，而不是自己再进行拓展。

- [SystemBase-](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html)创建自定义system时要实现的基类。
- [EntityCommandBufferSystem-](https://docs.unity3d.com/Packages/com.unity.entities@0.13/manual/entity_command_buffer.html)为其他systems提供[EntityCommandBuffer](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.EntityCommandBuffer.html)实例。每个默认system group在其child system列表的开头和结尾都维护一个“ Entity Command Buffer System”。这使您可以对结构更改进行分组，以使它们在框架中产生更少的[syncronization points](https://docs.unity3d.com/Packages/com.unity.entities@0.13/manual/sync_points.html)。
- [ComponentSystemGroup-](https://docs.unity3d.com/Packages/com.unity.entities@0.13/manual/system_update_order.html)为其他systems提供嵌套的组织和更新顺序。默认情况下，Unity ECS创建多个Component System Groups。
- [GameObjectConversionSystem-](https://docs.unity3d.com/Packages/com.unity.entities@0.13/manual/gp_overview.html)将游戏的GameObject的转换为高效的entityGame conversion systems在Unity编辑器中运行。

**重要提示：**[ComponentSystem](https://docs.unity3d.com/Packages/com.unity.entities@0.5/manual/entity_iteration_foreach.html)和[JobComponentSystem](https://docs.unity3d.com/Packages/com.unity.entities@0.5/manual/entities_job_foreach.html)类，以及[IJobForEach](https://docs.unity3d.com/Packages/com.unity.entities@0.5/manual/entity_iteration_job.html)，这些都是被淘汰的DOTS API，但是还没有官宣。请改用[SystemBase](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html)和[Entities.ForEach](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Entities)。

## 创建一个system

实现抽象类[SystemBase](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html)创建来ECS中的systems。

要创建system，需要实现必要的一些生命周期函数回调。使用[SystemBase OnUpdate（）](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_OnUpdate_)函数执行system必须在每一帧中完成的工作。其他回调函数是可选的。例如，您可以使用[OnCreate（）](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.ComponentSystemBase.html#Unity_Entities_ComponentSystemBase_OnCreate_)来初始化system，但并非每个系统都需要初始化。

system回调函数按以下顺序调用：

![image-20200813151442570](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200813151442570.png)

- [OnCreate（）](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.ComponentSystemBase.html#Unity_Entities_ComponentSystemBase_OnCreate_) -创建system时调用。
- [OnStartRunning（）](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.ComponentSystemBase.html#Unity_Entities_ComponentSystemBase_OnStartRunning_) -在第一个[OnUpdate（）之前](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_OnUpdate_)以及每当system恢复运行时调用。
- [OnUpdate（）](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_OnUpdate_) -system有工作要做的每一帧（请参见[ShouldRunSystem（）](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.ComponentSystemBase.html#Unity_Entities_ComponentSystemBase_ShouldRunSystem_)）并且系统已[启用](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.ComponentSystemBase.html#Unity_Entities_ComponentSystemBase_Enabled)时调用。
- [OnStopRunning（）](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.ComponentSystemBase.html#Unity_Entities_ComponentSystemBase_OnStopRunning_)－每当system停止更新时调用，这可能是由于将[Enabled](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.ComponentSystemBase.html#Unity_Entities_ComponentSystemBase_Enabled)设置为false或因为找不到与它的query匹配的entities。在[OnDestroy（）](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.ComponentSystemBase.html#Unity_Entities_ComponentSystemBase_OnDestroy_)之前也会调用。
- [OnDestroy（）](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.ComponentSystemBase.html#Unity_Entities_ComponentSystemBase_OnDestroy_) -销毁system时调用。

system的[OnUpdate（）](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_OnUpdate_)函数由其[父系统组](https://docs.unity3d.com/Packages/com.unity.entities@0.13/manual/system_update_order.html#groups)自己的[OnUpdate（）](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_OnUpdate_)函数触发。同样，当group更改状态时，例如，如果您设置了该group的[Enabled](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.ComponentSystemBase.html#Unity_Entities_ComponentSystemBase_Enabled)属性，它也会更改其子systems的状态。但是，子systems也可以独立于parent group而改变状态。有关更多信息，请参见[system update order](https://docs.unity3d.com/Packages/com.unity.entities@0.13/manual/system_update_order.html)。

所有system生命周期函数都在主线程上运行。理想情况下，您的[OnUpdate（）](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_OnUpdate_)函数可以schedules jobs以执行大部分工作。要从system schedules jobs，您可以使用以下机制之一：

- [Entities.ForEach-](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Entities)迭代ECS component数据的最简单方法。
- [Job.WithCode-](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Job)将lambda函数作为单个后台job执行。
- [IJobChunk-](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.IJobChunk.html)一种“底层”机制，用于逐chunk迭代ECS components数据。
- [C＃Job System](https://docs.unity3d.com/Manual/JobSystem.html) -创建和规划通用的C＃ Job。

以下示例说明了使用[Entities.ForEach](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Entities)来实现一种system，该system根据一个component来更新另一个component的值：

```cs
public struct Position : IComponentData
{
    public float3 Value;
}

public struct Velocity : IComponentData
{
    public float3 Value;
}

public class ECSSystem : SystemBase
{
    protected override void OnUpdate()
    {
        // Local variable captured in ForEach
        // 在ForEach中被使用的本地变量
        float dT = Time.DeltaTime;

        Entities
            .WithName("Update_Displacement")
            .ForEach(
                (ref Position position, in Velocity velocity) =>
                {
                    position = new Position()
                    {
                        Value = position.Value + velocity.Value * dT
                    };
                }
            )
            .ScheduleParallel();
    }
}
```

# 使用Entities.ForEach创建systems

使用[SystemBase](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html)类提供的[Entities.ForEach](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Entities)构造作为在entities及其components上定义和执行算法的简洁方法。[Entities.ForEach](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Entities)在由[entity query](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.EntityQuery.html)选择出的的所有[entities](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Entities)上执行您定义的lambda函数。

要执行job Lambda函数，您可以使用`Schedule()`和`ScheduleParallel()`来schedule job，或者使用`Run()`来立即执行该job（在*主线程*上）。您可以使用在[Entities.ForEach](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Entities)上定义的其他方法来设置entity query以及各种job选项。

下面的示例说明了一个简单的[SystemBase](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html)实现，该实现使用[Entities.ForEach](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Entities)读取一个entity的component（为Velocity）并写入另一个component（Translation）：

```cs
class ApplyVelocitySystem : SystemBase
{
    protected override void OnUpdate()
    {
        Entities
            .ForEach((ref Translation translation,
            in Velocity velocity) =>
            {
                translation.Value += velocity.Value;
            })
            .Schedule();
    }
}
```

请注意ForEach lambda函数的参数关键字`ref`以及`in`的使用。使用`ref`修饰的component，可读写，使用`in`修饰的component，只读。将component标记为只读可帮助job scheduler程序更有效地执行job。

## 选择entities

[Entities.ForEach](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Entities)提供了自己的机制，用于定义用于选择要处理的entities的entity query。该query自动包括您符合lambda函数参数的所有components。你也可以使用`WithAll`，`WithAny`及`WithNone`条款，以进一步细化哪些entities被选中。有关query选项的完整内容，请参见[SystemBase.Entities](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Entities)。

下面的示例选择具有以下components的entities：Destination，Source和LocalToWorld。并具有Rotation，Translation或Scale中的至少一项；但没有LocalToParent。

```cs
Entities.WithAll<LocalToWorld>()
    .WithAny<Rotation, Translation, Scale>()
    .WithNone<LocalToParent>()
    .ForEach((ref Destination outputData, in Source inputData) =>
    {
        /* do some work */
    })
    .Schedule();
```

在此示例中，在lambda函数内部只能访问目标和源components，因为它们是参数列表中的唯一的components。

### 访问EntityQuery对象

要访问[Entities.ForEach](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Entities)创建的[EntityQuery](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.EntityQuery.html)对象，请使用[WithStoreEntityQueryInField（ref query）]和ref参数修饰符。此函数将query的引用分配给您提供的字段。

下面的示例说明如何访问为[Entities.ForEach](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Entities)构造隐式创建的EntityQuery对象。在这种情况下，该示例使用EntityQuery对象调用[CalculateEntityCount（）](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.EntityQuery.html#Unity_Entities_EntityQuery_CalculateEntityCount_)方法。该示例使用此计数来创建一个本地化数组，该数组具有足够的空间来为query选择的每个entity存储一个值：

```cs
private EntityQuery query;
protected override void OnUpdate()
{
    int dataCount = query.CalculateEntityCount();
    NativeArray<float> dataSquared
        = new NativeArray<float>(dataCount, Allocator.Temp);
    Entities
        .WithStoreEntityQueryInField(ref query)
        .ForEach((int entityInQueryIndex, in Data data) =>
        {
            dataSquared[entityInQueryIndex] = data.Value * data.Value;
        })
        .ScheduleParallel();

    Job
        .WithCode(() =>
    {
        //Use dataSquared array...
        var v = dataSquared[dataSquared.Length - 1];
    })
        .WithDisposeOnCompletion(dataSquared)
        .Schedule();
}
```



### 可选components

您无法创建指定可选component的query（使用`WithAny<T，U>`），也无法在lambda函数中访问那些components。如果需要读取或写入可选component，则可以将Entities.ForEach构造拆分为多个job(每个可选components的组合是一个job)。

例如，如果您有两个可选components，有两种方案

- 需要三个ForEach构造：一个包含第一个可选component，一个包含第二个可选component，一个包含两个components
- 另一种选择是使用IJobChunk逐chunk进行迭代。



### 更改filtering（过滤选项）

如果自上次运行当前[SystemBase](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html)实例以来，仅想在该component的另一个entity发生更改时处理该entity component，则可以使用`WithChangeFilter <T>`启用更改filtering。更改filter中使用的component类型必须处于lambda函数参数列表中，或者必须是`WithAll <T>`语句的一部分。

```cs
Entities
    .WithChangeFilter<Source>()
    .ForEach((ref Destination outputData,
        in Source inputData) =>
        {
            /* Do work */
        })
    .ScheduleParallel();
```

entity query最多支持两种component类型的change filtering。

*请注意，change filtering应用于chunk级别。如果有任何代码通过写访问权限访问了chunk中的某个component，则该chunk中该component的类型将标记为已更改-即使该代码实际上并未更改任何数据。*



### Shared component filtering

具有shared component的entity与其他具有相同shared component值的entities分组在一起。您可以使用WithSharedComponentFilter（）函数选择具有特定shared component值的entities group。

以下示例选择按Cohort ISharedComponentData分组的entities。此示例中的lambda函数根据entity的Cohort设置DisplayColor IComponentData组件：

```cs
public class ColorCycleJob : SystemBase
{
    protected override void OnUpdate()
    {
        List<Cohort> cohorts = new List<Cohort>();
        EntityManager.GetAllUniqueSharedComponentData<Cohort>(cohorts);
        foreach (Cohort cohort in cohorts)
        {
            DisplayColor newColor = ColorTable.GetNextColor(cohort.Value);
            Entities.WithSharedComponentFilter(cohort)
                .ForEach((ref DisplayColor color) => { color = newColor; })
                .ScheduleParallel();
        }
    }
}
```

该示例使用EntityManager来获取所有不同的的Cohort值。然后，它为每个Cohort调度一个lambda job，将新颜色作为捕获变量传递给lambda函数。



## 定义ForEach函数

定义与[Entities.ForEach](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Entities)一起使用的lambda函数时，可以声明参数，[SystemBase](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html)类在执行该函数时使用该参数传递有关当前entity的信息。

典型的lambda函数如下所示：

```cs
Entities.ForEach(
    (Entity entity,
        int entityInQueryIndex,
        ref Translation translation,
        in Movement move) => { /* .. */})
```

默认情况下，您最多可以将八个参数传递给Entities.ForEach lambda函数。（如果需要传递更多参数，则可以[定义自定义委托](https://docs.unity3d.com/Packages/com.unity.entities@0.13/manual/ecs_entities_foreach.html#custom-delegates)。）使用标准委托时，必须按以下顺序对参数进行分组：



```cs
Parameters passed-by-value first(参数传递的值) (no parameter modifiers(无修饰符))
Writable parameters second(可读写参数)(`ref` parameter modifier(ref修饰符))
Read-only parameters last (只读参数)(`in` parameter modifier(in修饰符))
```



所有components都应使用`ref`或`in`参数修饰符。否则，传递给您的函数的components struct是副本而不是引用。这意味着为只读参数提供了额外的内存副本，并且意味着在函数返回后（复制的结构超出范围时）对要更新的component的任何更改都会被静默丢弃。

如果您的函数不遵守这些规则，并且您尚未创建合适的委托，则编译器将提供类似于以下内容的错误：



```cs
error CS1593: Delegate 'Invalid_ForEach_Signature_See_ForEach_Documentation_For_Rules_And_Restrictions' does not take N argumentscs
```

（请注意，即使问题是参数顺序，错误消息也会将会是这个报错。）



### 自定义委托

您可以在ForEach lambda函数中使用8个以上的参数。通过声明自己的委托类型和ForEach重载。这使您可以根据需要使用任意数量的参数，并以任意顺序放置ref / in / value参数。

你可以在任何地方声明有三个[特殊，命名参数](https://docs.unity3d.com/Packages/com.unity.entities@0.13/manual/ecs_entities_foreach.html#named-parameters) `entity`，`entityInQueryIndex`和`nativeThreadIndex`参数列表的委托。**但请勿对这些参数使用`ref`或`in`修饰符。**

```cs
static class BringYourOwnDelegate
{
    // Declare the delegate that takes 12 parameters. T0 is used for the Entity argument
    // 声明12个参数的委托，T0被用来当做Entity参数
    [Unity.Entities.CodeGeneratedJobForEach.EntitiesForEachCompatible]
    public delegate void CustomForEachDelegate<T0, T1, T2, T3, T4, T5, T6, T7, T8, T9, T10, T11>
        (T0 t0, in T1 t1, in T2 t2, in T3 t3, in T4 t4, in T5 t5,
         in T6 t6, in T7 t7, in T8 t8, in T9 t9, in T10 t10, in T11 t11);

    // Declare the function overload
    // 声明ForEach重载
    public static TDescription ForEach<TDescription, T0, T1, T2, T3, T4, T5, T6, T7, T8, T9, T10, T11>
        (this TDescription description, CustomForEachDelegate<T0, T1, T2, T3, T4, T5, T6, T7, T8, T9, T10, T11> codeToRun)
        where TDescription : struct, Unity.Entities.CodeGeneratedJobForEach.ISupportForEachWithUniversalDelegate =>
        LambdaForEachDescriptionConstructionMethods.ThrowCodeGenException<TDescription>();
}

// A system that uses the custom delegate and overload
// 一个使用自定义委托和重载的示例
public class MayParamsSystem : SystemBase
{
    protected override void OnUpdate()
    {
        Entities.ForEach(
                (Entity entity0,
                    in Data1 d1,
                    in Data2 d2,
                    in Data3 d3,
                    in Data4 d4,
                    in Data5 d5,
                    in Data6 d6,
                    in Data7 d7,
                    in Data8 d8,
                    in Data9 d9,
                    in Data10 d10,
                    in Data11 d11
                    ) => {/* .. */})
            .Run();
    }
}
```

**注意：**ForEach lambda函数默认八个参数限制，因为声明太多的委托和重载会对IDE性能产生负面影响。ref / in / value和参数数量的每种组合都需要唯一的委托类型和ForEach重载。



### Component参数

要访问与entity关联的component，您必须将该component类型的参数传递给lambda函数。编译器会自动将传递给函数的所有components作为必需components添加到entity query中。

要更新component值，必须使用`ref`参数列表中的关键字通过引用将其传递给lambda函数。（没有`ref`关键字，将对component的临时副本进行任何修改，因为它将通过值进行传递。）

要将传递给lambda函数的components指定为只读，请在参数列表中使用`in`关键字。

**注意：**使用`ref`表示当前chunk中的components被标记为已更改，即使lambda函数实际上并未对其进行修改。为了提高效率，请始终使用`in`关键字将lambda函数不会修改的components指定为只读。

以下示例将Source component作为只读参数传递给job，并将Destination component作为可写参数传递给作业：

```cs
Entities.ForEach(
    (ref Destination outputData,
        in Source inputData) =>
    {
        outputData.Value = inputData.Value;
    })
    .ScheduleParallel();
```

**注意：**当前，您不能将chunk component传递给Entities.ForEach lambda函数。

对于dynamic buffers，请使用`DynamicBuffer <T>`而不是存储在buffers中的Component类型：

```cs
public class BufferSum : SystemBase
{
    private EntityQuery query;

    //Schedules the two jobs with a dependency between them
    //使用一个在两个job之间的一个dependency来规划他们
    protected override void OnUpdate()
    {
        //The query variable can be accessed here because we are
        //using WithStoreEntityQueryInField(query) in the entities.ForEach below
        //这个query的变量可以在这里获取到，是因为我们在这个entities.ForEach中使用了WithStoreEntityQueryInField(query)
        int entitiesInQuery = query.CalculateEntityCount();

        //Create a native array to hold the intermediate sums
        //(one element per entity)
        //创建一个本地化的数组来存储中间和（每个entity的元素）
        NativeArray<int> intermediateSums
            = new NativeArray<int>(entitiesInQuery, Allocator.TempJob);

        //Schedule the first job to add all the buffer elements
        //规划第一个job来相加所有的缓冲区内元素
        Entities
            .ForEach((int entityInQueryIndex, in DynamicBuffer<IntBufferData> buffer) =>
        {
            for (int i = 0; i < buffer.Length; i++)
            {
                intermediateSums[entityInQueryIndex] += buffer[i].Value;
            }
        })
            .WithStoreEntityQueryInField(ref query)
            .WithName("IntermediateSums")
            .ScheduleParallel(); // Execute in parallel for each chunk of entities
        //并行执行每个chunk中的entities

        //Schedule the second job, which depends on the first
        //规划第二个job，他依赖第一个job
        Job
            .WithCode(() =>
        {
            int result = 0;
            for (int i = 0; i < intermediateSums.Length; i++)
            {
                result += intermediateSums[i];
            }
            //Not burst compatible:
            //对burst不兼容
            Debug.Log("Final sum is " + result);
        })
            .WithDisposeOnCompletion(intermediateSums)
            .WithoutBurst()
            .WithName("FinalSum")
            .Schedule(); // Execute on a single, background thread
        //执行在一个单一的后台线程
    }
}
```



### 特殊的命名参数

除了component之外，您还可以将以下特殊的命名参数传递给Entities.ForEach lambda函数，这些参数是根据job当前正在处理的entity分配的值：

- **`Entity entity`**—当前entity的Entity实例。（参数的名称可以是任何类型，只要类型是Entity。）
- **`int entityInQueryIndex`**—该entity在query选择的所有entities的列表中的索引。当您有一个[本地化数组](https://docs.unity3d.com/ScriptReference/Unity.Collections.NativeArray_1.html)需要为每个entity填充一个唯一值时，请使用entityInQueryIndex。您可以将entityInQueryIndex用作该数组中的索引。EntityInQueryIndex也应用作`sortKey`将命令添加到并发[EntityCommandBuffer](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.EntityCommandBuffer.html)。
- **`int nativeThreadIndex`**—执行lambda函数当前迭代的线程的唯一索引。使用Run（）执行lambda函数时，nativeThreadIndex始终为零。（不要将`nativeThreadIndex`用作并发[EntityCommandBuffer](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.EntityCommandBuffer.html)的`sortKey`；请改为使用`entityInQueryIndex`。）



## 捕获变量

您可以捕获Entities.ForEach lambda函数的局部变量。使用job执行函数时（通过调用Schedule几个函数之一而不是Run），对捕获的变量及其使用方式有一些限制：

- 只能捕获本地化容器和可漂白类型。
- job只能写入类型为本地化容器的捕获变量。（要“返回”单个值，请使用一个元素创建一个[本地化数组](https://docs.unity3d.com/ScriptReference/Unity.Collections.NativeArray_1.html)。）

如果您读取了[本地化容器]，但未写入该容器，请始终使用来指定只读访问权限`WithReadOnly(variable)`。有关设置捕获变量的属性的更多信息，请参见[SystemBase.Entities](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Entities)。您可以指定的属性包括`NativeDisableParallelForRestriction`及其他。[Entities.ForEach](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Entities)将这些作为函数提供，因为C＃语言不允许在局部变量上使用attribute。

您还可以使用表示要在[Entities.ForEach](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Entities)运行之后Dispose捕获的NativeContainer或包含NativeContainers的类型`WithDisposeOnCompletion(variable)`。这将在lambda运行之后立即Dispose类型（对于`Run()`），或者安排它们稍后通过Job进行Dispose并返回JobHandle（对于`Schedule()`/ `ScheduleParallel()`）。

**注意：**在通过`Run()`执行函数时，您可以写入不是本地化容器的捕获变量。但是，您仍应尽可能使用[blittable](https://docs.microsoft.com/en-us/dotnet/framework/interop/blittable-and-non-blittable-types)类型，以便可以使用[Burst](https://docs.unity3d.com/Packages/com.unity.burst@latest/index.html)编译函数。

## 支持的功能

您可以使用在主线程上使用`Run()`执行lambda函数，在单个后台线程使用`Schedule()`执行job，或者使用`ScheduleParallel()`来让多线程并行执行。这些不同的执行方法对访问数据的方式具有不同的约束。另外，[Burst](https://docs.unity3d.com/Packages/com.unity.burst@latest/index.html)使用C＃语言的受限子集，在此子集之外使用C＃功能（包括访问托管类型）时您需要指定`WithoutBurst()`。

下表显示了[Entities.ForEach](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Entities)当前支持哪些功能，用于[SystemBase中](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html)可用的各种计划方法：

| 支持功能                   | Run                                     | Schedule | ScheduleParallel    |
| :------------------------- | :-------------------------------------- | :------- | :------------------ |
| 捕获局部值类型             | X                                       | X        | X                   |
| 捕获局部引用类型           | x（仅不带Burst）                        |          |                     |
| 写入捕获的变量             | X                                       |          |                     |
| System命名空间下的字段     | x（仅不带Burst）                        |          |                     |
| 引用类型的方法             | x（仅不带Burst）                        |          |                     |
| Shared Components          | x（仅不带Burst）                        |          |                     |
| Managed Components         | x（仅不带Burst）                        |          |                     |
| 结构体变化                 | x（仅不带Burst和WithStructuralChanges） |          |                     |
| SystemBase.GetComponent    | X                                       | X        | X                   |
| SystemBase.SetComponent    | X                                       | X        |                     |
| GetComponentDataFromEntity | X                                       | X        | x（仅作为ReadOnly） |
| HasComponent               | X                                       | X        | X                   |
| WithDisposeOnCompletion    | X                                       | X        | X                   |

一个[Entities.ForEach](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Entities)构造使用专门的中间语言（IL）编译后处理你写的代码来转换成正确的ECS的代码。这种自动翻译使您无需包含复杂的样板代码即可表达算法的意图。但是，这可能意味着不允许使用某些常见的代码编写方式。

当前不支持以下功能：

| 不支持的功能                                                |
| :---------------------------------------------------------- |
| Dynamic code in .With invocations                           |
| 被ref修饰的SharedComponent参数                              |
| 嵌套的Entities.ForEach lambda expressions                   |
| 标有[ExecuteAlways]的系统中的Entities.ForEach（当前已修复） |
| 使用存储在变量，字段或方法中的委托进行调用                  |
| 具有lambda参数类型的SetComponent                            |
| 具有可写lambda参数的GetComponent                            |
| Lambdas中的泛型参数                                         |
| 在具有泛型参数的systems中                                   |

## Dependencies

默认情况下，系统使用其[Dependency](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Dependency)属性管理与ECS相关的依赖[关系](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Dependency)。默认情况下，系统将按[Entities.ForEach](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Entities)和[Job.WithCode] 创建的每个job按它们在[OnUpdate（）](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_OnUpdate_)函数中出现的顺序添加到[Dependency](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Dependency) job句柄中。您还可以通过将[JobHandle]传递给函数来手动管理job依赖关系，然后返回结果依赖关系。有关更多信息，请参见[依赖性](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Dependency)。`Schedule`

有关[job依赖性](https://docs.unity3d.com/Packages/com.unity.entities@0.13/manual/ecs_job_dependencies.html)的更多常规信息，请参见job依赖性。

# 使用Job.WithCode创建Systems

[SystemBase](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html)类提供的[Job.WithCode](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Job)构造是一种将函数作为单个后台job运行的简便方法。您甚至可以在主线程上运行[Job.WithCode](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Job)，并且仍然可以利用[Burst](https://docs.unity3d.com/Packages/com.unity.burst@latest/index.html)编译来加快执行速度。

以下示例使用一个[Job.WithCode](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Job) lambda函数用随机数填充[本地化数组](https://docs.unity3d.com/ScriptReference/Unity.Collections.NativeArray_1.html)，并使用另一个job将这些数字加在一起：

```cs
public class RandomSumJob : SystemBase
{
    private uint seed = 1;

    protected override void OnUpdate()
    {
        Random randomGen = new Random(seed++);
        NativeArray<float> randomNumbers
            = new NativeArray<float>(500, Allocator.TempJob);

        Job.WithCode(() =>
        {
            for (int i = 0; i < randomNumbers.Length; i++)
            {
                randomNumbers[i] = randomGen.NextFloat();
            }
        }).Schedule();

        // To get data out of a job, you must use a NativeArray
        // even if there is only one value
        // 想要在job外获取data，你必须使用一个本地化数组
        // 即使他只有一个值
        NativeArray<float> result
            = new NativeArray<float>(1, Allocator.TempJob);

        Job.WithCode(() =>
        {
            for (int i = 0; i < randomNumbers.Length; i++)
            {
                result[0] += randomNumbers[i];
            }
        }).Schedule();

        // This completes the scheduled jobs to get the result immediately, but for
        // better efficiency you should schedule jobs early in the frame with one
        // system and get the results late in the frame with a different system.
        // 这个语句会立即完成调度来获取结果，但是为了更好的效率，你应该在这一帧的早些时候使用一个system调度这个job，并且在这一帧的晚些时候使用另一个system获取结果
        this.CompleteDependency();
        UnityEngine.Debug.Log("The sum of "
            + randomNumbers.Length + " numbers is " + result[0]);

        randomNumbers.Dispose();
        result.Dispose();
    }
}
```

**注意：**要运行并行的job，请实现[IJobFor](https://docs.unity3d.com/Manual/JobSystemCreatingJobs.html)，您可以使用系统[OnUpdate（）](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_OnUpdate_)函数中的[ScheduleParallel（）](https://docs.unity3d.com/ScriptReference/Unity.Jobs.IJobForExtensions.ScheduleParallel.html)进行[调度](https://docs.unity3d.com/ScriptReference/Unity.Jobs.IJobForExtensions.ScheduleParallel.html)。

## 变量

您不能将参数传递给[Job.WithCode](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Job) lambda函数或返回一个值。取而代之的是，您可以在[OnUpdate（）](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_OnUpdate_)函数中捕获局部变量。

当你在[C＃Job System](https://docs.unity3d.com/Manual/JobSystem.html)中使用`Schedule()`调度你的job时，还有额外的限制：

- 捕获的变量必须声明为 [NativeArray-](https://docs.unity3d.com/ScriptReference/Unity.Collections.NativeArray_1.html)或其他[本地化容器](https://docs.unity3d.com/Manual/JobSystemNativeContainer.html) -或[blittable](https://docs.microsoft.com/en-us/dotnet/framework/interop/blittable-and-non-blittable-types)类型。
- 要返回数据，即使数据是单个值，也必须将返回值写入捕获的[本地化数组](https://docs.unity3d.com/ScriptReference/Unity.Collections.NativeArray_1.html)。（请注意，使用`Run()`时，您可以写入任何捕获的变量。）

[Job.WithCode](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Job)提供了一组函数，以将只读属性和安全属性应用于捕获的[本地化容器](https://docs.unity3d.com/Manual/JobSystemNativeContainer.html)变量。例如，您可以用`WithReadOnly`来指定您不更新容器，并用`WithDisposeOnCompletion`在job结束后自动处理容器。（[Entities.ForEach](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Entities)提供相同的功能。）

有关这些修饰符和属性的更多信息，请参见[Job.WithCode](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Job)。

## 执行函数

您有两种选择来执行lambda函数：

- `Schedule()`-将功能作为单个非并行job执行。调度job在后台线程上运行代码，因此可以更好地利用可用的CPU资源。
- `Run()`-在主线程上立即执行功能。在大多数情况下，可以对[Burst.WithCode](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Job)进行[Burst](https://docs.unity3d.com/Packages/com.unity.burst@latest/index.html)编译，因此即使[Job.WithCode](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Job)仍在主线程上运行，其执行代码也可以更快。

请注意，调用会`Run()`自动完成[Job.WithCode](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Job)构造的所有依赖关系。如果未明确为`Run()`system传入[JobHandle](https://docs.unity3d.com/ScriptReference/Unity.Jobs.JobHandle.html)对象，则假定当前[Dependency](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Dependency)属性表示该函数的依赖关系。（如果函数没有依赖关系，请[传入](https://docs.unity3d.com/ScriptReference/Unity.Jobs.JobHandle.html)新的[JobHandle](https://docs.unity3d.com/ScriptReference/Unity.Jobs.JobHandle.html)。）

## 依存关系

默认情况下，system使用其[Dependency](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Dependency)属性管理与ECS相关的[依赖关系](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Dependency)。system[将按](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Job)[Entities.ForEach](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Entities)和[Job.WithCode](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Job)创建的每个job按它们在[OnUpdate（）](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_OnUpdate_)函数中出现的顺序[添加](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Job)到[Dependency](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Dependency)job句柄中。您还可以通过将[JobHandle](https://docs.unity3d.com/ScriptReference/Unity.Jobs.JobHandle.html)传递给[函数](https://docs.unity3d.com/ScriptReference/Unity.Jobs.JobHandle.html)来手动管理job依赖性，然后将其返回结果依赖性。有关更多信息，请参见[依赖性](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Dependency)。`Schedule`

有关[作业依赖性](https://docs.unity3d.com/Packages/com.unity.entities@0.13/manual/ecs_job_dependencies.html)的更多常规信息，请参见作业依赖性。

# 使用IJobChunk jobs创建System

您可以在system内部实现[IJobChunk](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.IJobChunk.html)，以逐chunk遍历数据。当您在`OnUpdate()`功能中调度IJobChunk job时，该job为每个能匹配上由entity query传递给job `Schedule()`的chunk调用`Excute()`。然后，您可以逐entity地遍历每个chunk内的数据。

与Entities.ForEach相比，使用IJobChunk进行迭代需要更多的代码设置，但是也更明确，并且代表对数据的最直接访问，因为它才是真正被存储的对象。

按chunk进行迭代的另一个好处是，您可以使用`Archetype.Has<T>()`来检查每个chunk中是否存在可选component，然后相应地处理chunk中的所有entities。

要实现IJobChunk job，请使用以下步骤：

1. 创建一个`EntityQuery`以标识要处理的entities。
2. 定义job结构，并包括`ArchetypeChunkComponentType`对象的字段，这些字段标识job必须直接访问的components的类型。另外，指定job是读还是写这些component。
3. 实例化job结构并在system`OnUpdate()`函数中调度job。
4. 在该`Execute()`函数中，获取job读或写的component的`NativeArray`实例，然后在当前chunk上进行迭代以执行所需的工作。

有关更多信息，[ECS samples repository](https://github.com/Unity-Technologies/EntityComponentSystemSamples)包含一个简单的HelloCube示例，演示了如何使用`IJobChunk`。



## 使用EntityQuery查询数据

EntityQuery定义了archetype必须包含的一组components类型，system才能处理其关联的chunks和entities。archetype可以具有其他components，但是它必须至少包含EntityQuery定义的component。您还可以排除包含特定类型components的archetype。

对于简单query，可以使用该`SystemBase.GetEntityQuery()`函数并按如下所示传入component类型：

```cs
public class RotationSpeedSystem : SystemBase
{
    private EntityQuery m_Query;

    protected override void OnCreate()
    {
        m_Query = GetEntityQuery(ComponentType.ReadOnly<Rotation>(),
            ComponentType.ReadOnly<RotationSpeed>());
        //...
    }
}
```

对于更复杂的情况，您可以使用`EntityQueryDesc`。一个`EntityQueryDesc`提供了灵活的查询机制，以指定的组件类型：

- `All`：此数组中的所有components类型必须存在于archetype中
- `Any`：archetype中必须存在此数组中的至少一种component类型
- `None`：archetype中不能存在此数组中的任何component类型

例如，以下查询包括包含`RotationQuaternion`和`RotationSpeed`component的archetypes，但不包括包含`Frozen`component的任何archetype：

```cs
protected override void OnCreate()
{
    var queryDescription = new EntityQueryDesc()
    {
        None = new ComponentType[]
        {
            typeof(Static)
        },
        All = new ComponentType[]
        {
            ComponentType.ReadWrite<Rotation>(),
            ComponentType.ReadOnly<RotationSpeed>()
        }
    };
    m_Query = GetEntityQuery(queryDescription);
}
```

查询使用`ComponentType.ReadOnly<T>`而不是更简单的`typeof`表达式是为了指定system只读`RotationSpeed`。

您还可以组合多个query。为此，请传递`EntityQueryDesc`对象数组而不是单个实例。ECS使用*逻辑或*运算来组合每个query。下面的示例选择包含一个`RotationQuaternion`或多个`RotationSpeed`components（或两者都有）的任何archetype：

```cs
protected override void OnCreate()
{
    var queryDescription0 = new EntityQueryDesc
    {
        All = new ComponentType[] {typeof(Rotation)}
    };

    var queryDescription1 = new EntityQueryDesc
    {
        All = new ComponentType[] {typeof(RotationSpeed)}
    };

    m_Query = GetEntityQuery(new EntityQueryDesc[] {queryDescription0, queryDescription1});
}
```

**注意：**请勿在`EntityQueryDesc`中完全包含可选components。要处理可选components，请使用在`IJobChunk.Execute()`中的`chunk.Has<T>()`方法确定当前ArchetypeChunk是否具有可选components。因为同一chunk中的所有entities具有相同的components，所以您只需要每个chunk检查一个可选component是否存在一次就可以了：而不是每个entity一次。

为了提高效率并避免不必要地创建会有GC的引用类型，应在system`OnCreate()`方法中为system创建`EntityQueries`，然后将结果存储在实例变量中。（在以上示例中，`m_Query`变量正是如此。）



## 定义IJobChunk结构

IJobChunk结构为job运行时所需的数据以及job的`Execute()`方法定义字段。

要访问system传递给您的`Execute()`方法的chunk内的component数组，必须为job读取或写入的每种类型的componnet创建一个`ArchetypeChunkComponentType<T>`对象。您可以使用这些对象来获取`NativeArray`实例，这些实例提供对entities components的访问。包括`Execute()`方法读取或写入的job的EntityQuery中引用的所有components。您还可以为未包含在EntityQuery中的可选component类型提供`ArchetypeChunkComponentType`变量。

在尝试访问当前chunk之前，必须检查以确保当前chunk具有可选component。例如，HelloCube IJobChunk示例声明了一个job结构，该结构定义了两个components的`ArchetypeChunkComponentType<T>`变量。分别是`RotationQuaternion`和`RotationSpeed`：

```cs
[BurstCompile]
struct RotationSpeedJob : IJobChunk
{
    public float DeltaTime;
    public ComponentTypeHandle<Rotation> RotationTypeHandle;
    [ReadOnly] public ComponentTypeHandle<RotationSpeed> RotationSpeedTypeHandle;

    public void Execute(ArchetypeChunk chunk, int chunkIndex, int firstEntityIndex)
    {
        // ...
    }
}
```

system在`OnUpdate()`为函数中的这些变量分配值。ECS 在运行job时会使用`Execute()`方法内部的变量。

该job还使用Unity delta时间为3D对象的旋转设置动画。该示例使用struct字段将此值传递给`Execute()`方法。



## 编写Execute方法

IJobChunk `Execute()`方法的签名为：

```cs
public void Execute(ArchetypeChunk chunk, int chunkIndex, int firstEntityIndex)
```

`chunk`参数是内存块的句柄，该内存块包含此job的迭代必须处理的entities和components。因为chunk只能包含一个archetype，所以chunk中的所有entities都具有相同的component集。

使用`chunk`参数获取component的NativeArray实例：

```cs
var chunkRotations = chunk.GetNativeArray(RotationTypeHandle);
var chunkRotationSpeeds = chunk.GetNativeArray(RotationSpeedTypeHandle);
```

这些数组是对齐的，以便entities在所有数组中具有相同的索引。然后，您可以使用正常的for循环来遍历components数组。使用`chunk.Count`得到存储在当前chunk的entities的数量：

```cs
var chunkRotations = chunk.GetNativeArray(RotationTypeHandle);
var chunkRotationSpeeds = chunk.GetNativeArray(RotationSpeedTypeHandle);
for (var i = 0; i < chunk.Count; i++)
{
    var rotation = chunkRotations[i];
    var rotationSpeed = chunkRotationSpeeds[i];

    // Rotate something about its up vector at the speed given by RotationSpeed.
    // 根据RotationSpeed提供的速度围绕一个物体的z轴来旋转它
    chunkRotations[i] = new Rotation
    {
        Value = math.mul(math.normalize(rotation.Value),
            quaternion.AxisAngle(math.up(), rotationSpeed.RadiansPerSecond * DeltaTime))
    };
}
```

如果您在EntityQueryDesc中具有`Any`过滤器，或是完全没有在query中出现的可选components，则可以在使用之前使用该函数测试当前chunk是否包含这些`ArchetypeChunk.Has<T>()`组件之一：

```cs
if (chunk.Has<OptionalComp>(OptionalCompType))
{//...}
```

**注意：**如果使用并发形式的entity command buffer，请将`chunkIndex`参数作为`sortKey`参数传递给command buffer函数。



## 跳过那些内容都是未变化的entities的chunk

如果仅在components值更改后才需要更新entities，则可以将该component类型添加到EntityQuery的change filter中。例如，如果您的system读取两个components，并且仅在前两个components中的一个已更改时才需要更新第三个component，则可以按以下方式使用EntityQuery：

```cs
private EntityQuery m_Query;

protected override void OnCreate()
{
    m_Query = GetEntityQuery(
        ComponentType.ReadWrite<Output>(),
        ComponentType.ReadOnly<InputA>(),
        ComponentType.ReadOnly<InputB>());
    m_Query.SetChangedVersionFilter(
        new ComponentType[]
        {
            ComponentType.ReadWrite<InputA>(),
            ComponentType.ReadWrite<InputB>()
        });
}
```

EntityQuery的change filter最多支持两个componnets。如果您想进行更多检查或不使用EntityQuery，则可以手动进行检查。要进行此检查，请使用`ArchetypeChunk.DidChange()`函数将component的chunk的change version与system的`LastSystemVersion`进行比较。如果此函数返回false，则可以完全跳过当前chunk，因为自从上次system运行以来，该类型的component均未更改。

您必须使用一个struct字段将`LastSystemVersion`从system传递到job中，如下所示：

```cs
[BurstCompile]
struct UpdateJob : IJobChunk
{
    public ComponentTypeHandle<InputA> InputATypeHandle;
    public ComponentTypeHandle<InputB> InputBTypeHandle;
    [ReadOnly] public ComponentTypeHandle<Output> OutputTypeHandle;
    public uint LastSystemVersion;

    public void Execute(ArchetypeChunk chunk, int chunkIndex, int firstEntityIndex)
    {
        var inputAChanged = chunk.DidChange(InputATypeHandle, LastSystemVersion);
        var inputBChanged = chunk.DidChange(InputBTypeHandle, LastSystemVersion);

        // 如果没有component变化，就跳过当前chunk
        if (!(inputAChanged || inputBChanged))
            return;

        var inputAs = chunk.GetNativeArray(InputATypeHandle);
        var inputBs = chunk.GetNativeArray(InputBTypeHandle);
        var outputs = chunk.GetNativeArray(OutputTypeHandle);

        for (var i = 0; i < outputs.Length; i++)
        {
            outputs[i] = new Output { Value = inputAs[i].Value + inputBs[i].Value };
        }
    }
}
```

与所有job结构字段一样，在调度job之前，必须分配其值：

```cs
protected override void OnUpdate()
{
    var job = new UpdateJob();

    job.LastSystemVersion = this.LastSystemVersion;

    job.InputATypeHandle = GetComponentTypeHandle<InputA>(true);
    job.InputBTypeHandle = GetComponentTypeHandle<InputB>(true);
    job.OutputTypeHandle = GetComponentTypeHandle<Output>(false);

    this.Dependency = job.ScheduleParallel(m_Query, this.Dependency);
}
```

**注意：**为了提高效率，change version适用于整个chunk，而不是单个entity。如果另一个具有写入该类型component功能的job访问了chunk，则ECS会对该component的change version进行递增，并且`DidChange()`函数将返回true。即使声明对component进行写访问的job实际上并未更改component，ECS也会递增change version。



## 实例化并调度job

若要运行IJobChunk job，必须创建job结构的实例，设置结构字段，然后调度job。在SystemBase的`OnUpdate()`中执行此操作时，system会将每帧调度job。

```cs
protected override void OnUpdate()
{
    var job = new RotationSpeedJob()
    {
        RotationTypeHandle = GetComponentTypeHandle<Rotation>(false),
        RotationSpeedTypeHandle = GetComponentTypeHandle<RotationSpeed>(true),
        DeltaTime = Time.DeltaTime
    };
    this.Dependency =  job.ScheduleParallel(m_Query, this.Dependency);
}
```

调用`GetArchetypeChunkComponentType<T>()`函数设置component类型变量时，请确保将job读取但不写入的components的`isReadOnly`参数设置为true。正确设置这些参数可能会对ECS框架调度job的效率产生重大影响（害怕）。这些访问模式设置必须在结构定义和EntityQuery中都与它们的等效项匹配。

不要在system类的变量中缓存`GetArchetypeChunkComponentType<T>()`的返回值。您必须在每次system运行时调用该函数，并将更新后的值传递给job。

# 通过手动迭代创建Systems

您可以在NativeArray中显式请求所有chunks，并使用诸如`IJobParallelFor`的job来处理它们。如果适用于简化迭代EntityQuery中所有chunk的简化模型满足不了您的需求，则应使用此方法。以下是一个示例：

```cs
public class RotationSpeedSystem : SystemBase
{
   [BurstCompile]
   struct RotationSpeedJob : IJobParallelFor
   {
       [DeallocateOnJobCompletion] public NativeArray<ArchetypeChunk> Chunks;
       public ArchetypeChunkComponentType<RotationQuaternion> RotationType;
       [ReadOnly] public ArchetypeChunkComponentType<RotationSpeed> RotationSpeedType;
       public float DeltaTime;

       public void Execute(int chunkIndex)
       {
           var chunk = Chunks[chunkIndex];
           var chunkRotation = chunk.GetNativeArray(RotationType);
           var chunkSpeed = chunk.GetNativeArray(RotationSpeedType);
           var instanceCount = chunk.Count;

           for (int i = 0; i < instanceCount; i++)
           {
               var rotation = chunkRotation[i];
               var speed = chunkSpeed[i];
               rotation.Value = math.mul(math.normalize(rotation.Value), quaternion.AxisAngle(math.up(), speed.RadiansPerSecond * DeltaTime));
               chunkRotation[i] = rotation;
           }
       }
   }

   EntityQuery m_Query;   

   protected override void OnCreate()
   {
       var queryDesc = new EntityQueryDesc
       {
           All = new ComponentType[]{ typeof(RotationQuaternion), ComponentType.ReadOnly<RotationSpeed>() }
       };

       m_Query = GetEntityQuery(queryDesc);
   }

   protected override void OnUpdate()
   {
       var rotationType = GetArchetypeChunkComponentType<RotationQuaternion>();
       var rotationSpeedType = GetArchetypeChunkComponentType<RotationSpeed>(true);
       var chunks = m_Query.CreateArchetypeChunkArray(Allocator.TempJob);

       var rotationsSpeedJob = new RotationSpeedJob
       {
           Chunks = chunks,
           RotationType = rotationType,
           RotationSpeedType = rotationSpeedType,
           DeltaTime = Time.deltaTime
       };
       this.Dependency rotationsSpeedJob.Schedule(chunks.Length,32, this.Dependency);
   }
}
```

## 手动迭代

您可以使用EntityManager类手动遍历entities或chunk，尽管这不是最佳实践。您只应在测试或调试代码中（或仅在进行实验时）或在您拥有完全可控的实体集的独立world中使用这些迭代方法。

例如，以下代码段循环访问活动世界中的所有entities：

```cs
var entityManager = World.Active.EntityManager;
var allEntities = entityManager.GetAllEntities();
foreach (var entity in allEntities)
{
   //...
}
allEntities.Dispose();
```

此代码段循环遍历活动世界中的所有chunks：

```cs
var entityManager = World.Active.EntityManager;
var allChunks = entityManager.GetAllChunks();
foreach (var chunk in allChunks)
{
   //...
}
allChunks.Dispose();
```

# Systems更新顺序

使用Component System Groups来指定system的更新顺序。您可以使用system类声明中的[UpdateInGroup]attribute将system放在group中。然后，您可以使用[UpdateBefore]和[UpdateAfter]attribute来指定其在group中的更新顺序。

ECS框架会创建一组[default system groups](https://docs.unity3d.com/Packages/com.unity.entities@0.13/manual/system_update_order.html#default-system-groups)，可用于在框架的恰当阶段更新systems。您可以将一个group嵌套在另一个group中，以便group中的所有systems在恰当的阶段进行更新，并根据其在group中的顺序进行更新。



## Component System Groups

ComponentSystemGroup类表示应按特定顺序一起更新的相关Component Systems的列表。ComponentSystemGroup是从ComponentSystemBase派生的，因此在所有重要的方面它都像Component System一样工作-可以相对于其他systems进行排序，具有OnUpdate（）方法等。最重要的是，这意味着可以将Component System Group嵌套在其他Component System Group中，形成一个层次结构。

默认情况下，当调用ComponentSystemGroup的`Update()`方法时，它将在其成员system的排序列表中的每个system上调用Update（）。如果任何成员system本身就是system group，则它们将递归更新自己的成员。这种情况下生成的system顺序遵循树的深度优先遍历规则。



## System Ordering Attributes

现有的system ordering attribute会被保留，但语义和限制稍有不同。

- [UpdateInGroup]-指定此system应该是其成员的ComponentSystemGroup。如果省略此attribute，system将被自动添加到默认的World's SimulationSystemGroup（请参见下文）。
- [UpdateBefore]和[UpdateAfter]-是相对于其他systems的system ordering。为这些attribute指定的system类型必须是同一组的成员。跨组的排序是在包含两个系统的适当的最深组中进行的：
  - **例如：**如果SystemA在GroupA中，而SystemB在GroupB中，并且GroupA和GroupB都是GroupC的成员，则GroupA和GroupB的顺序将隐式确定SystemA和SystemB的相对顺序；无需对system进行明确排序。
- [DisableAutoCreation]-防止在默认World初始化期间创建system。您必须显式创建和更新system。但是，您可以将带有此attribute的system添加到ComponentSystemGroup的更新列表中，然后它将像该列表中的其他systems一样自动进行更新。



## Default System Groups

默认的World包含ComponentSystemGroup实例的层次结构。只有三个根级别的system groups被添加到Unity Player循环（以下列表还显示了每个group中的预定义成员systems）：

- InitializationSystemGroup（在Initialization播放器循环阶段的末尾更新）
  - BeginInitializationEntityCommandBufferSystem
  - CopyInitialTransformFromGameObjectSystem
  - SubSceneLiveLinkSystem
  - SubSceneStreamingSystem
  - EndInitializationEntityCommandBufferSystem
- SimulationSystemGroup（在Update播放器循环阶段的末尾更新）
  - BeginSimulationEntityCommandBufferSystem
  - TransformSystemGroup
    - EndFrameParentSystem
    - CopyTransformFromGameObjectSystem
    - EndFrameTRSToLocalToWorldSystem
    - EndFrameTRSToLocalToParentSystem
    - EndFrameLocalToParentSystem
    - CopyTransformToGameObjectSystem
  - LateSimulationSystemGroup
  - EndSimulationEntityCommandBufferSystem
- PresentationSystemGroup（在PreLateUpdate播放器循环阶段的末尾更新）
  - BeginPresentationEntityCommandBufferSystem
  - CreateMissingRenderBoundsFromMeshRenderer
  - RenderingSystemBootstrap
  - RenderBoundsUpdateSystem
  - RenderMeshSystem
  - LODGroupSystemV1
  - LodRequirementsUpdateSystem
  - EndPresentationEntityCommandBufferSystem

请注意，此列表的具体内容可能会更改（经典迭代）。

## 多个Worlds

除了（或代替上述）默认World，您可以创建多个World。同一component system类可以在多个world中实例化，并且每个实例可以在更新顺序的不同点以不同的速率进行更新。

当前还无法指定World中的每个system机进行手动更新。但是，您可以控制在哪个World中创建哪些systems，以及应将其添加到哪些现有system groups中。比如，自定义WorldB可以实例化SystemX和SystemY，将SystemX添加到默认的World's SimulationSystemGroup，并将SystemY添加到默认的World's PresentationSystemGroup。这些systems可以像往常一样相对于其group同级对其进行排序，并将与相应的groups一起进行更新。

为了支持此种情况，现在提供了新的ICustomBootstrap接口：

```cs
public interface ICustomBootstrap
{
    // Returns the systems which should be handled by the default bootstrap process.
    // If null is returned the default world will not be created at all.
    // Empty list creates default world and entrypoints
    // 返回那些需要被默认启动程序处理的systems
    // 如果为null，默认的world都不会被创建
    // 返回0个元素的list创建默认的world和入口点
    List<Type> Initialize(List<Type> systems);
}
```

当实现此接口时，component system类型的完整列表将在默认世界初始化之前传递给classes `Initialize()`方法。自定义的引导程序可以遍历此列表，并在所需的任何World中创建systems。您可以从Initialize（）方法返回systems列表，它们将作为常规的默认world初始化的一部分创建。

例如，以下是自定义`MyCustomBootstrap.Initialize()`实现的典型过程：

1. 创建任何其他Worlds及其顶层ComponentSystemGroups。
2. 对于system类型列表中的每个类型：
   1. 向上遍历ComponentSystemGroup层次结构以找到此system Type的顶级group。
   2. 如果它是在步骤1中创建的groups之一，请在该world中创建system，然后使用`group.AddSystemToUpdateList()`将其添加到层次结构中。
   3. 如果不是，请将此类型附加到列表以返回到DefaultWorldInitialization。
3. 在新的顶级组上调用group.SortSystemUpdateList（）。
   1. （可选）将它们添加到默认世界组之一
4. 将未处理systems的列表返回给DefaultWorldInitialization。

**注意：** ECS框架通过反射查找您的ICustomBootstrap实现。

## 提示和最佳实践

- **使用[UpdateInGroup]为您编写的每个system指定system group。**如果未指定，则隐式默认组为SimulationSystemGroup。
- **使用手动选定的ComponentSystemGroups来更新Unity播放器循环中其他位置的system。**将[DisableAutoCreation]属性添加到component system（或system group）可防止将其创建或添加到默认system group。您仍然可以使用World.GetOrCreateSystem手动创建system并通过从主线程手动调用MySystem.Update（）进行更新。这是在Unity Player循环中的其他位置插入system的简便方法（例如，如果您的system应在框架中的更早或更晚的时候运行）。
- **如果可能的话，请使用现有的EntityCommandBufferSystem而不是添加新的。**An `EntityCommandBufferSystem`代表一个sync point，在该sync point，主线程在处理任何未完成的`EntityCommandBuffer`s 之前等待工作线程完成。与创建新的“气泡”（这个东西暂时没想好怎么翻译，应该和冒泡的事件传递机制一个意思）相比，在每个顶级system group中重用预定义的Begin / End system之一不太可能在帧管线中引入新的“气泡”。
- **避免在`ComponentSystemGroup.OnUpdate()`中加入自定义逻辑**。由于从`ComponentSystemGroup`功能上来说本身就是一个component system，因此可能很想在其OnUpdate（）方法中添加自定义处理，执行一些工作，生成一些工作等。我们通常建议不要这样做，因为从外部尚不清楚自定义逻辑是在更新组成员之前或之后执行。最好将system group限制为一种分组机制，并在相对于该组显式排序的单独的component system中实现所需的逻辑。

# Job dependencies

Unity根据system读取和写入的ECS component分析每个system的数据依赖性。如果在框架中较早更新的system读取了较新system写入的数据，或写入了较新system读取的数据，则第二个system将依赖于第一个system。为避免出现[竞争状况](https://en.wikipedia.org/wiki/Race_condition)，job调度程序确保在运行system job之前，system所依赖的所有jobs均已完成。

system的[Dependency](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Dependency)属性是[JobHandle](https://docs.unity3d.com/ScriptReference/Unity.Jobs.JobHandle.html)，代表与system的ECS相关的依赖关系。在[OnUpdate（）](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_OnUpdate_)之前，[Dependency](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Dependency)属性反映了system对先前job的传入依赖关系。默认情况下，system根据您在system中调度job时读取和写入的component来更新[Dependency](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Dependency)属性。

要覆盖此默认行为，请使用[Entities.ForEach](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Entities)和[Job.WithCode](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Job)的重载版本，这些重载版本将job依赖项作为参数，并将更新后的依赖项作为[JobHandle返回](https://docs.unity3d.com/ScriptReference/Unity.Jobs.JobHandle.html)。使用这些构造的显式版本时，ECS不会自动将job handles与system的[Dependency](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Dependency)属性结合在一起。您必须在需要时手动组合它们。

请注意，system的[Dependency](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Dependency)属性不会跟踪job对通过[NativeArrays](https://docs.unity3d.com/ScriptReference/Unity.Collections.NativeArray_1.html)或其他类似容器传递的数据可能具有的依赖关系。如果您在一个job中编写NativeArray并在另一个job中读取该数组，则必须手动添加第一个job的JobHandle作为第二个job的依赖项（通常使用[JobHandle.CombineDependencies](https://docs.unity3d.com/ScriptReference/Unity.Jobs.JobHandle.CombineDependencies.html)）。

当您调用[Entities.ForEach.Run（）时](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Entities)，作业调度程序会在开始ForEach迭代之前完成system所依赖的所有调度job。如果您还使用[WithStructuralChanges（）](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Entities)作为构造的一部分，则job调度程序将完成所有正在运行和待调度的jobs。结构更改还会使对component数据的任何直接引用无效。有关更多信息，请参见[Sync Point](https://docs.unity3d.com/Packages/com.unity.entities@0.13/manual/sync_points.html)。

有关更多信息，请参见[JobHandle和依赖项](https://docs.unity3d.com/Manual/JobSystemJobDependencies.html)。

# 查找数据

访问和修改ECS数据的最有效方法是使用带有实体查询和作业的系统。这样可以以最少的内存高速缓存未命中来最佳利用CPU资源。实际上，数据设计的目标之一应该是使用最有效，最快的路径来执行大部分数据转换。但是，有时您需要在程序的任意位置访问任意实体的任意组件。

给定一个Entity对象，您可以在其[IComponentData](https://docs.unity3d.com/Packages/com.unity.entities@0.13/manual/component_data.html)和[动态缓冲区中](https://docs.unity3d.com/Packages/com.unity.entities@0.13/manual/dynamic_buffers.html)查找数据。该方法根据您的代码是在系统中使用[Entities.ForEach](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Entities)还是使用[IJobChunk](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.IJobChunk.html)作业还是在主线程上的其他位置执行而有所不同。

## 在Systems中查找entities数据

使用[GetComponent （Entity）](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_GetComponent__1_Unity_Entities_Entity_)从system的[Entities.ForEach](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Entities)或[Job.WithCode]函数内部查找存储在任意entities components中的数据。

例如，如果您的“目标”component的“实体”字段定义了目标entity，则可以使用以下代码将entity向其目标旋转：

```cs
public class TrackingSystem : SystemBase
{
    protected override void OnUpdate()
    {
        float deltaTime = this.Time.DeltaTime;

        Entities
            .ForEach((ref Rotation orientation,
            in LocalToWorld transform,
            in Target target) =>
            {
                // Check to make sure the target Entity still exists and has
                // the needed component
                // 确保目标entity仍然存在，并且拥有我们需要的component
                if (!HasComponent<LocalToWorld>(target.entity))
                    return;

                // Look up the entity data
               	// 查找entity数据
                LocalToWorld targetTransform
                    = GetComponent<LocalToWorld>(target.entity);
                float3 targetPosition = targetTransform.Position;

                // Calculate the rotation
                // 计算旋转四元数
                float3 displacement = targetPosition - transform.Position;
                float3 upReference = new float3(0, 1, 0);
                quaternion lookRotation =
                    quaternion.LookRotationSafe(displacement, upReference);

                orientation.Value =
                    math.slerp(orientation.Value, lookRotation, deltaTime);
            })
            .ScheduleParallel();
    }
}
```

访问存储在[dynamic buffers](https://docs.unity3d.com/Packages/com.unity.entities@0.13/manual/dynamic_buffers.html)中的数据需要额外的步骤。您必须在[OnUpdate（）](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_OnUpdate_)方法中声明[BufferFromEntity](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.BufferFromEntity-1.html)类型的局部变量。然后，您可以在lambda函数中“捕获”局部变量。

```cs
public struct BufferData : IBufferElementData
{
    public float Value;
}
public class BufferLookupSystem : SystemBase
{
    protected override void OnUpdate()
    {
        BufferFromEntity<BufferData> buffersOfAllEntities
            = this.GetBufferFromEntity<BufferData>(true);

        Entities
            .ForEach((ref Rotation orientation,
            in LocalToWorld transform,
            in Target target) =>
            {
                // Check to make sure the target Entity with this buffer type still exists
                // 确保缓冲区中的目标entity仍然存在
                if (!buffersOfAllEntities.HasComponent(target.entity))
                    return;

                // Get a reference to the buffer
                // 获取缓冲区引用
                DynamicBuffer<BufferData> bufferOfOneEntity =
                    buffersOfAllEntities[target.entity];

                // Use the data in the buffer
                // 使用缓冲区的数据
                float avg = 0;
                for (var i = 0; i < bufferOfOneEntity.Length; i++)
                {
                    avg += bufferOfOneEntity[i].Value;
                }
                if (bufferOfOneEntity.Length > 0)
                    avg /= bufferOfOneEntity.Length;
            })
            .ScheduleParallel();
    }
}
```

## 在IJobChunk中查找entity数据

要随机访问IJobChunk或其他job结构中的component数据，请使用以下类型之一来获取component的类似于数组的接口，并由[Entity](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.Entity.html)对象索引：

- [ComponentDataFromEntity](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.ComponentDataFromEntity-1.html)
- [BufferFromEntity](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.BufferFromEntity-1.html)

声明类型为[ComponentDataFromEntity](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.ComponentDataFromEntity-1.html)或[BufferFromEntity](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.BufferFromEntity-1.html)的字段，并在调度job之前设置该字段的值。

例如，如果您的“目标”component的“实体”字段定义了目标entity，则可以将以下字段添加到job结构中以查找目标的世界位置：

```cs
[ReadOnly]
public ComponentDataFromEntity<LocalToWorld> EntityPositions;
```

请注意，此声明使用[ReadOnly](https://docs.unity3d.com/ScriptReference/Unity.Collections.ReadOnlyAttribute.html)属性。您应该始终声明ComponentDataFromEntity 除非您确实写入要访问的component，否则对象为只读。



您可以在调度job时按以下方式设置此字段：

```cs
var job = new ChaserSystemJob();
job.EntityPositions = this.GetComponentDataFromEntity<LocalToWorld>(true);
```

在job的`Execute()`函数内，您可以使用Entity对象查找component的值：

```cs
float3 targetPosition = EntityPositions[targetEntity].Position;
```

以下完整示例显示了一个system，该system将具有包含其目标的Entity对象的Target字段的entity移向目标的当前位置：

```cs
public class MoveTowardsEntitySystem : SystemBase
{
    private EntityQuery query;

    [BurstCompile]
    private struct MoveTowardsJob : IJobChunk
    {
        // Read-write data in the current chunk
        // 当前chunk的可读写数据
        public ComponentTypeHandle<Translation> PositionTypeHandleAccessor;

        // Read-only data in the current chunk
        // 当前chunk的只读数据
        [ReadOnly]
        public ComponentTypeHandle<Target> TargetTypeHandleAccessor;

        // Read-only data stored (potentially) in other chunks
        // 被其他chunk存储的只读数据（潜在的）
        [ReadOnly]
        public ComponentDataFromEntity<LocalToWorld> EntityPositions;

        // Non-entity data
        // 非entity数据
        public float deltaTime;

        public void Execute(ArchetypeChunk chunk,
            int chunkIndex,
            int firstEntityIndex)
        {
            // Get arrays of the components in chunk
            // 获取chunk中的组件数组
            NativeArray<Translation> positions
                = chunk.GetNativeArray<Translation>(PositionTypeHandleAccessor);
            NativeArray<Target> targets
                = chunk.GetNativeArray<Target>(TargetTypeHandleAccessor);

            for (int i = 0; i < positions.Length; i++)
            {
                // Get the target Entity object
                // 获取目标实体对象
                Entity targetEntity = targets[i].entity;

                // Check that the target still exists
                // 检查entity是否存在
                if (!EntityPositions.HasComponent(targetEntity))
                    continue;

                // Update translation to move the chasing enitity toward the target
                // 更新transform以将实体移向目标
                float3 targetPosition = EntityPositions[targetEntity].Position;
                float3 chaserPosition = positions[i].Value;

                float3 displacement = targetPosition - chaserPosition;
                positions[i] = new Translation
                {
                    Value = chaserPosition + displacement * deltaTime
                };
            }
        }
    }

    protected override void OnCreate()
    {
        // Select all entities that have Translation and Target Componentx
        // 一个获取所有拥有transform和componentx entities的query
        query = this.GetEntityQuery
            (
                typeof(Translation),
                ComponentType.ReadOnly<Target>()
            );
    }

    protected override void OnUpdate()
    {
        // Create the job
        // 创建job
        var job = new MoveTowardsJob();

        // Set the chunk data accessors
        // 设置chunk数据获取者
        job.PositionTypeHandleAccessor =
            this.GetComponentTypeHandle<Translation>(false);
        job.TargetTypeHandleAccessor =
            this.GetComponentTypeHandle<Target>(true);

        // Set the component data lookup field
        // 设置component data的lookup字段
        job.EntityPositions = this.GetComponentDataFromEntity<LocalToWorld>(true);

        // Set non-ECS data fields
        // 设置非ECS数据
        job.deltaTime = this.Time.DeltaTime;

        // Schedule the job using Dependency property
        // 使用Dependency property调度job
        this.Dependency = job.Schedule(query, this.Dependency);
    }
}
```

## 获取数据失败

如果您正在查找的数据与您直接在job中读取和写入的数据冲突，则随机访问会导致竞争状况和BUG。如果确定直接在job中读取或写入的特定entity数据与您随机读取或写入的特定entity数据之间没有重叠，则可以使用[NativeDisableParallelForRestriction](https://docs.unity3d.com/ScriptReference/Unity.Collections.NativeDisableParallelForRestrictionAttribute.html) attribute标记访问器对象。

# Entity Command Buffers

[`EntityCommandBuffer`](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.EntityCommandBuffer.html)（ECB）解决两个重要问题：

1. 在job中，您无法访问[`EntityManager`](https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.EntityManager.html)。
2. 当执行[structural change](https://docs.unity3d.com/Packages/com.unity.entities@0.13/manual/sync_points.html)（如创建entity）时，您将创建一个[Sync Point](https://docs.unity3d.com/Packages/com.unity.entities@0.13/manual/sync_points.html)并且必须等待所有jobs完成。

`EntityCommandBuffer`允许你将变动队列化（无论是从job或从主线程），使他们能够在主线程上后生效。

## Entity command buffer systems

使您可以在一帧中明确定义的位置播放在ECB中排队的命令。这些system通常是使用ECB的最佳方法。您可以从同一entity Entity command buffer systems中获取多个ECB，并且system将按照更新时创建它们的顺序来播放所有ECB。这将在system更新时创建一个Sync Point，而不是每个ECB一个Sync Point，并确保确定性。

默认的World初始化提供了三个system group，分别用于初始化，模拟和执行，并按每帧的顺序进行更新。在一个组中，有一个Entity command buffer system在该组中的任何其他system之前运行，而另一个在该组中的所有其他system之后运行。最好，您应该使用现有的Entity command buffer systemss之一，而不是创建自己的Entity command buffer systems，以最大程度地减少Sync point。有关[default groups](https://docs.unity3d.com/Packages/com.unity.entities@0.13/manual/system_update_order.html)和Entity command buffer systems的内容，请参见[default system group](https://docs.unity3d.com/Packages/com.unity.entities@0.13/manual/system_update_order.html)。

如果要使用并行job中的ECB（例如，`Entities.ForEach`中的），则必须确保首先通过调用`ToConcurrent`将其转换为并发ECB 。为确保ECB中命令的顺序不取决于job在job之间的分配方式，还必须将当前query中entities的索引传递给每个操作。

您可以像这样获取和使用ECB：

```cs
struct Lifetime : IComponentData
{
    public byte Value;
}

class LifetimeSystem : SystemBase
{
    EndSimulationEntityCommandBufferSystem m_EndSimulationEcbSystem;
    protected override void OnCreate()
    {
        base.OnCreate();
        // Find the ECB system once and store it for later usage
        // 获取ECB system一次，并保存以供后续使用
        m_EndSimulationEcbSystem = World
            .GetOrCreateSystem<EndSimulationEntityCommandBufferSystem>();
    }

    protected override void OnUpdate()
    {
        // Acquire an ECB and convert it to a concurrent one to be able
        // to use it from a parallel job.
        // 获取一个ECB，并将其转换成一个并行的来在并行job中使用它
        var ecb = m_EndSimulationEcbSystem.CreateCommandBuffer().AsParallelWriter();
        Entities
            .ForEach((Entity entity, int entityInQueryIndex, ref Lifetime lifetime) =>
        {
            // Track the lifetime of an entity and destroy it once
            // the lifetime reaches zero
            // 追踪一个entity的生命时长，并在目标时间销毁他
            if (lifetime.Value == 0)
            {
                // pass the entityInQueryIndex to the operation so
                // the ECB can play back the commands in the right
                // order
                // 将entityInQueryIndex传递给操作，这样ECB可以晚些时候进行调用
                ecb.DestroyEntity(entityInQueryIndex, entity);
            }
            else
            {
                lifetime.Value -= 1;
            }
        }).ScheduleParallel();

        // Make sure that the ECB system knows about our job
        // 确保ECB system知道我们这个job
        m_EndSimulationEcbSystem.AddJobHandleForProducer(this.Dependency);
    }
}
```
