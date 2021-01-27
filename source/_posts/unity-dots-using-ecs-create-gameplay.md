---
title: Unity DOTS：使用ECS进行GamePlay开发
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
cover: https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200817093035244.png
aplayer:
---
<meta name="referrer" content="no-referrer" />

本节包含有关如何在Unity Editor中创建基于DOTS的游戏和其他应用程序的信息。它还涵盖了ECS提供的可帮助您实现游戏功能的systems和components。

该systems包括：

- [Unity.Transforms](https://docs.unity3d.com/Packages/com.unity.entities@0.14/api/Unity.Transforms.html)：提供用于定义世界空间变换，3D对象层次结构以及管理它们的systems的components。
- [Unity.Hybrid.Renderer](https://docs.unity3d.com/Packages/com.unity.rendering.hybrid@latest)：提供components和systems以在Unity运行时中渲染ECS entities。

## GamePlayer支持包

DOTS中的某些游戏功能需要额外的程序包来支持它们。有关需要其他软件包的功能列表，请参见下表。

| **Feature**                       | **Packages**                                                 |
| :-------------------------------- | :----------------------------------------------------------- |
| **DOTS ECS**                      | [com.unity.entities](https://docs.unity3d.com/Packages/com.unity.entities@latest) |
| **Rendering**                     | [com.unity.rendering.hybrid](https://docs.unity3d.com/Packages/com.unity.rendering.hybrid@latest) |
| **- Hybrid Renderer V2**          | [com.unity.render-pipelines.high-definition](https://docs.unity3d.com/Packages/com.unity.render-pipelines.high-definition@latest) or [com.unity.render-pipelines.universal](https://docs.unity3d.com/Packages/com.unity.render-pipelines.universal@latest) |
| **- Animation**                   | [com.unity.animation](https://docs.unity3d.com/Packages/com.unity.animation@latest) |
| **Audio**                         | [com.unity.audio.dspgraph](https://docs.unity3d.com/Packages/com.unity.audio.dspgraph@latest) |
| **Physics**                       | [com.unity.physics](https://docs.unity3d.com/Packages/com.unity.physics@latest) or [com.havok.physics](https://docs.unity3d.com/Packages/com.havok.physics@latest) |
| **- Smooth Penetration Recovery** | [com.havok.physics](https://docs.unity3d.com/Packages/com.havok.physics@latest) |
| **- Stable Object Stacking**      | [com.havok.physics](https://docs.unity3d.com/Packages/com.havok.physics@latest) |
| **- Remove Speculative Contacts** | [com.havok.physics](https://docs.unity3d.com/Packages/com.havok.physics@latest) |
| **- Rigidbody Sleeping**          | [com.havok.physics](https://docs.unity3d.com/Packages/com.havok.physics@latest) |
| **- Visual Debugger**             | [com.havok.physics](https://docs.unity3d.com/Packages/com.havok.physics@latest) |
| **Multiplayer**                   | [com.unity.netcode](https://docs.unity3d.com/Packages/com.unity.netcode@latest) |
| **- Lag Compensation**            | [com.unity.physics](https://docs.unity3d.com/Packages/com.unity.physics@latest) |
| **Project Building**              | [com.unity.platforms](https://docs.unity3d.com/Packages/com.unity.platforms@latest) |
| **- Android**                     | [com.unity.platforms.android](https://docs.unity3d.com/Packages/com.unity.platforms.android@latest) |
| **- Linux**                       | [com.unity.platforms.linux](https://docs.unity3d.com/Packages/com.unity.platforms.linux@latest) |
| **- macOS**                       | [com.unity.platforms.macos](https://docs.unity3d.com/Packages/com.unity.platforms.macos@latest) |
| **- Web**                         | [com.unity.platforms.web](https://docs.unity3d.com/Packages/com.unity.platforms.web@latest) |
| **- Windows**                     | [com.unity.platforms.windows](https://docs.unity3d.com/Packages/com.unity.platforms.windows@latest) |

## 创作概述

您可以使用Unity Editor（带有必需的DOTS包）来创建基于DOTS的游戏。在编辑器中，您可以正常使用GameObjects来编写场景，而ECS代码会将GameObjects转换为entities。

使用DOTS的最大区别在于，您无需[定义](https://docs.unity3d.com/ScriptReference/MonoBehaviour.html)自己的[MonoBehaviours](https://docs.unity3d.com/ScriptReference/MonoBehaviour.html)来存储实例数据和实现自定义游戏逻辑，而是可以定义[ECS components](https://docs.unity3d.com/Packages/com.unity.entities@0.14/manual/ecs_components.html)以在运行时存储数据，并为自定义逻辑编写[systems](https://docs.unity3d.com/Packages/com.unity.entities@0.14/manual/ecs_systems.html)。

### GameObject转换

在GameObject转换期间，各种转换系统会处理它们识别的MonoBehaviour组件，然后将它们转换为基于ECS的components。例如，一个Unity.Transforms转换系统将检查该`UnityEngine.Transform`组件并添加ECS component（例如[LocalToWorld](https://docs.unity3d.com/Packages/com.unity.entities@0.14/api/Unity.Transforms.LocalToWorld.html)）来替换它。

您可以实现[IConvertGameObjectToEntity](https://docs.unity3d.com/Packages/com.unity.entities@0.14/api/Unity.Entities.IConvertGameObjectToEntity.html) MonoBehaviour组件以指定自定义转换步骤。*ECS转换的GameObjects数量与它创建的entities数量之间通常没有一对一的关系。也不是在GameObject上的MonoBehaviours数量与它添加到entities上的ECS components数量之间。*

![image-20200817093035244](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200817093035244.png)

如果ECS转换的代码具有[ConvertToEntity](https://docs.unity3d.com/Packages/com.unity.entities@0.14/api/Unity.Entities.ConvertToEntity.html) MonoBehaviour组件，或者它是SubScene的一部分，则ECS转换代码将转换GameObject。在任何一种情况下，为各种DOTS功能（例如Unity.Transforms和Unity.Hybrid.Render）提供的转换系统都会处理GameObject或Scene Asset及其任何子GameObject。

用ConvertToEntity转换GameObjects和用SubScene转换之间的区别是：ECS将它从转换SubScene生成的entities数据序列化并保存到磁盘上。您可以在运行时非常快速地加载或流化此序列化数据。相比之下，ECS始终在运行时将具有ConvertToEntity MonoBehaviours的GameObjects进行转换。

最佳实践是使用标准MonoBehaviours进行创作，并使用`IConvertGameObjectToEntity`将那些创作组件的值应用于[IComponentData](https://docs.unity3d.com/Packages/com.unity.entities@0.14/api/Unity.Entities.IComponentData.html)结构以供运行时使用。通常，最方便编写的数据布局不是运行时最高效的数据布局。

您可以使用[IConvertGameObjectToEntity](https://docs.unity3d.com/Packages/com.unity.entities@0.14/api/Unity.Entities.IConvertGameObjectToEntity.html)来自定义SubScene中任何GameObject或具有`ConvertToEntity`MonoBehaviour 的GameObject或具有`ConvertToEntity`MonoBehaviour 的GameObject的子物体的转换。

**注意：**基于DOTS的应用程序的创作工作流是活跃开发的领域。大致结构已经确定，但是后面可能还会有很多细节上的变化。

## Generated authoring components

Unity可以自动为简单的运行时ECS components生成authoring components。Unity生成authoring components时，您可以将包含ECS IComponentData的脚本直接添加到编辑器中的GameObject。然后，您可以使用“ **Inspector”**窗口来设置component的初始值。

### 对于IComponentData

Unity可以自动为简单的[IComponentData](https://docs.unity3d.com/Packages/com.unity.entities@0.14/api/Unity.Entities.IComponentData.html) components生成创作authoring components。当Unity生成authoring components时，您可以在Unity编辑器中直接将`IComponentData`添加到场景中GameObject中。然后，您可以使用**Inspector**窗口来设置component的初始值。

为了表示您要生成authoring components，请将`[GenerateAuthoringComponent]`attribute添加到IComponentData声明中。Unity自动生成一个MonoBehaviour类，该类包含component的公共字段，并提供一个Conversion方法，将这些字段转换为运行时component数据。

```cs
[GenerateAuthoringComponent]
public struct RotationSpeed_ForEach : IComponentData
{
    public float RadiansPerSecond;
}
```

请注意以下限制：

- 单个C＃文件中只有一个component可以具有生成的authoring component，并且C＃文件中不能包含另一个MonoBehaviour。
- ECS仅反射公共字段，并且与component中指定的名称相同。
- ECS将IComponentData中的Entity类型的字段反射为它生成的MonoBehaviour中的GameObject类型的字段。ECS将您分配给这些字段的GameObjects或Prefabs转换为引用的Prefabs。
- 仅反射公共字段，它们的名称与组件中指定的名称相同。
- IComponentData中的Entity类型的字段在生成的MonoBehaviour中反射为GameObject类型的字段。您分配给这些字段的GameObjects或Prefabs会转换为引用的prefabs。

### 对于IBufferElementData

您还可以通过添加`[GenerateAuthoringComponent]`attribute来为实现`IBufferElementData`的类型生成authoring component：

```cs
[GenerateAuthoringComponent]
public struct IntBufferElement: IBufferElementData
{
    public int Value;
}
```

在此示例中，将生成一个名为`IntBufferElementAuthoring`（继承自`MonoBehaviour`）的类，并公开一个`List<int>`类型为public的字段。在转换过程中，此List将转换为`DynamicBuffer<IntBufferElement>`，然后添加到转换后的entity中。

请注意以下限制：

- 单个C＃文件中只有一个component可以有生成的authoring component，并且C＃文件中不能包含另一个MonoBehaviour。
- `IBufferElementData` 对于包含2个或更多字段的类型，无法自动生成authoring component。
- `IBufferElementData` 无法为具有显式布局的类型自动生成authoring component。

# TransformSystem

------

## 第1节: Non-hierarchical Transforms (Basic)

LocalToWorld（float4x4）表示从本地空间到世界空间的转换。它是规范的表示形式，并且是唯一可以依靠它在systems之间传递本地空间的component。

- 某些DOTS功能可能依赖LocalToWorld的存在才能起作用。
- 例如，RenderMesh component依赖存在的LocalToWorld component来渲染实例。
- 如果仅存在LocalToWorld transform component，则任何transform system都不会写入或影响LocalToWorld数据。
- 如果没有其他transform component与同一entity相关联，则用户代码可以直接写入LocalToWorld以定义实例的tranform。

所有transform system和所有其他transform component的目的是提供用于写入LocalToWorld的接口。

LocalToWorld = Translation * Rotation * Scale

如果同时存在平移（float3），旋转（四元数）或缩放（float）components的任何组合(都和LocalToWorld component相关联，且同时存在)，则transform system将组合这些components并写入LocalToWorld。

具体而言，这些components组合中的每一个将以以下形式写入LocalToWorld：

- [TRSToLocalToWorldSystem] LocalToWorld <= Translation
- [TRSToLocalToWorldSystem] LocalToWorld <= Translation * Rotation
- [TRSToLocalToWorldSystem] LocalToWorld <= Translation * Rotation * Scale
- [TRSToLocalToWorldSystem] LocalToWorld <= Rotation
- [TRSToLocalToWorldSystem] LocalToWorld <= Rotation * Scale
- [TRSToLocalToWorldSystem] LocalToWorld <= Scale

例如，如果存在以下components...

| (Entity)     |
| :----------- |
| LocalToWorld |
| Translation  |
| Rotation     |

...所以transform system应该是:

- [TRSToLocalToWorldSystem] Write LocalToWorld <= Translation * Rotation

![image-20200817114319911](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200817114319911.png)

或者，当以下components存在时

| (Entity)     |
| :----------- |
| LocalToWorld |
| Translation  |
| Rotation     |
| Scale        |

...所以transform system应该是:

- [TRSToLocalToWorldSystem] Write LocalToWorld <= Translation * Rotation * Scale

![image-20200817114517341](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200817114517341.png)

## 第2节：Hierarchical Transforms (Basic)

transform system需要LocalToParent和Parent component才能基于分层transform编写LocalToWorld。

- LocalToParent（float4x4）表示从本地空间到父级本地空间的转换。
- 父级（entity）引用父级的LocalToWorld。
- 如果没有其他transform systems定义为写入用户代码的目标，则用户代码可以直接写入LocalToParent。

例如，如果存在以下components...

| Parent (Entity) | Child (Entity) |
| :-------------- | :------------- |
| LocalToWorld    | LocalToWorld   |
| Translation     | LocalToParent  |
| Rotation        | Parent         |
| Scale           |                |

...然后transform system将会是：

1. [TRSToLocalToWorldSystem]父级：如上文“Non-hierarchical Transforms (Basic)”中所定义的那样写入LocalToWorld
2. [LocalToParentSystem]子级：LocalToWorld <= LocalToWorld [Parent] * LocalToParent

![image-20200817134717984](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200817134717984.png)

与父entity ID关联的LocalToWorld component在与子entity ID关联的LocalToParent相乘之前，自己一定要先计算。

注意：循环图关系无效。结果是不确定的。

当更改层次结构（拓扑）（即，添加，删除或更改任何父级component）时，内部状态作为SystemStateComponentData添加为：

- 与父entity ID相关联的子component（entity的ISystemStateBufferElementData）
- 与子entity ID相关联的PreviousParent component（entity的ISystemStateComponentData）

| Parent (Entity) | Child (Entity)  |
| :-------------- | :-------------- |
| LocalToWorld    | LocalToWorld    |
| Translation     | LocalToParent   |
| Rotation        | Parent          |
| Scale           | PreviousParent* |
| Child*          |                 |

这些component的添加，删除和更新由[ParentSystem]处理。我们希望transform system外部的system不会读取或写入这些components。

LocalToParent = Translation * Rotation * Scale

如果同时存在平移（float3），旋转（四元数）或缩放（float）component的任何组合(都和LocalToWorld component相关联，且同时存在)，则transform system将组合这些components并写入LocalToParent。

具体而言，这些components组合中的每一个将以下方式写入LocalToParent：

- [TRSToLocalToParentSystem] LocalToParent <= Translation
- [TRSToLocalToParentSystem] LocalToParent <= Translation * Rotation
- [TRSToLocalToParentSystem] LocalToParent <= Translation * Rotation * Scale
- [TRSToLocalToParentSystem] LocalToParent <= Rotation
- [TRSToLocalToParentSystem] LocalToParent <= Rotation * Scale
- [TRSToLocalToParentSystem] LocalToParent <= Scale

例如，如果存在以下components...

| Parent (Entity) | Child (Entity)  |
| :-------------- | :-------------- |
| LocalToWorld    | LocalToWorld    |
| Translation     | LocalToParent   |
| Rotation        | Parent          |
| Scale           | PreviousParent* |
| Child*          | Translation     |
|                 | Rotation        |
|                 | Scale           |

...然后transform system将会是：

1. [TRSToLocalToWorldSystem]父级：如上文“Non-hierarchical Transforms (Basic)”中所定义的那样写入LocalToWorld
2. [TRSToLocalToParentSystem]子级：写入LocalToParent <= Translation * Rotation * Scale
3. [LocalToParentSystem]子级：写入LocalToWorld <= LocalToWorld [Parent] * LocalToParent

![image-20200817135448128](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200817135448128.png)

父级当然可以成为LocalToWorld其他components的child。

例如，如果存在以下components...

| Parent (Entity) | Child (Entity)  |
| :-------------- | :-------------- |
| LocalToWorld    | LocalToWorld    |
| LocalToParent   | LocalToParent   |
| Parent          | Parent          |
| PreviousParent* | PreviousParent* |
| Child*          | Translation     |
| Translation     | Rotation        |
| Rotation        | Scale           |
| Scale           |                 |

...然后transform system将会是：

1. [TRSToLocalToParentSystem] Parent: Write LocalToParent <= Translation * Rotation * Scale
2. [TRSToLocalToParentSystem] Child: Write LocalToParent <= Translation * Rotation * Scale
3. [LocalToParentSystem] Parent: Write LocalToWorld <= LocalToWorld[Parent] * LocalToParent
4. [LocalToParentSystem] Child: Write LocalToWorld <= LocalToWorld[Parent] * LocalToParent

![image-20200817135933951](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200817135933951.png)

------

## 第3节：Default Conversion (Basic)

混合转换：

UnityEngine.Transform MonoBehaviours作为GameObjects一部分，包含在Sub Scenes中，或者在带有“Convert To Entity” Monobehaviours的GameObjects上，具有默认转换为Transform system components的功能。可以在Unity.Transforms.Hybrid程序集的TransformConversion system中找到该转换。

1. 与被转换的GameObject相关联的entity（且具有静态component），仅将LocalToWorld添加到生成的entities中。因此，在静态实例的情况下，在运行时不会发生任何transform system更新。
2. 对于非静态entities，（1）Positioncomponent将以Transform.position值填充自己。（2）Rotation 将以Transform.rotation值填充自己。（3）Transform.parent == null
   - 对于非标准化的Transform.localScale，将为NonUniformScale component添加Transform.localScale值。（4）如果Transform.parent！= null，但在此要转换的层次结构的顶部：
   - 对于非标准化的Transform.lossyScale，将向NonUniformScale组件添加Transform.lossyScale值。（5）对于其他Transform.parent！= null的情况，
   - 父component将与entity一起添加，该entity引用转换后的Transform.parent GameObject。
   - 将添加LocalToParent component。

------

## 第4节：Non-hierarchical Transforms (Advanced)

NonUniformScale（float3）作为Scale的替代方法，用于指定每个轴的比例。请注意，并非所有DOTS功能都完全支持非均匀缩放。请务必查看这些功能的文档以了解其限制。

- [TRSToLocalToWorldSystem] LocalToWorld <= Translation
- [TRSToLocalToWorldSystem] LocalToWorld <= Translation * Rotation
- [TRSToLocalToWorldSystem] LocalToWorld <= Translation * Rotation * NonUniformScale
- [TRSToLocalToWorldSystem] LocalToWorld <= Rotation
- [TRSToLocalToWorldSystem] LocalToWorld <= Rotation * NonUniformScale
- [TRSToLocalToWorldSystem] LocalToWorld <= NonUniformScale

同时存在Scale和NonUniform scale不是合法情况，但是还会有结果。将使用Scale，将忽略NonUniformScale。

例如，如果存在以下components...

| (Entity)        |
| :-------------- |
| LocalToWorld    |
| Translation     |
| Rotation        |
| NonUniformScale |

...然后transform system将会是：

- [TRSToLocalToWorldSystem] Write LocalToWorld <= Translation * Rotation * NonUniformScale

![image-20200817141223878](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200817141223878.png)

旋转分量可以由用户代码直接作为四元数写入。但是，如果首选Euler接口，则每个旋转顺序都可以使用相应component，这将导致对Rotation component的写入（如果存在）。

- [RotationEulerSystem] Rotation <= RotationEulerXYZ
- [RotationEulerSystem] Rotation <= RotationEulerXZY
- [RotationEulerSystem] Rotation <= RotationEulerYXZ
- [RotationEulerSystem] Rotation <= RotationEulerYZX
- [RotationEulerSystem] Rotation <= RotationEulerZXY
- [RotationEulerSystem] Rotation <= RotationEulerZYX

例如，如果存在以下components...

| (Entity)         |
| :--------------- |
| LocalToWorld     |
| Translation      |
| Rotation         |
| RotationEulerXYZ |

...然后transform system将会：

1. [RotationEulerSystem] Write Rotation <= RotationEulerXYZ
2. [TRSToLocalToWorldSystem] Write LocalToWorld <= Translation * Rotation * Scale

![img](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20200903201506.png)

多个RotationEulerXXX组件与同一entity相关联是错误的，但是会有结果。将应用按优先级顺序找到的第一个。该命令是：

1. RotationEulerXYZ
2. RotationEulerXZY
3. RotationEulerYXZ
4. RotationEulerYZX
5. RotationEulerZXY
6. RotationEulerZYX

对于更复杂的旋转要求，可以使用CompositeRotation（float4x4） component替代Rotation。

对Rotation有效的所有组合对CompositeRotation也有效。即

- [TRSToLocalToWorldSystem] LocalToWorld <= Translation * CompositeRotation
- [TRSToLocalToWorldSystem] LocalToWorld <= Translation * CompositeRotation * Scale
- [TRSToLocalToWorldSystem] LocalToWorld <= CompositeRotation
- [TRSToLocalToWorldSystem] LocalToWorld <= CompositeRotation * Scale
- [TRSToLocalToWorldSystem] LocalToWorld <= Translation * CompositeRotation
- [TRSToLocalToWorldSystem] LocalToWorld <= Translation * CompositeRotation * NonUniformScale
- [TRSToLocalToWorldSystem] LocalToWorld <= CompositeRotation
- [TRSToLocalToWorldSystem] LocalToWorld <= CompositeRotation * NonUniformScale

用户可以将CompositeRotation component直接写为float4x4。但是，如果首选Maya/FBX样式的接口，则可以使用将要写入CompositeRotation component（如果存在）的component。



CompositeRotation = RotationPivotTranslation * RotationPivot * Rotation * PostRotation * RotationPivot^-1



如果RotationPivotTranslation（float3），RotationPivot（float3），Rotation（四元数）或PostRotation（四元数）component与CompositeRotation component一起存在，则transform system将组合这些components并写入CompositeRotation。

具体来说，这些components组合中的每一个都将以以下形式写如CompositeRotation：

- [CompositeRotationSystem] CompositeRotation <= RotationPivotTranslation
- [CompositeRotationSystem] CompositeRotation <= RotationPivotTranslation * RotationPivot * Rotation * RotationPivot^-1
- [CompositeRotationSystem] CompositeRotation <= RotationPivotTranslation * RotationPivot * Rotation * PostRotation * RotationPivot^-1
- [CompositeRotationSystem] CompositeRotation <= RotationPivotTranslation * RotationPivot * PostRotation * RotationPivot^-1
- [CompositeRotationSystem] CompositeRotation <= RotationPivotTranslation * Rotation
- [CompositeRotationSystem] CompositeRotation <= RotationPivotTranslation * Rotation * PostRotation
- [CompositeRotationSystem] CompositeRotation <= RotationPivotTranslation * PostRotation
- [CompositeRotationSystem] CompositeRotation <= RotationPivot * Rotation * RotationPivot^-1
- [CompositeRotationSystem] CompositeRotation <= RotationPivot * Rotation * PostRotation * RotationPivot^-1
- [CompositeRotationSystem] CompositeRotation <= PostRotation
- [CompositeRotationSystem] CompositeRotation <= Rotation
- [CompositeRotationSystem] CompositeRotation <= Rotation * PostRotation

如果指定的RotationPivot不带任何Rotation，则PostRotation对CompositeRotation没有其他影响。

注意，由于 Rotation 被重新用作CompositeRotation的源头之一，因此 Rotation 的替代数据接口仍然可用。

例如，如果存在以下components...

| (Entity)                 |
| :----------------------- |
| LocalToWorld             |
| Translation              |
| CompositeRotation        |
| Rotation                 |
| RotationPivotTranslation |
| RotationPivot            |
| PostRotation             |
| RotationEulerXYZ         |
| Scale                    |

...然后transform system将会是：

1. [CompositeRotationSystem] Write CompositeRotation <= RotationPivotTranslation * RotationPivot * Rotation * PostRotation * RotationPivot^-1
2. [TRSToLocalToWorldSystem] Write LocalToWorld <= Translation * CompositeRotation * Scale

![img](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20200903201517.png)

用户可以将PostRotation component直接作为四元数写入。但是，如果首选Euler接口，则每个旋转顺序都可以使用components，这将导致写入PostRotation component（如果存在）。

- [PostRotationEulerSystem] PostRotation <= PostRotationEulerXYZ
- [PostRotationEulerSystem] PostRotation <= PostRotationEulerXZY
- [PostRotationEulerSystem] PostRotation <= PostRotationEulerYXZ
- [PostRotationEulerSystem] PostRotation <= PostRotationEulerYZX
- [PostRotationEulerSystem] PostRotation <= PostRotationEulerZXY
- [PostRotationEulerSystem] PostRotation <= PostRotationEulerZYX

例如，如果存在以下组件...

| (Entity)                 |
| :----------------------- |
| LocalToWorld             |
| Translation              |
| CompositeRotation        |
| Rotation                 |
| RotationPivotTranslation |
| RotationPivot            |
| RotationEulerXYZ         |
| PostRotation             |
| PostRotationEulerXYZ     |
| Scale                    |

...然后 transform system 将会是：

1. [RotationEulerSystem] Write Rotation <= RotationEulerXYZ
2. [PostRotationEulerSystem] Write PostRotation <= PostRotationEulerXYZ
3. [CompositeRotationSystem] Write CompositeRotation <= RotationPivotTranslation * RotationPivot * Rotation * PostRotation * RotationPivot^-1
4. [TRSToLocalToWorldSystem] Write LocalToWorld <= Translation * CompositeRotation * Scale

![img](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20200903201636.png)

对于更复杂的Scale要求，可以使用CompositeScale（float4x4）component替代Scale（或NonUniformScale）。

对Scale或NonUniformScale有效的所有组合对CompositeScale也有效。即

- [TRSToLocalToWorldSystem] LocalToWorld <= Translation * Rotation * CompositeScale
- [TRSToLocalToWorldSystem] LocalToWorld <= Rotation * CompositeScale
- [TRSToLocalToWorldSystem] LocalToWorld <= CompositeScale
- [TRSToLocalToWorldSystem] LocalToWorld <= Translation * CompositeRotation * CompositeScale
- [TRSToLocalToWorldSystem] LocalToWorld <= CompositeRotation * CompositeScale

用户可以将CompositeScale component直接写为float4x4。但是，如果首选Maya / FBX风格的接口，则可以使用可以写入CompositeScale component（如果存在）的components。



CompositeScale = ScalePivotTranslation * ScalePivot * Scale * ScalePivot^-1 CompositeScale = ScalePivotTranslation * ScalePivot * NonUniformScale * ScalePivot^-1



如果同时存在ScalePivotTranslation（float3），ScalePivot（float3），Scale（float）component与CompositeScale component的任何组合，则transform system将组合这些components并写入CompositeScale。

或者，如果同时存在ScalePivotTranslation（float3），ScalePivot（float3），NonUniformScale（float3）component和CompositeScale component的任意组合，则transform system将组合这些components并写入CompositeScale。

具体来说，这些components组合中的每一个都将写入CompositeRotation：

- [CompositeScaleSystem] CompositeScale <= ScalePivotTranslation
- [CompositeScaleSystem] CompositeScale <= ScalePivotTranslation * ScalePivot * Scale * ScalePivot^-1
- [CompositeScaleSystem] CompositeScale <= ScalePivotTranslation * Scale
- [CompositeScaleSystem] CompositeScale <= ScalePivot * Scale * ScalePivot^-1
- [CompositeScaleSystem] CompositeScale <= Scale
- [CompositeScaleSystem] CompositeScale <= ScalePivotTranslation * ScalePivot * NonUniformScale * ScalePivot^-1
- [CompositeScaleSystem] CompositeScale <= ScalePivotTranslation * Scale
- [CompositeScaleSystem] CompositeScale <= ScalePivot * NonUniformScale * ScalePivot^-1
- [CompositeScaleSystem] CompositeScale <= NonUniformScale

如果指定ScalePivot且不使用Scale或NonScale中的任何一个，则NonUniformScale都没有其他影响，对CompositeScale也没有其他影响。

例如，如果存在以下components...

| (Entity)                 |
| :----------------------- |
| LocalToWorld             |
| Translation              |
| CompositeRotation        |
| Rotation                 |
| RotationPivotTranslation |
| RotationPivot            |
| RotationEulerXYZ         |
| PostRotation             |
| PostRotationEulerXYZ     |
| CompositeScale           |
| Scale                    |
| ScalePivotTranslation    |
| ScalePivot               |

...然后 transform system 将会是：

1. [RotationEulerSystem] Write Rotation <= RotationEulerXYZ
2. [PostRotationEulerSystem] Write PostRotation <= PostRotationEulerXYZ
3. [CompositeScaleSystem] Write CompositeScale <= ScalePivotTranslation * ScalePivot * Scale * ScalePivot^-1
4. [CompositeRotationSystem] Write CompositeRotation <= RotationPivotTranslation * RotationPivot * Rotation * PostRotation * RotationPivot^-1
5. [TRSToLocalToWorldSystem] Write LocalToWorld <= Translation * CompositeRotation * CompositeScale

![img](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20200903202027.png)

------

## 第5节：Hierarchical Transforms (Advanced)

注意：高级层次转换组件规则很大程度上反映了非层次组件的用法，只是它们正在写入LocalToParent（而不是LocalToWorld）。层次转换特有的额外component是ParentScaleInverse。

------

NonUniformScale（float3）作为Scale的替代方法，用于指定每轴的比例。请注意，并非所有DOTS功能都完全支持非均匀缩放。请务必查看这些功能的文档以了解其限制。

- [TRSToLocalToParentSystem] LocalToParent <= Translation
- [TRSToLocalToParentSystem] LocalToParent <= Translation * Rotation
- [TRSToLocalToParentSystem] LocalToParent <= Translation * Rotation * NonUniformScale
- [TRSToLocalToParentSystem] LocalToParent <= Rotation
- [TRSToLocalToParentSystem] LocalToParent <= Rotation * NonUniformScale
- [TRSToLocalToParentSystem] LocalToParent <= NonUniformScale

同时存在Scale和NonUniform scale不是合法情况，但是定义了结果。将使用Scale，将忽略NonUniformScale。

例如，如果存在以下components...

| Parent (Entity) | Child (Entity)  |
| :-------------- | :-------------- |
| LocalToWorld    | LocalToWorld    |
| Translation     | LocalToParent   |
| Rotation        | Parent          |
| Scale           | PreviousParent* |
| Child*          | Translation     |
|                 | Rotation        |
|                 | NonUniformScale |

...然后transform system将会是：

1. TRSToLocalToWorldSystem] Parent: Write LocalToWorld as defined above in "Non-hierarchical Transforms (Basic)"
2. [TRSToLocalToParentSystem] Child: Write LocalToParent <= Translation * Rotation * NonUniformScale
3. [LocalToParentSystem] Child: Write LocalToWorld <= LocalToWorld[Parent] * LocalToParent

![img](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20200903202314.png)

父级LocalToWorld乘以子级LocalToWorld，其中包括任何缩放比例。但是，如果首选删除“父级比例”（“AKA比例补偿”），则可以使用ParentScaleInverse。

- [TRSToLocalToParentSystem] LocalToParent <= ParentScaleInverse
- [TRSToLocalToParentSystem] LocalToParent <= Translation * ParentScaleInverse
- [TRSToLocalToParentSystem] LocalToParent <= Translation * ParentScaleInverse * Rotation
- [TRSToLocalToParentSystem] LocalToParent <= Translation * ParentScaleInverse * Rotation * NonUniformScale
- [TRSToLocalToParentSystem] LocalToParent <= Translation * ParentScaleInverse * CompositeRotation
- [TRSToLocalToParentSystem] LocalToParent <= Translation * ParentScaleInverse * CompositeRotation * NonUniformScale
- [TRSToLocalToParentSystem] LocalToParent <= Translation * ParentScaleInverse * Rotation * Scale
- [TRSToLocalToParentSystem] LocalToParent <= Translation * ParentScaleInverse * CompositeRotation * Scale
- [TRSToLocalToParentSystem] LocalToParent <= Translation * ParentScaleInverse * Rotation * CompositeScale
- [TRSToLocalToParentSystem] LocalToParent <= Translation * ParentScaleInverse * CompositeRotation * CompositeScale
- [TRSToLocalToParentSystem] LocalToParent <= ParentScaleInverse * Rotation
- [TRSToLocalToParentSystem] LocalToParent <= ParentScaleInverse * Rotation * NonUniformScale
- [TRSToLocalToParentSystem] LocalToParent <= ParentScaleInverse * CompositeRotation * NonUniformScale
- [TRSToLocalToParentSystem] LocalToParent <= ParentScaleInverse * Rotation * Scale
- [TRSToLocalToParentSystem] LocalToParent <= ParentScaleInverse * CompositeRotation
- [TRSToLocalToParentSystem] LocalToParent <= ParentScaleInverse * CompositeRotation * Scale
- [TRSToLocalToParentSystem] LocalToParent <= ParentScaleInverse * Rotation * CompositeScale
- [TRSToLocalToParentSystem] LocalToParent <= ParentScaleInverse * CompositeRotation * CompositeScale

如果存在任何显式分配的父scale值的倒数，则将其写入ParentScaleInverse，如下所示：

- [arentScaleInverseSystem] ParentScaleInverse <= CompositeScale[Parent]^-1
- [ParentScaleInverseSystem] ParentScaleInverse <= Scale[Parent]^-1
- [ParentScaleInverseSystem] ParentScaleInverse <= NonUniformScale[Parent]^-1

如果LocalToWorld [Parent]由用户直接编写，或者以其他没有明确使用scale component的方式应用缩放，则不会将任何内容写入ParentScaleInverse。应用该缩放值的倒数写到ParentScaleInverse是system的责任。在这种情况下，system未更新ParentScaleInverse的结果是不确定的。

例如，如果存在以下components...

| Parent (Entity) | Child (Entity)     |
| :-------------- | :----------------- |
| LocalToWorld    | LocalToWorld       |
| Translation     | LocalToParent      |
| Rotation        | Parent             |
| Scale           | PreviousParent*    |
| Child*          | Translation        |
|                 | Rotation           |
|                 | ParentScaleInverse |

...然后transform system将会是：

1. [TRSToLocalToWorldSystem] Parent: Write LocalToWorld as defined above in "Non-hierarchical Transforms (Basic)"
2. [ParentScaleInverseSystem] Child: ParentScaleInverse <= Scale[Parent]^-1
3. [TRSToLocalToParentSystem] Child: Write LocalToParent <= Translation * ParentScaleInverse * Rotation
4. [LocalToParentSystem] Child: Write LocalToWorld <= LocalToWorld[Parent] * LocalToParent

![img](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20200903202754.png)

旋转分量可以由用户直接作为四元数写入。但是，如果首选Euler接口，则每个旋转顺序都可以使用component，这将导致对Rotation component的写入（如果存在）。

- [RotationEulerSystem] Rotation <= RotationEulerXYZ
- [RotationEulerSystem] Rotation <= RotationEulerXZY
- [RotationEulerSystem] Rotation <= RotationEulerYXZ
- [RotationEulerSystem] Rotation <= RotationEulerYZX
- [RotationEulerSystem] Rotation <= RotationEulerZXY
- [RotationEulerSystem] Rotation <= RotationEulerZYX

例如，如果存在以下components...

| Parent (Entity) | Child (Entity)   |
| :-------------- | :--------------- |
| LocalToWorld    | LocalToWorld     |
| Translation     | LocalToParent    |
| Rotation        | Parent           |
| Scale           | PreviousParent*  |
| Child*          | Translation      |
|                 | Rotation         |
|                 | RotationEulerXYZ |

...然后transform system将会是：

1. [TRSToLocalToWorldSystem] Parent: Write LocalToWorld as defined above in "Non-hierarchical Transforms (Basic)"
2. [RotationEulerSystem] Child: Write Rotation <= RotationEulerXYZ
3. [TRSToLocalToParentSystem] Child: Write LocalToParent <= Translation * Rotation
4. [LocalToParentSystem] Child: Write LocalToWorld <= LocalToWorld[Parent] * LocalToParent

![img](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20200903202852.png)

对于更复杂的旋转要求，可以使用CompositeRotation（float4x4） component替代Rotation。

对Rotation有效的所有组合对CompositeRotation也有效。即

- [TRSToLocalToParentSystem] LocalToParent <= Translation * CompositeRotation
- [TRSToLocalToParentSystem] LocalToParent <= Translation * CompositeRotation * Scale
- [TRSToLocalToParentSystem] LocalToParent <= Translation * CompositeRotation * NonUniformScale
- [TRSToLocalToParentSystem] LocalToParent <= Translation * CompositeRotation * CompositeScale
- [TRSToLocalToParentSystem] LocalToParent <= Translation * ParentScaleInverse * CompositeRotation
- [TRSToLocalToParentSystem] LocalToParent <= Translation * ParentScaleInverse * CompositeRotation * Scale
- [TRSToLocalToParentSystem] LocalToParent <= Translation * ParentScaleInverse * CompositeRotation * NonUniformScale
- [TRSToLocalToParentSystem] LocalToParent <= Translation * ParentScaleInverse * CompositeRotation * CompositeScale
- [TRSToLocalToParentSystem] LocalToParent <= ParentScaleInverse * CompositeRotation
- [TRSToLocalToParentSystem] LocalToParent <= ParentScaleInverse * CompositeRotation * Scale
- [TRSToLocalToParentSystem] LocalToParent <= ParentScaleInverse * CompositeRotation * NonUniformScale
- [TRSToLocalToParentSystem] LocalToParent <= ParentScaleInverse * CompositeRotation * CompositeScale
- [TRSToLocalToParentSystem] LocalToParent <= CompositeRotation
- [TRSToLocalToParentSystem] LocalToParent <= CompositeRotation * Scale
- [TRSToLocalToParentSystem] LocalToParent <= CompositeRotation * NonUniformScale
- [TRSToLocalToParentSystem] LocalToParent <= CompositeRotation * CompositeScale

用户代码可以将CompositeRotation component直接写为float4x4。但是，如果首选Maya/FBX样式的接口，则可以使用将会写入CompositeRotation component（如果存在）的component。



CompositeRotation = RotationPivotTranslation * RotationPivot * Rotation * PostRotation * RotationPivot^-1



如果RotationPivotTranslation（float3），RotationPivot（float3），Rotation（四元数）或PostRotation（四元数）component与CompositeRotation component一起存在，则transform system将组合这些components并写入CompositeRotation。

具体来说，这些component组合中的每一个都将写入CompositeRotation：

- [CompositeRotationSystem] CompositeRotation <= RotationPivotTranslation
- [CompositeRotationSystem] CompositeRotation <= RotationPivotTranslation * RotationPivot * Rotation * RotationPivot^-1
- [CompositeRotationSystem] CompositeRotation <= RotationPivotTranslation * RotationPivot * Rotation * PostRotation * RotationPivot^-1
- [CompositeRotationSystem] CompositeRotation <= RotationPivotTranslation * RotationPivot * PostRotation * RotationPivot^-1
- [CompositeRotationSystem] CompositeRotation <= RotationPivotTranslation * Rotation
- [CompositeRotationSystem] CompositeRotation <= RotationPivotTranslation * Rotation * PostRotation
- [CompositeRotationSystem] CompositeRotation <= RotationPivotTranslation * PostRotation
- [CompositeRotationSystem] CompositeRotation <= RotationPivot * Rotation * RotationPivot^-1
- [CompositeRotationSystem] CompositeRotation <= RotationPivot * Rotation * PostRotation * RotationPivot^-1
- [CompositeRotationSystem] CompositeRotation <= PostRotation
- [CompositeRotationSystem] CompositeRotation <= Rotation
- [CompositeRotationSystem] CompositeRotation <= Rotation * PostRotation

如果指定的RotationPivot不带任何Rotation，则PostRotation对CompositeRotation没有其他影响。

注意，由于Rotation被重新用作CompositeRotation的源头，因此Rotation的替代数据接口仍然可用。

例如，如果存在以下components...

| Parent (Entity) | Child (Entity)           |
| :-------------- | :----------------------- |
| LocalToWorld    | LocalToWorld             |
| Translation     | LocalToParent            |
| Rotation        | Parent                   |
| Scale           | PreviousParent*          |
| Child*          | Translation              |
|                 | CompositeRotation        |
|                 | Rotation                 |
|                 | RotationPivotTranslation |
|                 | RotationPivot            |
|                 | PostRotation             |
|                 | RotationEulerXYZ         |
|                 | Scale                    |

...然后transform system将会是：

1. [TRSToLocalToWorldSystem] Parent: Write LocalToWorld as defined above in "Non-hierarchical Transforms (Basic)"
2. [RotationEulerSystem] Child: Write Rotation <= RotationEulerXYZ
3. [CompositeRotationSystem] Child: Wirte CompositeRotation <= RotationPivotTranslation * RotationPivot * Rotation * PostRotation * RotationPivot^-1
4. [TRSToLocalToParentSystem] Child: Write LocalToParent <= Translation * CompositeRotation * Scale
5. [LocalToParentSystem] Child: Write LocalToWorld <= LocalToWorld[Parent] * LocalToParent

![img](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20200903203338.png)

用户可以将PostRotation component直接作为四元数写入。但是，如果首选Euler接口，则每个旋转顺序都可以使用component，这将导致写入PostRotation component（如果存在）。

- [PostRotationEulerSystem] PostRotation <= PostRotationEulerXYZ
- [PostRotationEulerSystem] PostRotation <= PostRotationEulerXZY
- [PostRotationEulerSystem] PostRotation <= PostRotationEulerYXZ
- [PostRotationEulerSystem] PostRotation <= PostRotationEulerYZX
- [PostRotationEulerSystem] PostRotation <= PostRotationEulerZXY
- [PostRotationEulerSystem] PostRotation <= PostRotationEulerZYX

例如，如果存在以下components...

| Parent (Entity) | Child (Entity)           |
| :-------------- | :----------------------- |
| LocalToWorld    | LocalToWorld             |
| Translation     | LocalToParent            |
| Rotation        | Parent                   |
| Scale           | PreviousParent*          |
| Child*          | Translation              |
|                 | CompositeRotation        |
|                 | Rotation                 |
|                 | RotationPivotTranslation |
|                 | RotationPivot            |
|                 | PostRotation             |
|                 | RotationEulerXYZ         |
|                 | Scale                    |
|                 | PostRotationEulerXYZ     |

...然后transform system将会是：

1. [TRSToLocalToWorldSystem] Parent: Write LocalToWorld as defined above in "Non-hierarchical Transforms (Basic)"
2. [PostRotationEulerSystem] Child: Write PostRotation <= PostRotationEulerXYZ
3. [RotationEulerSystem] Child: Write Rotation <= RotationEulerXYZ
4. [CompositeRotationSystem] Child: Wirte CompositeRotation <= RotationPivotTranslation * RotationPivot * Rotation * PostRotation * RotationPivot^-1
5. [TRSToLocalToParentSystem] Child: Write LocalToParent <= Translation * CompositeRotation * Scale
6. [LocalToParentSystem] Child: Write LocalToWorld <= LocalToWorld[Parent] * LocalToParent

![img](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20200903203510.png)

多个PostRotationEulerXXX component与同一entity相关联是一个错误，但是定义了结果。将应用按优先级顺序找到的第一个。该顺序是：

1. PostRotationEulerXYZ
2. PostRotationEulerXZY
3. PostRotationEulerYXZ
4. PostRotationEulerYZX
5. PostRotationEulerZXY
6. PostRotationEulerZYX

对于更复杂的Scale要求，可以使用CompositeScale（float4x4） component替代Scale（或NonUniformScale）。

对Scale或NonUniformScale有效的所有组合对CompositeScale也有效。即

- [TRSToLocalToParentSystem] LocalToParent <= Translation * Rotation * CompositeScale
- [TRSToLocalToParentSystem] LocalToParent <= Rotation * CompositeScale
- [TRSToLocalToParentSystem] LocalToParent <= CompositeScale
- [TRSToLocalToParentSystem] LocalToParent <= Translation * CompositeRotation * CompositeScale
- [TRSToLocalToParentSystem] LocalToParent <= CompositeRotation * CompositeScale
- [TRSToLocalToParentSystem] LocalToParent <= Translation * ParentScaleInverse * Rotation * CompositeScale
- [TRSToLocalToParentSystem] LocalToParent <= Translation * ParentScaleInverse * CompositeRotation * CompositeScale
- [TRSToLocalToParentSystem] LocalToParent <= ParentScaleInverse * Rotation * CompositeScale
- [TRSToLocalToParentSystem] LocalToParent <= ParentScaleInverse * CompositeRotation * CompositeScale
- [TRSToLocalToParentSystem] LocalToParent <= ParentScaleInverse * CompositeScale

用户代码可以将CompositeScale component直接写为float4x4。但是，如果首选Maya/FBX风格的接口，则可以使用可以写入CompositeScale component（如果存在）的component。



CompositeScale = ScalePivotTranslation * ScalePivot * Scale * ScalePivot^-1 CompositeScale = ScalePivotTranslation * ScalePivot * NonUniformScale * ScalePivot^-1



如果同时存在ScalePivotTranslation（float3），ScalePivot（float3），Scale（float）component与CompositeScale component的任何组合，则transform system将组合这些components并写入CompositeScale。

或者，如果同时存在ScalePivotTranslation（float3），ScalePivot（float3），NonUniformScale（float3）component和CompositeScale component的任意组合，则transform system将组合这些components并写入CompositeScale。

具体来说，这些components组合中的每一个都将写入CompositeRotation：

- [CompositeScaleSystem] CompositeScale <= ScalePivotTranslation
- [CompositeScaleSystem] CompositeScale <= ScalePivotTranslation * ScalePivot * Scale * ScalePivot^-1
- [CompositeScaleSystem] CompositeScale <= ScalePivotTranslation * Scale
- [CompositeScaleSystem] CompositeScale <= ScalePivot * Scale * ScalePivot^-1
- [CompositeScaleSystem] CompositeScale <= Scale
- [CompositeScaleSystem] CompositeScale <= ScalePivotTranslation * ScalePivot * NonUniformScale * ScalePivot^-1
- [CompositeScaleSystem] CompositeScale <= ScalePivotTranslation * Scale
- [CompositeScaleSystem] CompositeScale <= ScalePivot * NonUniformScale * ScalePivot^-1
- [CompositeScaleSystem] CompositeScale <= NonUniformScale

如果指定ScalePivot且不使用Scale或NonScale中的任何一个，则NonUniformScale都没有其他影响，对CompositeScale也没有其他影响。

例如，如果存在以下components...

| Parent (Entity) | Child (Entity)           |
| :-------------- | :----------------------- |
| LocalToWorld    | LocalToWorld             |
| Translation     | LocalToParent            |
| Rotation        | Parent                   |
| Scale           | PreviousParent*          |
| Child*          | Translation              |
|                 | CompositeRotation        |
|                 | Rotation                 |
|                 | RotationPivotTranslation |
|                 | RotationPivot            |
|                 | PostRotation             |
|                 | RotationEulerXYZ         |
|                 | Scale                    |
|                 | PostRotationEulerXYZ     |
|                 | CompositeScale           |
|                 | ScalePivotTranslation    |
|                 | ScalePivot               |

...然后transform system将会是：

1. [TRSToLocalToWorldSystem] Parent: Write LocalToWorld as defined above in "Non-hierarchical Transforms (Basic)"
2. [PostRotationEulerSystem] Child: Write PostRotation <= PostRotationEulerXYZ
3. [RotationEulerSystem] Child: Write Rotation <= RotationEulerXYZ
4. [CompositeRotationSystem] Child: Wirte CompositeRotation <= RotationPivotTranslation * RotationPivot * Rotation * PostRotation * RotationPivot^-1
5. [TRSToLocalToParentSystem] Child: Write LocalToParent <= Translation * CompositeRotation * Scale
6. [LocalToParentSystem] Child: Write LocalToWorld <= LocalToWorld[Parent] * LocalToParent

![img](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20200903203828.png)

...然后transform system将会是：

1. [TRSToLocalToWorldSystem] Parent: Write LocalToWorld as defined above in "Non-hierarchical Transforms (Basic)"
2. [PostRotationEulerSystem] Child: Write PostRotation <= PostRotationEulerXYZ
3. [RotationEulerSystem] Child: Write Rotation <= RotationEulerXYZ
4. [CompositeScaleSystem] Child: Write CompositeScale <= ScalePivotTranslation * ScalePivot * Scale * ScalePivot^-1
5. [CompositeRotationSystem] Child: Wirte CompositeRotation <= RotationPivotTranslation * RotationPivot * Rotation * PostRotation * RotationPivot^-1
6. [TRSToLocalToParentSystem] Child: Write LocalToParent <= Translation * CompositeRotation * Scale
7. [LocalToParentSystem] Child: Write LocalToWorld <= LocalToWorld[Parent] * LocalToParent

![img](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20200903203843.png)

------

## 第6节：Custom Transforms (Advanced)

有两种编写与transform system完全兼容的用户定义转换的方法。

1. 重写trasnform components
2. 拓展trasnform components

## 重写trasnform components

定义了一个用户 component（UserComponent）并将其添加到LocalToWorld WriteGroup中，如下所示：

```cs
[Serializable]
[WriteGroup(typeof(LocalToWorld))]
struct UserComponent : IComponentData
{
}
```

Overriding transform components意味着不可能有其他扩展。用户定义的transform是指定用户 component可以发生的唯一转换。

在UserTransformSystem中，使用默认query方法来请求对LocalToWorld的写权限。

例如

```cs
    public class UserTransformSystem : SystemBase
    {
        protected override void OnUpdate()
        {
            Entities
                .ForEach(
                    (ref LocalToWorld localToWorld, in UserComponent userComponent)=>{
                        localToWorld.Value = ... // Assign localToWorld as needed for UserTransform
                    }).ScheduleParallel();
        }
    }
```

写入UserTo的transform system将忽略所有写入LocalToWorld的其他transform components。

例如，如果存在以下components...

| (Entity)      |
| :------------ |
| LocalToWorld  |
| Translation   |
| Rotation      |
| Scale         |
| UserComponent |

...然后：

- [TRSToLocalToWorldSystem]不会在此entity上运行
- [UserTransformSystem]将在此entity上运行

但是，如果两个不同的system都覆盖LocalToWorld并且同时存在两个component，则可能导致意外行为。例如

例如，如果还有：

```cs
    [Serializable]
    [WriteGroup(typeof(LocalToWorld))]
    struct UserComponent2 : IComponentData
    {
    }
```

和等效的system：

```cs
    public class UserTransformSystem2 : SystemBase
    {
        protected override void OnUpdate()
        {
            Entities
                .ForEach(
                    (ref LocalToWorld localToWorld, in UserComponent2 userComponent2)=>{
                        localToWorld.Value = ... // Assign localToWorld as needed for UserTransform
                    }).ScheduleParallel();
        }
    }
```

然后，如果存在以下components...

| (Entity)       |
| :------------- |
| LocalToWorld   |
| Translation    |
| Rotation       |
| Scale          |
| UserComponent  |
| UserComponent2 |

两个systems都将尝试写入LocalToWorld，这可能会导致意外行为。在确定的上下文中这可能不是问题，但是在多线程中就说不准了。

## Extending transform components

为了确保多个重写的transform components可以以定义良好的方式进行交互，可以使用WriteGroup query仅显式地匹配所请求的component。

例如，如果存在：

```cs
    [Serializable]
    [WriteGroup(typeof(LocalToWorld))]
    struct UserComponent : IComponentData
    {
    }
```

以及一个基于LocalToWorld的WriteGroup进行过滤的system：

```cs
    public class UserTransformSystem : SystemBase
    {

        protected override void OnUpdate()
        {
            Entities
                .WithEntityQueryOptions(EntityQueryOptions.FilterWriteGroup)
                .ForEach(
                    (ref LocalToWorld localToWorld, in UserComponent userComponent)=>{
                        localToWorld.Value = ... // Assign localToWorld as needed for UserTransform
                    }).ScheduleParallel();
        }

    }
```

UserTransformSystem中的m_Query仅与明确提到的components匹配。

例如，以下与match匹配并包含在EntityQuery中：

| (Entity)      |
| :------------ |
| LocalToWorld  |
| UserComponent |

但下面这样却不会：

| (Entity)      |
| :------------ |
| LocalToWorld  |
| Translation   |
| Rotation      |
| Scale         |
| UserComponent |

隐含的期望是UserComponent是要写入LocalToWorld的一组完全匹配的要求，因此不应存在同一WriteGroup中的其他（未声明的）components。

但是，通过添加到query中，UserComponent system可能明确支持它们，如下所示：

```cs
    public class UserTransformExtensionSystem : SystemBase
    {
        protected override void OnUpdate()
        {
            Entities
                .WithEntityQueryOptions(EntityQueryOptions.FilterWriteGroup)
                .ForEach(
                    (ref LocalToWorld localToWorld, 
                     in UserComponent userComponent,
                     in Translation translation,
                     in Rotation rotation,
                     in Scale scale) => {
                        localToWorld.Value = ... // Assign localToWorld as needed for UserTransform
                    }).ScheduleParallel();
        }
    }
```

同样，如果还有其他内容：

```cs
    [Serializable]
    [WriteGroup(typeof(LocalToWorld))]
    struct UserComponent2 : IComponentData
    {
    }
```

并且有：

| (Entity)       |
| :------------- |
| LocalToWorld   |
| UserComponent  |
| UserComponent2 |

上面定义的UserTransformSystem将不匹配，因为没有明确提到UserComponent2，它位于LocalToWorld WriteGroup中。

但是，可以创建一个明确的query，该query可以解决该问题并确保行为已正确定义。如：

```cs
    public class UserTransformComboSystem : SystemBase
    {
        protected override void OnUpdate()
        {
            Entities
                .ForEach(
                (ref LocalToWorld localToWorld, 
                 in UserComponent userComponent,
                 in UserComponent2 userComponent2)=>{
                    localToWorld.Value = ... // Assign localToWorld as needed for UserTransform
                }).ScheduleParallel();
        }
    }
```

然后是以下system（或等效system）：

- UserTransformSystem (LocalToWorld FilterWriteGroup:UserComponent)
- UserTransformSystem2 (LocalToWorld FilterWriteGroup:UserComponent2)
- UserTransformComboSystem (LocalToWorld FilterWriteGroup:UserComponent, UserComponent2)

将全部并行运行，query并在各自的component archtype上运行，并具有明确定义的行为。

------

## 第7节：Relationship to Maya transform nodes

有关Maya变换节点的参考，请参见：[https](https://download.autodesk.com/us/maya/2010help/Nodes/transform.html) : [//download.autodesk.com/us/maya/2010help/Nodes/transform.html](https://download.autodesk.com/us/maya/2010help/Nodes/transform.html)

Maya转换矩阵定义为：

> 矩阵= SP ^ -1 * S * SH * SP * ST * RP ^ -1 * RA * R * RP * RT * T

可以将它们映射为transform component，如下所示：

| Maya                       | Unity                    |
| :------------------------- | :----------------------- |
| T                          | Translation              |
| (RT * RP * R * RA * RP^-1) | CompositeRotation        |
| RT                         | RotationPivotTranslation |
| RP                         | RotationPivot            |
| R                          | Rotation                 |
| RA                         | PostRotation             |
| (ST * SP * S * SP^-1)      | CompositeScale           |
| ST                         | ScalePivotTranslation    |
| SP                         | ScalePivot               |
| SH                         | --- Unused ---           |
| S                          | NonUniformScale          |

# 渲染

Hybrid.Rendering包提供ECS system来渲染3D对象。

有关当前与DOTS兼容的渲染API的信息，请参见[DOTS Hybrid Renderer](https://docs.unity3d.com/Packages/com.unity.rendering.hybrid@latest/index.html)。

# 游戏代码中的常见模式

## 使用Entities.ForEach构建代码

[Entities.ForEach](https://docs.unity3d.com/Packages/com.unity.entities@0.14/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Entities)允许您编写处理一组entities的内联job代码。在组织代码时，它可以帮助将功能封装到方法和结构中。以下模式提供了执行此操作的方法：

- [静态方法](https://docs.unity3d.com/Packages/com.unity.entities@0.14/manual/gp_common_patterns.html#static-methods)
- [封装数据和方法](https://docs.unity3d.com/Packages/com.unity.entities@0.14/manual/gp_common_patterns.html#encapsulation)



### 从Entities.ForEach调用静态方法

此模式可帮助您在多个地方重用功能。它还可以帮助简化复杂system的结构，并使代码更具可读性。

您可以将静态方法用作ForEach lambda函数，如以下示例所示。称为[Burst](https://docs.unity3d.com/Packages/com.unity.burst@latest/index.html)的静态函数将被[Burst](https://docs.unity3d.com/Packages/com.unity.burst@latest/index.html)编译（如果该函数与Burst不兼容，则将[.WithoutBurst（）](https://docs.unity3d.com/Packages/com.unity.entities@0.14/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Entities)添加到[Entities.ForEach](https://docs.unity3d.com/Packages/com.unity.entities@0.14/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Entities)构造中）。

```csharp
public class RotationSpeedSystem_ForEach : SystemBase
{
    protected override void OnUpdate()
    {
        float deltaTime = Time.DeltaTime;
        Entities
            .WithName("RotationSpeedSystem_ForEach")
            .ForEach((ref Rotation rotation, in RotationSpeed_ForEach rotationSpeed) 
                => DoRotation(ref rotation, rotationSpeed.RadiansPerSecond * deltaTime))
            .ScheduleParallel();
    }

    static void DoRotation(ref Rotation rotation, float amount)
    {
        rotation.Value = math.mul(
            math.normalize(rotation.Value), 
            quaternion.AxisAngle(math.up(), amount));
    }
}
```

有关创建ECS系统的更多信息，请参阅：

- [Systems](https://docs.unity3d.com/Packages/com.unity.entities@0.14/manual/ecs_systems.html)
- [SystemBase](https://docs.unity3d.com/Packages/com.unity.entities@0.14/api/Unity.Entities.SystemBase.html)



### 将数据和方法封装为捕获的值类型：

此模式可帮助您组织数据并将它们放在一个单元中一起工作。

您可以定义一个结构，该结构声明数据的本地字段以及[Entities.ForEach](https://docs.unity3d.com/Packages/com.unity.entities@0.14/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_Entities)调用的方法。在系统[OnUpdate（）](https://docs.unity3d.com/Packages/com.unity.entities@0.14/api/Unity.Entities.SystemBase.html#Unity_Entities_SystemBase_OnUpdate_)函数中，可以将结构的实例创建为局部变量，然后按以下示例所示调用该函数：

```csharp
public class RotationSpeedSystem_ForEach : SystemBase
{
    struct RotateData
    {
        float3 m_Direction;
        float m_DeltaTime;
        float m_Speed;

        public RotateData(float3 direction, float deltaTime, float speed) 
            => (m_Direction, m_DeltaTime, m_Speed) = (direction, deltaTime, speed);
        public void DoWork(ref Rotation rotation) 
            => rotation.Value = math.mul(math.normalize(rotation.Value), 
                quaternion.AxisAngle(m_Direction, m_Speed * m_DeltaTime));
    }

    protected override void OnUpdate()
    {
        var rotateUp = new RotateData(math.up(), Time.DeltaTime, 3.0f);
        Entities.ForEach((ref Rotation rotation) 
            => rotateUp.DoWork(ref rotation))
            .ScheduleParallel();
    }
}
```

**注意：**此模式将数据复制到您的job结构中（如果与`.Run`一起使用，则将其备份）。如果使用非常大的job结构来执行此操作，由于结构的复制，可能会产生一些性能开销。在这种情况下，这可能表明您的job应分为多个较小的jobs。
