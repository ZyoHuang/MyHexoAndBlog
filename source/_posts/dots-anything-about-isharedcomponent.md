---
title: 关于ISharedComponent的一切
date:
updated:
tags:
categories:
keywords:
top_img:
cover:
katex: true
aplayer:
---
<meta name="referrer" content="no-referrer" />

## 前言

ECS中的ISharedComponent是一个非常重要的概念，但是理解起来却不是这么容易，前阵子找到一篇讲ISharedComponent的文章，感觉例子举的很好，讲的也很透彻，故翻译出来分享给大家。

本文翻译自：[https://gametorrahod.com/everything-about-isharedcomponentdata/](https://gametorrahod.com/everything-about-isharedcomponentdata/)

# 正文

这是ECS最容易被误解的功能之一。当您不知道它是如何设计的时候，通常会出现诸如“如何在工作中获取SCD数据？我不能？那有什么卵用！”这样的问题。因此，让我们开始了解它是如何工作的。

## 数据共享并不那么ECS

众所周知，ECS会紧密排布entity数据成为chunk。同样，高性能C＃（HPC＃）的限制是得确保您没有通往外界的任何“门户”。`static`无法使用。禁用别名。分析器无法通过公有字段引入job的引用类型。它也不会让您将指针潜入`IComponentData`其中。另外，存储器必须是线性的。

“共享”的概念听起来根本不像ECS。在ECS中，可以从chunk中获取任何用来工作的数据。您不会在任何地方跳转，这就是“默认情况下的性能”的来源。可预测，可优化，并且缓存命中率很高。

但是，如果我告诉你，我们可以有一个真正的与`Entity`**相关联的**共享值？那不是违反ECS原则的吗？如果有人更改了主线程中的共享值，或者我们正在job中迭代的`Entity`突然得到了其他值又该怎么办？

事实证明`ISharedComponentData`，这样做有一定的限制，使其可以与ECS一起生活。

## 指针：SCD索引

如何仅将数据存储在一个地方并“共享”到多个`Entity`？以与C++等语言中的指针相同的方式，我们将数据存储在其他位置，但只给出一个简单的数字作为地址。

但是，通过解指针（C++中的`*`运算符）可以很容易地将指针作为内存地址跳转到真实对象。您可能想到了，ECS是没有办法让您做到这一点的。

使用SCD索引，您可以通过向`EntityManager`查询该数字来完成相同的操作。`EntityManager`将共享数据的真实值保存在每个world中的超大的`List<object>`中。它给出的SCD索引只是这个法外之地的List的`object`索引。如果您想要对象，只需告诉`EntityManager`索引即可。很简单吧？

## 它不能在job中工作

`EntityManager`是唯一的主线程。在job内部无法使用。我看到太多的程序员认为他们可以将SCD**值**用作繁重的计算工作中的一部分，但这是不可能的。（但是您在job中仍然有SCD索引！没用吗？也不完全是。）

即使有可能也会对性能造成很大的影响。

## SCD可以存储任何内容

我提到的`List<object>`是共享组件数据的数据存储。也就是说，你可以保存`int`，`GameObject`，`Material`，`EpicMonster : MonoBehaviour`。。。引用类型也是可能的。什么都行！

现在，你需要节省空间。想象一下，您获取了一个`struct`他有5个`float3`字段。该数据适合直接放在chunk中每个`Entity`上。但是，如果你将他们存储为一个`ISharedComponentData`，那么你就将5 $*$ `float3`$*$ entity个数的数据量转换为一个存储在chunk头部的int数据了（SCD 索引）

### 取回SCD值时实际发生了什么

当使用类似`GetSharedComponentData<T>(entity)`的东西时，它将如何与那个大`List<object>`交互？每个`T`类型不是必须有多个列表吗？或者其他的东西？

获取实际上非常简单。该`entity`会知道自己的chunk。该chunk具有多个SCD索引，具体数量取决于`ISharedComponentData`的索引数。用`T`就可以获取正确的索引。SCD索引直接插入`List<object>`到列表中作为索引器。然后`object`被**转换为**了`T`。（如`(T)scdValue`）

## 创建SCD类型

当然，您可以存储任何东西，但是您将使用ECS语言进行处理。

如果要存储一个`Material`并将其共享给多个`Entity`，则必须执行以下操作：

```csharp
public struct SharedMaterial : ISharedComponentData
{
	public Material material;
}
```

如果您要存储`int`并共享它，则必须执行以下操作：

```csharp
public struct Measure : ISharedComponentData
{
    //Which music measure since the beginning of song this note belongs to
    public int measure;
}
```

有点麻烦但是

- 您可以存储多件东西并将其称为一个单元。

## SCD索引按块

与其将这个SCD索引提供给每个`Entity`对象，不如说是chunk所需要的，这允许很多`Entity`与一个共享值关联起来，因为`Entity`一定会知道自己的块。

**关联**是关键字。共享值不在chunk中。只是SCD索引。

然后，这直接说明了为什么必须指定SCD类型：一个chunk与一个archetype相关联。现在，SCD已成为archetype的一部分！因此，根据此chunk中有多少种SCD，chunk头上将具有同等数量SCD索引。

此外，即使您从未在chunk迭代job中获得SCD的真正值，chunk迭代也为您提供了`ArchetypeChunkSharedComponentType`。类型本身就非常有用，例如，检查chunk上类型的存在，甚至检查SCD**索引**在chunk上是否为某个数字。

## 智能自哈希确定值的唯一性

这是关于SCD的最棒的事情之一。

考虑传统的值共享，您在C++中声明`int` = 555。您希望将`int`共享给多个对象。您不会将555给所有对象，因为那只是一个数字副本，与“共享”的定义相去甚远。您将需要询问该`int`地址并分发。当您更改原始`int`时，所有对象都可以获取更新。

在Unity ECS中，你可能会说，对一个Entity执行`SetSharedComponentData`（`int = 555`）。然后将此555添加到`List<object>`然后生成一个SCD索引，并作为关联将其返回并放到chunk上。此时，它还会在字典中创建一个特殊记录，该记录记住哈希索引。

在另一个`Entity`，你再一次对一个Entity执行`SetSharedComponentData`（`int = 555`）。它不知为何知道555已经存在于`List<object>`，并且返回相同的SCD索引，而不再将555添加到列表中了！！它确实是共享的，但是很神奇。

由于使用了的词典，它可以立即知道该哈希存在于SCD`List<object>`数据库中。该字典使散列索引为O（1）复杂度。这使得`List<object>`索引值也变为O（1）复杂度。没有线性搜索或其他任何东西。

换句话说，`ISharedComponentData`通过值散列使值类型自动像引用一样。对于哈希算法，它递归地向下钻取所有字段以“您的事物”以检查其是否相等。如果找到了引用类型，它会停止向下钻取并对该引用类型变量的指针进行哈希处理。

## 您永远不会“更改” SCD值，只需交换即可。

但是，ECS不允许发生“魔术”。您不能要求`EntityManager`**将SCD数据库**中的555值更改为666，而以某种方式（`GetSharedComponentData`）获得555的SCD索引的每个对象都将改为666 。您只能将一个`Entity`或整个chunk设置为某个值。该值获得引用计数+1。而且旧值是将引用计数-1移至实际删除（如果引用计数为零的话）。

您可以正确地将其称为“交换SCD值”来替代编辑SCD值。您可以通过交换SCD值，以使旧的值消失，先调用`EntityQuery`然后将过滤器设置为要交换的旧值，最后使用`EntityManager.SetSharedComponentData`和`EntityQuery`以更改为新值。如果您的查询覆盖了所有旧引用，则旧引用的所有引用计数应立即减少为零。

但是，如果它是您存储在SCD中的引用类型，则可以通过保留该引用并更改其中的内容来施展魔法。一个示例是带有`Material`的SCD 。您可以在为其分配了很长的时间后使用`SetSharedComponentData`更改`Material`的颜色，然后`Entity`通过引用类型（如果有的话）要求获得新的颜色。（但是，无法将`Material`突然链接到另一个实例，就像555无法突然更改为666 ）

## 当没有人持有该SCD索引时，SCD数据将自行删除

像引用计数一样，它知道在引用计数达到0时即删除自身，也就是说，不再有chunk使用该SCD索引。仅通过`null`元素即可完成`List<object>`的删除操作。因此，所有发生在这个`List<object>`操作只有`.Add`。从不`.Remove`。这样，我们就无需去更新所有chunk上的所有现有索引，确保它们始终可用。

然后，哈希索引字典也将删除该哈希条目，因此当该哈希再次出现时，它将知道该值再次是新的。（但是新条目将一直`.Add`到List末尾，而忽略您可能已经通过`null`操作创建的漏洞。）

### `List<object>`数据结构

至此，您可能已经可以想象出`List<object>`它的外观了。它甚至没有保持彼此接近的相同类型。只需按时间顺序添加到末尾即可。

并记住，每个`object`都是装箱值，是指向某个地址的指针。即使相邻的SCD`object`只是包含一个简单的`int`，它也不是真的在内存中彼此相邻。因此，如果您试图修改SCD**值**，请意识到，SCD值会是一个缓存未命中的对象。（因此，请尽可能只处理SCD**类型**）

## 练习

我们有这些：

```csharp
public struct A : IComponentData { }
public struct B : ISharedComponentData { public int value; }
```

- 创建5个archetype为`A`的实体。该chunk现在是具有5个entities的archetype `A`的1个chunk。每个entity各被命名`v w x y z`
- 将`B`SCD(`new B { value = 555 }`)添加到entity`v`和`w`。仅有一个数字555被添加到`List<object>`,并且为两个添加返回相同的SCD索引。我们现在有2个chunk，archetype的第一个chunk `A`减少为3个entities `x y z`。具有archetype `AB`的新chunk具有2个实体`v w`。
- 使用`SetSharedComponentData`(与`new B { value = 666 }`)于entity`w`。添加666到`List<object>`，然后返回新的SCD索引。现在`w`不再与`v`SCD索引位于同一chunk中，因为SCD索引**每个类型**（类型`B`）**每个chunk中**只有一个数字。现在，我们有3个chunk，具有`x y z`的chunk，具有`v`的chunk，具有`w`的chunk。

### 通过不同的SCD值进行chunk分割/分段

当然，添加新的SCD类型会更改archetype并拆分chunk。这很经典，因为它的作用类似于`IComponentData`类型添加。但是使用“每个chunk**一个**索引”规则中的SCD时，更改SCD**值**也会导致chunk分裂，因为它会获得该块的新SCD索引！

（如果它允许每个chunk使用多个SCD索引，则我们还必须考虑一种方法来说明该块中的哪个`Entity`获得了哪个索引，这在设计IMO中是一团糟，因此非常好用。）

即使SCD是同一类型，这种“chunk分割技术”也已在hybrid renderer package中广泛使用。渲染时，您通常希望处理一组唯一的唯一性`Mesh`，`Material`然后一次进行下一个。与其尝试花哨的算法进行排序和迭代，不如让我们拥有像这样的SCD来进行自动分类。

```cs
public struct C : ISharedComponentData { 
    public Material mat; 
    public Mesh mesh;
}
```

chunk具有固定大小，您可能会认为这是浪费空间。但具有多chunk并不全是坏事，就像这样`IJobChunk`或`IJobForEach`每个chunk可以并行工作在不同的工作线程！

同时，这意味着SCD**值的**任何变化都将导致结构变化。而更改`IComponentData`值对结构没有任何作用。无需移动到任何地方。

实际上，这似乎是SCD的重点，而不是打算共享数据。引用[此片段](https://forum.unity.com/threads/need-way-to-evaluate-animationcurve-in-the-job.532149/#post-4525402)：

> 共享组件数据实际上是用于将您的entity细分为强制chunk组。我觉得这个名字很不幸。因为实际上，如果将其用作数据共享机制，您通常会大吃一惊，因为常常会得到的块太小。

### 用SCD不移动数据的某种方法

```cs
EntityManager.AddSharedComponentData<T>(EntityQuery entityQuery,...
EntityCommandBuffer.AddSharedComponentData<T>(EntityQuery entityQuery,...
```

使用`EntityQuery`重载时，您对每个chunk中的每个entity（而不是每个`Entity`)。通过这样做，它只修改了chunk头，允许所有entities保持原样。`RemoveComponent`也适用于SCD，它也带有`EntityQuery`重载。在交换（更改）SCD值之前，我已经提到了这一点。

## SCD版本号

因为您不能像前面几节所述那样更改SCD数据，所以您可能会认为SCD类型上没有版本号的概念。（什么是版本号？[http://gametorrahod.com/designing-an-efficiency-system-with-version-numbers/](http://gametorrahod.com/designing-an-efficient-system-with-version-numbers/)）

```cs
- EntityManager -

public int GetSharedComponentOrderVersion<T>(T sharedComponent) where T : struct, ISharedComponentData

- ArchetypeChunk -

public bool DidChange<T>(ArchetypeChunkSharedComponentType<T> chunkSharedComponentData, uint version) where T : struct, ISharedComponentData
```

原来**是**每个chunk每个SCD类型的版本号，就像正常的类型！但是，由于SCD无法更改其自身的数据，因此它能够跟踪在chunk上发生的**所有**结构更改。但是，这与**更改**SCD值并不完全无关。如您所知，更改值**将**导致结构变化。因此，该版本号可以对此进行跟踪。但是，添加`IComponentData`到SCD类型的chunk也会增加该chunk上的所有SCD版本号。

## 获得实际SCD值的方法

### `EntityManager.GetSharedComponentData <SCDTYPE>（entity）`

直截了当。在`Entity`将访问其自己的SCD索引chunk然后向`EntityManager`查询实际值。

### `EntityManager.GetAllUniqueSharedComponentData <T>（List <T> sharedComponentValues，List <int> sharedComponentIndices）`

此方法相当粗暴，因为它返回`List<object>`至少约束为一种SCD类型的中每个存储的SCD 。您准备一个已分配`List`的数据，它将为您填充数据。（填充=附加，如果需要，您可以自行`Clear`。）

`List<int>`为你想对应的SCD索引。通常，您不只是想要所有唯一值，而是想知道每个值的索引表示形式，可以将其带入job以进行高级过滤。该两份list的`Count`将是相同的。

这是给您的一些陷阱。这个测试通过了。

```cs
private struct SampleScd : ISharedComponentData
{
    public int value;
}

[Test]
public void UniqueScd()
{
    var w = new World("Test World");
    var em = w.EntityManager;
    List<SampleScd> unique = new List<SampleScd>(8);
    em.GetAllUniqueSharedComponentData<SampleScd>(unique);
    Assert.That(unique.Count, Is.EqualTo(1));
}
```

还没有一个entity，我只是从一开始就请求所有唯一的SCD，为什么已经有一个东西了呢？由于某种原因，这个类型的default总是会在列表中，即使根本没有entity目前拥有SCD。

### `EntityManager.GetSharedComponentData <T>（int sharedComponentIndex）其中T：struct，ISharedComponentData`

这是一种相当严格的方法，因为它会要求您提供SCD索引。您可以找到类似的内容，`entityManager.GetAllUniqueSharedComponentData<T>(List<T> sharedComponentValues, List<int> sharedComponentIndices)`或者只是在chunk迭代时询问该chunk：`archetypeChunk.GetSharedComponentIndex<T>(ArchetypeChunkSharedComponentType<T> chunkSharedComponentData)`

进入此方法的索引直接是您的`object`索引器，然后将其简单地转换为`<T>`。

### Entities.ForEach（SCDTYPE scd，...）

这是获取SCD值的非常时髦的方法。只需`ISharedComponentData`在您的lambda结构中作为第一个参数（它仅支持1个SCD，并且它必须是第一个）`ForEach`只可以在主线程中工作，并且可以要求`EntityManager`为您准备很多SCD值。请记住，在进行`ForEach`过程中，SCD值可能会长时间保持不变，并且只有在过渡到新的chunk时才会改变。

### chunk.GetSharedComponentData（ArchetypeChunkSharedComponentType，EntityManager）

您也可以在执行chunk迭代时执行此操作。哈！您是否认为自己现在可以在job中获得共享值？它需要`EntityManager`作为最后一个参数来防止您这样做，因为您永远无法将整个EM（充满引用类型的东西）带到job中。

在这里，这是非常合乎逻辑的和准系统的。在`chunk`将只有一堆愚蠢的SCD索引。ACSCT知道如何从该chunk中选择要使用的特定SCD索引。`EntityManager`使用并从`List<object>`中返回东西。使SCD正常运行的所有3个基本要素都在同一行。

### 在job中

许多人抱怨我们为什么不能从工作中获得SCD值。阅读本文后，我希望您意识到为什么这与设计中的其他所有内容大相径庭。另外，您是否真的要访问诸如托管类型（`List<object>`）之类的东西，这些东西可能在job中“扭曲”到任何地方？

## Filtering

`ISharedComponentData`有最流行的**过滤**用途。过滤不需要访问`EntityManager`保持的实际值，因此即使在job内部它也非常高效且有用。

### 仅获取`EntityQuery`的子集及其archetypes

每个system都包含`EntityQuery`。您只处理`Entity`与query匹配的内容。那是您获得的第一个过滤器，因此您无需遍历宇宙中的所有内容。在`EntityQuery`返回**多chunks**，即使你可能只有一个纯`IComponentData`它的类型，因为chunk的存储空间有限，如果是满的，则它扩展到一个新的chunk。

但是，您可能希望从query中删除更多的chunk。假设在您的`EntityQuery`中类型之一是`ISharedComponentData`。在query中具有SCD类型是一个信号，即使这些chunk还远远不够满，您也可能会获得多个chunk，因为它们上的SCD索引不同。

现在，您可以再过滤一个级别：**丢弃没有所需SCD索引的块。**

### 共享组件数据过滤器： `eq.SetSharedComponentFilter<SCDTYPE>(scdValue)`

您可以在`EntityQuery`中添加或删除过滤器。当前只有两种过滤器：SCD过滤器和更改的过滤器。您只能同时激活一种`EntityQuery`类型。每个过滤器可以查找数量有限的component类型，当前为2。（对于更改的过滤器，您可以阅读[此内容](http://gametorrahod.com/designing-an-efficient-system-with-version-numbers#change-version-in-cg-setfilterchanged-componenttype-)）

当您做一些涉及`EntityQuery`获取数据的操作时，添加的过滤器将生效。移除过滤器（`eq.ResetFilter`）将恢复query，以返回与query archetype匹配的所有chunk。

您甚至不必说出过滤器的SCD索引。只需说出该**值**，它就会计算出您想要哪个SCD索引（由于使用了智能哈希技术），然后过滤掉不匹配该索引的其余chunks。如果archetype 中有2个`ISharedComponentData` ，2中SCD类型也有一个重载。

设置过滤器后，

- `To___` 方法将开始自动返回较少的数据。
- 从`EntityQuery`进行`CalculateLength`将被正确减少。
- `eq.CreateArchetypeChunkArray`将自动返回较少`NativeArray<ArchetypeChunk>`中的`ArchetypeChunk`。
- `Entities.With(eqWithScdFilter).ForEach(SCDTYPE scd, ...)` 还可以自动迭代较少的数据。

### SCD过滤 `IJobForEach`

您可以在`IJobForEach`的`Schedule`中使用`EntityQuery`，而不是在`IJobForEach`+ `[RequireComponentTag]`+ `[ExcludeComponent]`依赖`ref`类型。（它将忽略属性和`ref`类型贡献）。也可以在query中包括SCD类型，以仅获取具有该SCD类型的entities。

然后，如果您`EntityQuery`安装了过滤器，则可以轻松轻松地成功过滤SCD！（但是，没有得到值。只是根据值进行过滤。）

### 主线程chunk迭代中的SCD过滤`IJob`//`IJobChunk`

如果您想更加严格并且不使用`EntityQuery`或`IJobForEach`过滤器功能，您可以在遍历chunk时比较chunk上的SCD索引和您自己想要的SCD值的SCD索引。尽管在我看来，这是获得过滤器“aha”时刻的最佳方法，因为您需要手工完成所有工作！

1. 制作一个`EntityQuery`包含您的`ISharedComponentData`类型和其他数据。
2. `eq.CreateArchetypeChunkArray`，您将获得所有具有SCD**类型的**相关chunk，但仍包含带有您不想要的**`ISharedComponentData`索引的错误chunk**。您将获得archetype块数组（ACA）`NativeArray<ArchetypeChunk>`
3. 获取job外部的archetypr chunk类型，以便稍后在job中使用ACA。使用`GetArchetypeChunkSharedComponentType<T>()`得到ACSCT。
4. 因为当遍历每个对象`ArchetypeChunk`（可能在job中）时，它将仅具有SCD索引。您想知道该索引是正确还是错误。开始迭代之前，必须首先找到要与之比较的SCD值的SCD索引。不幸的是，没有“ SCD值索引”方法。您必须分配2个`List`然后使用`entityManager.GetAllUniqueSharedComponentData<T>(List<T> sharedComponentValues, List<int> sharedComponentIndices)`，然后搜索所需的SCD值，并在相同位置的其他list中获取其索引。这可以在实际job之前完成。
5. 您应该已经具备了以下三点：`ArchetypeChunkSharedComponentType`，`int`SCD索引表示您发现的SCD值更早，最后是chunk：`NativeArray<ArchetypeChunk>`（因为`IJobChunk`您已经在遍历每个`ArchetypeChunk`。）无论您是在主线程中`IJob`中还是`IJobChunk`中。
6. 请记住，所有`NativeArray<ArchetypeChunk>`中的chunks都具有所需的`ISharedComponentData`类型，但不一定要具有正确的SCD索引，因此我们正在寻找这种类型（本质上是“过滤”）。对于每个块（`ArchetypeChunk`）使用`chunk.GetSharedComponentIndex(ACSCT)`您将获得一个`int`。将返回`int`值与您之前准备的`int`值进行比较。如果相同，则您不仅找到了具有正确`ISharedComponentData`类型的chunk，而且还具有正确`ISharedComponentData`值的chunk。（即使因为缺少`EntityManager`而无法看到它在job中具有什么值。）然后，您可以决定跳过（过滤出）或处理（过滤进）chunk。

如果您使用这种硬核方式，您可能已经注意到一些独特的优势：因为SCD过滤器（`eq.SetFilter`）用作“过滤器”，也就是说，您只希望具有此SCD值。手动SCD过滤使您可以灵活地反转过滤器，或一次查找多个索引。（EQ滤波器一次最多可重载2次）

### SCD过滤的批处理EntityManager操作

这是我最喜欢的SCD过滤操作。

通常类似`AddComponentData`或`RemoveComponentData`或`DestroyEntity`在`EntityManager`上的东西是高消耗的因为结构性变化。

但是，如果您对chunk中的每个entity都执行相同的操作，则无需移动任何数据，因为我们可以改为更改chunk的标头。`Entity`通过使用`EntityManager`接受的重载来实现“整体操作”而不是每次操作`EntityQuery`。（[在此](http://gametorrahod.com/batched-operation-on-entitymanager/)了解更多信息）

谈到`EntityQuery`，如果当前已使用SCD过滤器过滤，则可以执行“过滤的整个chunk操作”！由于不同的SCD值将分隔多个chunk，因此批处理`EntityManager`操作可以知道要丢弃或处理哪个chunk是有道理的。

例如，如何使用 `entityManager.RemoveComponent(eq)`删除所有特定类型的SCD。**但**不是全部，只有具有特定SCD值的entities？只需在您的query上放一个SCD过滤器，然后将query扔到那里即可进行批量清除。

### 回顾

到了现在，您应该意识到“过滤”对“通过SCD值进行chunk分割”属性起作用，最后只是每个chunk的简单SCD索引比较。（根本没有entity访问！）

这与遍历所有entities并像LINQ一样取出某些entity完全不一样，而是遍历所有匹配的chunk并从其中删除一些chunk。

仅使用`Chilli : IComponentData`，假设其数据如此之大，以至于一个chunk只能包含100个带有`Chilli`的`Entity`。如果我有600`Entity`我只能从`EntityQuery`中得到6个`Chilli`chunk。

假设我有一个额外的名为`ChilliColor : ISharedComponentData`的SCD，有一个`enum`是说chilli是`Red`，`Yellow`或`Green`。（我们假定实体`Chilli`和`ChilliColor`仍然使chunk容量= 100），我们有50个`Yellow`，150个`Green`和400个`Red`。您能解释一下从`Chilli`和`ChilliColor`的`EntityQuery`中获得多少块吗？

答案是 ：

- 1：50/100 `Yellow`
- 第2块：100/100 `Green`
- 第3块：50/100 `Green`
- 4〜7块：100/100 `Red`

返回的块增加了SCD，每个块都只包含一种颜色的辣椒。在现实世界中，您可以想象它们按颜色令人满意地分组为<= 100的一堆，其中有些堆不是很饱满，但是出于整洁的颜色，我们对此表示满意。如果您`ChilliColor`只是正常`IComponentData`那将是6整堆五颜六色的烂摊子。

如果我将SCD值过滤器添加到`EntityQuery`（`Red`）中，会发生什么？因为过滤器只保留了那部分，所以现在我们只得到第4~7 chunk。它必须逐字地`for`循环通过chunk头7次，并在没有该SCD的情况下直接丢弃该chunk，而无需实际寻找任何entity，而且由于我们只保留一个整数SCD索引，因此检查chunk头速度也很快。这是“过滤”的真实身份。本质上，这只是对SCD索引的相等性测试。

如果我将所有辣椒添加`Rotten : ISharedComponentData`（`bool`（`true`或`false`））随机添加，您可以想象我们将得到大约15个chunk，然后我们可以放置一个不腐烂的SCD过滤器，以仅从`EntityQuery`获取剩下的优质辣椒，而不管颜色是多少。

## BlobBuilder和BlobAssetReference

这不是SCD，为什么要在本文中提到它？在此之前，我曾说过很多人希望以一种**每个`Entity`都**可以访问该资源的方式共享某些东西。这是您要查找的Blob数据。但是，让我们先看看替代方案。

您可以在job中复制一份`NativeArray<T>`，但是这数据**属于**谁？是保留它并开始job的system吗？这不仅是不是很ECS方式，因为该system现在具有数据，您无法将其与`Entity`关联，因为如果将`IComponentData`with`NativeArray<T>`作为`public`字段创建，它会抱怨您在那里有一个本地容器。

把`DynamicArray`作为一个`Entity`的新的component怎么样？那不是**共享的，**所以⑧行。

将Blob数据解救出来。本文不是为了这个目的，而是简短介绍`BlobBuilder`，您将以可以想象的任何形式分配内存。您应该在package中查看一下`BlobificationTests.cs`。

```cs
struct MyData
{
    public BlobArray<float> floatArray;
    public BlobPtr<float> nullPtr;
    public BlobPtr<Vector3> oneVector3;
    public float embeddedFloat;
    public BlobArray<BlobArray<int>> nestedArray;
}
```

然后，您可以从构建器那里获得`BlobAssetReference<MyData>`。这`BlobAssetReference`是`IComponentData`上的有效公共字段，但是当您通过引用将其分发给许多`Entity`时，实际上并没有复制分配的值。（就像您将看起来像一个`NativeArray`副本（因为他是一个struct）放入job，但是您只在其中复制一个包装的指针一样）。到这里，这是一个真正的共享数据。
