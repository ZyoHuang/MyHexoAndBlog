---
title: Unity DOTS：Entities部分
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

# Entities
Entities是实体组件系统体系结构的三个主要元素之一。它们代表游戏或应用程序中的各个“事物”。`Entities既没有行为也没有数据；取而代之的是，它担任索引各种数据的职责。Systems提供行为，而Components存储数据。`

entity本质上是一个ID。最简单方法是把它作为一个超轻量级GameObject甚至没有名称。实体ID稳定；您可以使用它们来存储对另一个component或entities的引用。例如，Hierarchy中的子entity可能需要引用其父entity。

`一个EntityManager管理在一个World中所有的entities。`EntityManager维护entities列表并组织与entities相关联的数据以实现最佳性能。

尽管实体没有类型，但是可以按与entities相关联的components的类型对其进行分组。创建entities并向其添加components时，EntityManager会持续跟踪监控entities上components的唯一组合。这种独特的组合称为`Archetype（原型）`。`将components添加到entities时（或者从entities移除时），EntityManager会创建EntityArchetype struct（或者将entities转移到已有的符合其形式的EntityArchetype中）`。您可以使用现有EntityArchetype来创建符合该Archetype的新entities。您也可以预先创建一个EntityArchetype，然后使用它来创建entities。

## 创建实体
创建实体的最简单方法是在Unity编辑器中(啊这)。您可以用ECS在运行时将放置在场景中的GameObject和Prefabs转换为entities。对于游戏或应用程序中更灵活的部分，您可以创建spawning system，以在一个job中创建多个entities。最后，您可以使用EntityManager.CreateEntity函数创建一个实体。

### 使用EntityManager创建实体
使用EntityManager.CreateEntity函数创建一个实体。ECS在与EntityManager相同的world中创建entities。

您可以通过以下方式一个接一个的创建实体：

- 使用ComponentType对象数组创建一个包含多个component的entity。
- 使用一个EntityArchetype创建一个包含多个component的entity。
- 使用Instantiate复制现有entity，包括其当前数据。
- 创建一个没有components的entity，然后向其添加components。（您可以立即添加components，也可以在需要其他components时添加。）

您也`可以一次创建多个实体`：

- CreateEntity：使用具有相同archetype的新entities填充NativeArray。
- Instantiate：通过现有entities的副本，包括其当前数据来填充NativeArray。
- CreateChunk：显式创建由指定数量的并给定archetype的entities填充的chunks。

## 添加和删除组件
创建实体后，您可以添加或删除components。当您执行此操作时，`受影响entities的archetype会更改，并且EntityManager必须将更改后的数据移动到新的memory chunk中，并对原始chunk中的component数组进行压缩`。

`导致结构改变的entities操作（即添加或删除更改了SharedComponentData值的组件以及破坏该实体）无法在job内部进行，因为这些操作可能会使该job正在处理的数据无效。相反，您可以将进行这类更改的命令添加到EntityCommandBuffer，并在job完成后执行此命令缓冲区。`

EntityManager提供的功能可从单个实体以及NativeArray中的所有实体中删除组件。有关更多信息，请参见Components上的文档。

## 遍历entities
遍历具有一组匹配compoennts的所有entities，是ECS体系结构的核心。请参阅访问entity数据。

## 使用EntityQuery查询数据
要读取或写入数据，您必须首先找到想要更改的数据。ECS中的数据存储在components中，ECS根据它们所属entity的archetype在内存中进行分组。您可以使用EntityQuery获取ECS数据的视图，该视图仅包含给定算法或流程所需的特定数据。

您可以使用EntityQuery执行以下操作：

- 运行job以处理为视图选择的entities和components
- 获取一个包含所有目标entities的NativeArray
- 获取所选compoennts的NativeArrays（按component type）

EntityQuery返回的entitie和components数组保证是“并行”的，也就是说，`相同的索引值始终会应用于任何数组中的相同entity`。

注：SystemBase.Entities.ForEach结构基于components type和属性为这些API创建指定的内部EntityQuery实例。您不能将其他EntityQuery对象与Entities.ForEach一起使用（但是您可以获得Entities.ForEach实例构造的查询对象并在其他地方使用它）。

### 定义Query
`EntityQuery需要指定包含在一个archetype里的一组component type，您也可以排除特定类型组件的archetype。`

对于简单查询，您可以基于一系列component types创建EntityQuery。以下示例定义了一个EntityQuery，该查询查找具有RotationQuaternion和RotationSpeed组件的所有实体。

```csharp
EntityQuery m_Query = GetEntityQuery(typeof(RotationQuaternion), 
    ComponentType.ReadOnly<RotationSpeed>());
```
该查询使用`ComponentType.ReadOnly<T>`而不是更简单的typeof表达式来表示该system不写入RotationSpeed。尽可能始终指定只读，因为对数据的读取访问限制较少，这可以帮助job scheduler程序更有效地执行作job。

#### EntityQueryDesc
对于更复杂的查询，可以使用EntityQueryDesc对象创建EntityQuery。EntityQueryDesc提供了一种灵活的查询机制，可以根据以下几组组件指定要选择的原型：

- All：此数组中的所有组件类型必须存在于原型中
- Any：原型中必须至少存在此数组中的一种组件类型
- None：原型中不能存在此数组中的任何组件类型

例如，以下查询包括包含RotationQuaternion和RotationSpeed组件的archetype，但会排除Frozen组件的任何archetype：

```csharp
var query = new EntityQueryDesc
{
   None = new ComponentType[]{ typeof(Frozen) },
   All = new ComponentType[]{ typeof(RotationQuaternion), 
       ComponentType.ReadOnly<RotationSpeed>() }
}
EntityQuery m_Query = GetEntityQuery(query);
```

注意：不要在EntityQueryDesc中包括可选组件。要处理可选组件，请使用`ArchetypeChunk.Has<T>()`方法确定chunk中是否包含可选组件。`因为同一chunk中的所有entities具有相同的组件，所以您只需要检查一个可选组件是否存在于每个chunk：而不是存在于每个实体。`

#### Query options
创建EntityQueryDesc时，可以设置其Options变量。这些选项允许进行专门的查询（通常不需要设置它们）：

- Default：未设置任何选项。一般的查询行为。
- IncludePrefab：包括那些包含特殊的Prefab标签component的archetypes。
- IncludeDisabled：包括那些包含特殊的Disabled标签component的archetypes。
- FilterWriteGroup：考虑query中任何component的WriteGroup。

设置FilterWriteGroup选项时，`视图中仅包含明确在query中指定的WriteGroup中的components的entities。ECS将会排除具有WriteGroup中任何其他components的entities。`

在以下示例中，C2和C3是基于C1的同一WriteGroup中的components，并且此query使用了需要C1和C3，以及FilterWriteGroup选项：

```csharp
public struct C1: IComponentData{}

[WriteGroup(C1)]
public struct C2: IComponentData{}

[WriteGroup(C1)]
public struct C3: IComponentData{}

// ... In a system:
var query = new EntityQueryDesc{
    All = new ComponentType{typeof(C1), ComponentType.ReadOnly<C3>()},
    Options = EntityQueryDescOptions.FilterWriteGroup
};
var m_Query = GetEntityQuery(query);
```
此query将排除同时具有C2和C3的所有entities，因为query中未显式包含C2。尽管您可以使用None来设计query中的内容，但通过WriteGroup来进行操作有一个重要的好处：您无需更改其他systems使用的query（只要这些systems也使用WriteGroup）。

`WriteGroup是一种可用于扩展现有system（systemA）的机制。例如，如果C1和C2是在另一个systemB中定义的（也许是您无法控制的库的一部分），则可以将C3与C2放在同一WriteGroup中来更改C1的更新方式。对于添加了C3 component的任何entity，systemB会更新C1，而systemA不会更新C1（因为write group的缘故被过滤了）。对于其他没有C3的entity，systemA将像以前一样更新C1。`

更多WriteGroup相关：[WriteGroup](https://docs.unity3d.com/Packages/com.unity.entities@0.11/manual/ecs_write_groups.html "WriteGroup")

想要更加详细的了解WriteGroup内容，请参照本系列文章的《Unity DOTS：ECS拓展内容》的WriteGroup部分

#### 组合queries
要组合多个查询，可以传递EntityQueryDesc对象的数组而不是单个实例。您必须使用逻辑或运算来组合每个查询。以下示例选择包含RotationQuaternion组件或RotationSpeed组件（或两者）的任何原型：

```csharp
var query0 = new EntityQueryDesc
{
   All = new ComponentType[] {typeof(RotationQuaternion)}
};

var query1 = new EntityQueryDesc
{
   All = new ComponentType[] {typeof(RotationSpeed)}
};

EntityQuery m_Query 
    = GetEntityQuery(new EntityQueryDesc[] {query0, query1});
```
### 创建一个EntityQuery
在system类之外，可以使用以下EntityManager.CreateEntityQuery()功能创建EntityQuery ：

```csharp
EntityQuery m_Query = CreateEntityQuery(typeof(RotationQuaternion), 
    ComponentType.ReadOnly<RotationSpeed>());
```
但是，在system类中，必须将GetEntityQuery()函数用于IJobChunk job：

```csharp
public class RotationSpeedSystem : SystemBase
{
   private EntityQuery m_Query;
   protected override void OnCreate()
   {
       m_Query = GetEntityQuery(typeof(RotationQuaternion), 
           ComponentType.ReadOnly<RotationSpeed>());
   }
   //…
}
```
如果您打算重复使用同一视图，请缓存EntityQuery实例，而不是为每次使用创建一个新视图。例如，在系统中，您可以在系统OnCreate()功能中创建EntityQuery 并将结果存储在实例变量中。上例中的m_Query变量用于此目的。

请注意，为system创建的查询由system缓存。`对于GetEntityQuery()，如果已经存在queryA，则返回queryA，而不是创建一个新query`。但是，在对比两个查询是否相同时将不考虑filter设置。另外，如果您在查询中设置filter，则下次使用访问相同的查询时，GetEntityQuery()将设置相同的filter。使用ResetFilter()清除现有的filter。

### 定义filter
您可以过滤视图以及定义必须包含在query中或从query中排除的compoennts。您可以指定以下类型的filter：

- Shared component filter：根据共享组件的特定值过滤实体集。
- Change filter：根据特定组件类型的值是否已更改来过滤实体集。

设置的过滤器将保持有效，直到您调用ResetFilter()查询对象。

#### Shared component filters
要使用shared component筛选器，请将shared component（与其他所需component一起）包含在EntityQuery中，然后调用该SetFilter()函数。然后传入具有相同ISharedComponent类型的结构，其中包含要选择的值。所有值必须匹配。您最多可以向filter添加两个不同的shared component。

您可以随时更改filter，但是如果您更改filter，它不会更改从ToComponentDataArray()或ToEntityArray()函数接收到的任何现有实体或组件数组。您必须重新创建这些数组。

以下示例定义了一个名为SharedGrouping的shared component和一个处理所有包含他的entities的system，这个system会把shared component的Group字段设置为1.

```csharp
struct SharedGrouping : ISharedComponentData
{
    public int Group;
}

class ImpulseSystem : SystemBase
{
    EntityQuery m_Query;

    protected override void OnCreate(int capacity)
    {
        m_Query = GetEntityQuery(typeof(Position), 
            typeof(Displacement), 
            typeof(SharedGrouping));
    }

    protected override void OnUpdate()
    {
        // Only iterate over entities that have the SharedGrouping data set to 1
        // 只遍历那些把SharedGrouping data设置为1的entities
        m_Query.SetFilter(new SharedGrouping { Group = 1 });

        var positions = m_Query.ToComponentDataArray<Position>(Allocator.Temp);
        var displacememnts = m_Query.ToComponentDataArray<Displacement>(Allocator.Temp);

        for (int i = 0; i != positions.Length; i++)
            positions[i].Value = positions[i].Value + displacememnts[i].Value;
    }
}
```
#### 更换filters
如果仅在一个component值更改后才更新实体，则可以使用SetFilterChanged()函数将该component添加到EntityQuery过滤器中。例如，以下EntityQuery仅包括已经被另一个system写入Translation component的chunk的entity：

```csharp
protected override void OnCreate(int capacity)
{
    m_Query = GetEntityQuery(typeof(LocalToWorld), 
        ComponentType.ReadOnly<Translation>());
    m_Query.SetFilterChanged(typeof(Translation));
}
```
注意：`为了提高效率，更改filters会应用于整个chunk，而不应用于单个entity。`更改filters还检查system是否已执行了对component进行写访问，而不检查它是否实际上更改了任何数据。换句话说，如果另一个具有写入该类型component能力job业访问该chunk，则更改filters将影响该块中的所有实体。`这就是为什么您应该始终声明对不需要修改的组件的只读访问权限的原因`。

### 执行query
当您在job中使用EntityQuery或调用其中一个EntityQuery方法来返回视图中的entities，components或chunks的数组时，EntityQuery执行其查询：

- ToEntityArray() 返回所选entities的数组。
- `ToComponentDataArray<T>`返回T指定tnities类型的components的数组。
- CreateArchetypeChunkArray()返回包含所选实体的所有chunk。因为查询操作的是archetypes，sharedComponent值和更改filters，这些对于chunk中的所有entities都，所以在返回的chunk中存储的实体集与ToEntityArray()返回的实体集完全相同。

### 在jobs中
在计划IJobChunk job的system中，将EntityQuery对象传递给job的 ScheduleParallel()或ScheduleSingle()方法。在下面的示例中，该m_Query参数是EntityQuery对象

```csharp
// OnUpdate runs on the main thread.
// OnUpdate运行在主线程
protected override void OnUpdate()
{
    var rotationType 
        = GetArchetypeChunkComponentType<Rotation>(false); 
    var rotationSpeedType 
        = GetArchetypeChunkComponentType<RotationSpeed>(true);

    var job = new RotationSpeedJob()
    {
        RotationType = rotationType,
        RotationSpeedType = rotationSpeedType,
        DeltaTime = Time.deltaTime
    };

    return job.ScheduleParallel(m_Query, this.Dependency);
}
```

EntityQuery在内部使用job来创建所需的数组。将组传递给其中一种Schedule()方法时，ECS会调度EntityQuery的job以及system自己的job，因此您可以利用并行处理。

## World
世界同时拥有EntityManager和一组ComponentSystems。您可以根据需要创建任意多个World对象。通常，您可以创建一个模拟世界以及一个渲染或演示世界。

默认情况下，进入“Play”时会创建一个“World”，并在项目中填充所有可用ComponentSystem对象。但是，您可以禁用默认的World创建，并通过以下全局定义将其替换为您自己的代码：

- #UNITY_DISABLE_AUTOMATIC_SYSTEM_BOOTSTRAP_RUNTIME_WORLD 禁用默认运行时世界的生成。
- #UNITY_DISABLE_AUTOMATIC_SYSTEM_BOOTSTRAP_EDITOR_WORLD 禁用默认编辑器世界的生成。
- #UNITY_DISABLE_AUTOMATIC_SYSTEM_BOOTSTRAP 禁用两个默认世界的生成。
