---
title: Game Engine EntityObject Models
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

本文内容来自：[https://www.youtube.com/watch?v=jjEsB611kxs&feature=youtu.be&ab_channel=BobbyAnguelov](https://www.youtube.com/watch?v=jjEsB611kxs&feature=youtu.be&ab_channel=BobbyAnguelov)

视频原作者还出了一个续集，主要是针对ECS空间结构的部分重讲，有兴趣可以去看看：[https://www.youtube.com/watch?v=fuiNOWEUnJ8&ab_channel=BobbyAnguelov](https://www.youtube.com/watch?v=fuiNOWEUnJ8&ab_channel=BobbyAnguelov)

# 正文

主要分为两大部分

第一部分：讨论当前游戏业界两种主要的对象/实体（object/entity）模型

- Game-Object/Entity Component模型
- Entity Component System模型

第二部分：讨论Kruger Entity模型作为一种可替代的方案

## Ye Olde Object Model

- 非常简单随性的方法
- 不容易拓展，也不容易复用
- 最后经常会有这些问题：
  - 深层级
  - 继承地狱
  - 重复代码
  - 所有标准OOP会遇到的问题

## 警告

Object/Entity模式不是必须践行的标准，每个引擎都有自己独特的味道（确实，味道很大）

## Object Component模型

### 介绍

本质上还是采用传统的模式，将共享的功能和数据拆分成组件，例如UE4，ubisoft，cryengine。。。

![image-20210110172054110](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20210110172054.png)

### Game Object

GameObject是一个包含一个components列表的对象。

理想情况下，gameobject只是一个容器（但这种情况并不常见）

GameObject有他们自己的init/update方法

可能会，也可能不会update他们的components

可以引用其他objects/components

### Components

可以包含数据和逻辑

- 有时它是纯逻辑的（例如事件总线（message bus）组件）
- 有时他是纯数据的

通常来说有它自己的update方法，可能是全局注册的，也可能是被父对象调用的

可以引用其他objects/components

### Hiearchies

层级关系一定不能坏掉

### 优点

Gameobject可以完整描述场景中某个元素

概念简单（理论上的）

容易为其创建工具

### 缺点

考虑每个东西Update顺序

引用/依赖地狱（在大部分Object Component模型中都存在循环引用）

对于大量的小对象缓存不友好

### 引用地狱的例子

下面是一个对象身上的components

![image-20210110183305372](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20210110183305.png)

他们会有两个主要问题

- Init顺序
- Update顺序和数据传送

#### Init例子

如果只是一个object，我们如何保证这些组件有下面的初始化顺序来保证正确的行为呢？

![image-20210110183809363](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20210110183809.png)

一些引擎的做法是Hack到一个多阶段的初始化过程中（比如UE4），但是这并不能完全解决问题，它会变成一场噩梦，在你增加越来越多的组件时，可能会陷入初始化顺序的循环依赖中。

当你在某个时刻遭遇了循环引用，该怎么解决呢？

可以派生一个新的object类型，在这个对象初始化的时候，手动执行组件初始化，在这个情况下，我们又回到了“Ye Olde Object Model”（之所以加双引号，我理解是因为现在这种做法是模块化的，原视频是except with the illusion of modularity，大家自行理解）

#### Update例子

![image-20210110185413444](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20210110185413.png)

我们发现Skeletal Mesh Component和Cloth Component循环依赖了

我们可以反转一些调用来解除这个循环依赖

![image-20210110185538284](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20210110185538.png)

现在显式依赖没有了，但是我们现在有了隐式依赖，例如可能会在一帧中对Skeletal Mesh Component进行多次Set Pose操作，然后需要对其余相关组件进行一系列操作。

这种情况很蛋疼，因为你不可能知道代码中有哪些地方调用到了这些内容。

有几种解决方案供参考：

- 在gameobject中显式定义update顺序（ye olde style）
- 显式指定每个component的update优先级（但如果在一个特殊的情形下需要一个特殊的update顺序呢）
- 结合所有相关component成一个（UE4的USkeletalMeshComponent就是这样，请千万不要这样做，仅仅因为更新顺序就把这些可能用不到的组件全部组合在一起。。。）

### 重用性和可拓展性

创建components的初衷是从一个class分离共享的方法到多个对象，这些对象可进行组合。

但Components不符合“单一职责原则”，他们通常用于执行特定领域的任务（例如更新布娃娃身体，执行动画逻辑等），他们同样也会执行数据传送和任务路由（直接从别的component拉取数据，直接update其他组件/对象）

因为隐式依赖的存在，一些组件一些情况下可能无法工作（没有它所依赖的组件）（如果没有Anim Component，不能有RagDoll Component，如果没有Skeletal Mesh Component，你不能有Anim Component），为了解决这个问题，可以把所有数据路由，传送，排序操作都放到父物体上，然后就又回到了“Ye Olde Model”，仅仅是多了一些模块性。但是这样的话会因为现代游戏疯狂的需求而导致父entity过于复杂。

### GameObject和单例

单例通常是一个下下策，被用于“直接引用/管理”多个gameobjects

通常说，没有干净的方式来注册component到单例，所以你将会有更多的循环依赖。

你同样需要考虑这些单例Init和Update的时机。

### 跨对象依赖

但对于直接访问另一个对象来说，单例是一个更加干净的方案。（如果实现的好的话）（比如A对B发起攻击，伤害结算应该交给DamageCalculateHelper，这样A,B都不互相依赖，而是依赖DamageCalculateHelper）

跨对象的访问也是使并行游戏的Update基本上不可能的原因。

### 总结

很多代码库使用这个模型，然后以component，object，单例的网状依赖郁郁而终。

大部分问题都是因为对于entity data的无限制的访问条件。

通常来说，对于下面的情况，没有系统性的解决方案

- Init和Update问题
- 并行和跨对象数据依赖/传送

在实践中，组件的重用性因为种种依赖会受到很大的限制。

## ECS模型

近年来正变得越来越流行，但并没有普及应用。（Stingray/Bitsquid engine，OverWatch，Unity实验阶段的ECS）

### 基础概念

Entity（例如一个gameobject）仅仅是一个Id，而不再是一个真正的对象。

Components是纯数据的容器。因为没有了隐式依赖，所以也没有了Init顺序问题，因为他们自身不Update，也没有Update的顺序问题。

System，所有的逻辑都被移动到这里，直接操作Component。

这是Object Component Model和ECS的对比

```cpp
//Object Component Model
struct A
{
	void Foo();
}

//ECS
struct A
{
	
}
void Foo(A& a);
```

### ECS Systems

全局Component管理者。Systems基本上都是单例。

所有Components的Update逻辑都在这，由于对Components数据的依赖，这仍是耦合的，Systems执行Update逻辑和传送数据。

基于components的type对component组进行操作，例如一个animation system需要Animation Component和Mesh Component。

System移除了隐式依赖。

### 限制

Component类型限制了System。

派生的Component将仍然匹配已有的System。

如果使用最上层的类型作为System匹配类型，那么在新派生一个component的时候，将需要更新所有已存在的Systems。

你可以使用一个DSL（domain-specific language）来定义System匹配类型，但这是另一大坨新东西了。

派生一个system的时候，你将会遇到完全相同的问题。

### 例子

System A需要匹配X和Y类型的components，现在你想要对其中几个Entities应用新的逻辑。

![image-20210111200116809](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20210111200122.png)

#### 方案A

创建一个新System B，他里面包含新的逻辑。

创建新I,J组件对应X,Y，然后System B的匹配类型就是I,J。

这个方法避免使用新的components

![image-20210111203942017](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20210111203942.png)

#### 方案B

创建一个新的Component Z，用于标识需要进行特殊逻辑的Entities，然后System A内部加上if判断，来决定是否需要执行新逻辑。（这个括号里的内容是译者自己的理解，个人认为一个新component+一个新system更优雅，第一，不用写if分支，第二，不需要装入多余的内存，但是仍然需要对System A做判断，因为XY是XYZ的子集，在query的时候包含XYZ的entities会被XY条件查询到。）

![image-20210111205227749](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20210111205227.png)

### 更多限制

每个Entity对于每种类型的component只支持单个实例。（例如，只有一个transform component，只有一个mesh component，[https://ourmachinery.com/post/should-entities-support-multiple-instances-of-the-same-component/](https://ourmachinery.com/post/should-entities-support-multiple-instances-of-the-same-component/)）

对于一个给定的entity，没有空间层次结构（空间层次结构需要用一个额外的component来描述）

### 没有空间层次结构

![image-20210111210402585](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20210111210402.png)

[https://takinginitiative.wordpress.com/2019/09/30/ecs-questions/](https://takinginitiative.wordpress.com/2019/09/30/ecs-questions/)

如果你的人物被多个东西组合而成：头，手，腿，脚等，每个mesh都是一个entity，那现在人物就被分为10个entities

如果拥有一个复杂的武器，你将会爆炸、、、（例如下图的枪，会变成20多个entities的组合）

![image-20210111212443340](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20210111212443.png)

没有空间层次结构会在每个system引入更多的复杂度（如果某个system修改了一个transform，你需要保证所有的system访问响应transform的准确性，即同步），但是这往往会带来很大的精神负担。

不能保证所有的entities都将在同一帧更新他们的positions。

对此 你有两种方案可选：

- 当一个transform更改时，更新所有的子对象
- 在必要的时候，计算整个世界的transform

这两个方案要求你每次都得查找所有子/父对象，如果你进行缓存的话，你就得更新这个缓存

### 更新顺序

[https://takinginitiative.wordpress.com/2019/11/09/more-ecs-questions/](https://takinginitiative.wordpress.com/2019/11/09/more-ecs-questions/)

对于更新来说，仍然存在内部entity依赖。

某些entities需要在其他那些依赖他们更新后状态的entities前完全更新。

举个例子

一个坦克，他有底盘，炮塔，和机枪，那么就必须先更新底盘，然后炮塔，然后机枪。

当然，可以选择回归“Ye Olde Model”

可以创建一个TankSystem来更新你整个坦克。

可以创建多个systems，然后对他们进行排序。

### ECS性能

ECS现在被默认为高性能的代名词？为什么呢？

Components的数据内存布局，通常来说，所有同类型的components被放在一个数组中，理想情况下，当开始轮询他们的时候（比如刚开始轮询前一部分），剩余部分也已经被加载到缓存中了。

这也是另一个不推荐派生components的原因（因为会被无用内存占据珍贵的缓存空间）。

让我们以一个开放世界的3A游戏为例。

我们会有一些东西，他们有大约250k的mesh实例。

他们当中有多少是动态的？可能有1k个？所以我们有250k个transform components，但只有其中1000个需要每帧update。

所以对于这1k个transform components，让我们假设他们可以用同样的方式来更新。

那我们如何保证他们在transform array中是连续的？我们是否需要一直对这个数组进行排序？我们是否需要分开存储动态/静态的transform？那我们又如何知道一个tranform是否是静态/动态的？

对于这些问题都有答案，但这些答案本身也有自己的权衡在里面。

Systems通常在单个线程上顺序运行，并允许用户在内部扩展。最基本的，并行所有匹配components类型的entities。
在许多情况下，这是有道理的，但在很多情况下却没有。游戏不只是粒子系统或者集群模拟！

### ECS != DOD

我们在很多底层系统中用到了DOD（data-oriented design），但ECS并不等于DOD。

## Kruger Entity模型

Kruger（KRG）是一个实验性/原型引擎。

各种想法的试验场。

本质上是一个gameobject component模型，对于同种类型的components的数目没有系统上的限制。提供components空间结构，components不再是纯数据。

移除所有components和entities的内部依赖，systems被用作链接components/entities。

Object的更新被严格控制。

### KRG Entity模型目标

易于理解

易于Debug

易于开发

需要根据处理器核心数量自行拓展，Entities需要进行原子更新（不能有内部依赖），Entity更新需要细粒度的并行（数据分割方法）

### 未来的趋势是并行

当今硬件趋于稳定的时钟频率

缓存大小和核心数量仍然在增加

6核处理器现在算是中等，10核以上的处理器才是更强的

IMO Ideal面向未来的游戏引擎是一种可以随核心数量线性扩展的引擎

多线程很重要！

### 多线程：锁

易于使用，预测行为

相对来说比较安全

相对来说很方便（如果使用得当的话）

会导致性能问题，错误的访问模式会导致强烈的线程竞争，死锁会导致用户层面的错误

### 多线程：数据分割

分割/分组数据到多个互不依赖的工作区，概念上等于并行

依赖工作区可能会更加复杂，可能需要代理数据或者数据拷贝

并不适用所有领域，一些情况下你不能移除内部依赖

### 多线程：无锁

请不要这样做！

这会使事情变得无比复杂

几乎不可能进行预测和Debug

可能会依赖于硬件

### 为什么需要混合多个模型

所有的模型都有自己的问题，唯一真正可行的方案是找到一个折中模型，能最小化每个模型的问题，

### Entity

Entities现在是真正的对象，每个entity有一个GUID，Entities有一个components的数组，Entities有一个systems数组。

### Entity Component

数据存储：定义了设置和资源的属性，反射系统自动生成序列化和资源加载代码。

Components在一些情况下可以指定为单例。

Components没有访问其他components的权利。

Components没有访问entity的权利。

Components没有默认的Update。

Components的继承是被允许的。

Components可以包含逻辑。

Components可以对他们自己的数据执行操作，例如Animation Graph Component，所有的graph逻辑都在这个组件上，我们有update graph和获取pose结果的方法。

Components可以被看成一个黑盒。

### Entity Component Life-Cycle

既然entity component可以引用资源，这些资源需要在这些components被使用之前被加载和初始化。

每个components可以处于下列状态的其中一个：

- Unloaded-被构造，并且所有属性都被设置了
- Loading-资源加载已经被请求，但是还没有完成
- Loaded-所有资源都加载成功
- Loading Failed-其中一个或多个资源加载失败
- Initialized-初始化完成后

![image-20210112100530994](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20210112100537.png)

### Component初始化

提供调用来分配/回收内部的临时数据，通常这些数据是资源依赖

对称的API设计，如果你初始化完毕，你也会强制终止

初始化不允许失败。

这里以Skeletal Mesh Component为例，他需要分配pose内存为mesh服务，在初始化过程中，我们可以扥配内存来保存临时骨骼transform，在终止过程中，我们释放已分配的骨骼transform内存。

![image-20210112101155280](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20210112101155.png)

### 基础的Entity Components

![image-20210112101236613](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20210112101236.png)

### 有空间的Entity Components

有一个本地tranform和本地bounds（OBB）

有一个世界transform和世界bounds（OOB），缓存来避免频繁的计算，**写入权限被下放到子类**。

有一个父空间的component指针和附加到的插槽id，对于子类来说是不可访问的。

有一个子空间components指针列表，对于子类来说是不可访问的。

空间的Components是唯一允许对其他components有依赖的components，否则没办法构建一个空间层次结构。

在一个层次结构中的所有的components必须归属同一个entity。

component依赖被用于保持层次结构更新。

### 空间层次结构

当我们update一个本地transform的时候，我们也update它的世界transform，世界transform基于父component重新计算，世界bounds基于父component更新，所有子世界transform也被更新。

这意味着世界transform总是最新的。

对于一个长的层次结构链来说，设置一个本地transformm是很昂贵的操作，所以我们基本没有很深的层次结构链，我们也基本不会每帧都多次更新transform。但这也确实是一个无法避免的问题。

每个entity可以有一个根空间component（root spatial component）。

如果这个根空间component被设置了，这个entity被认为是一个有空间的entity。

一个有空间的entity可能会被添加到另一个有空间的entity，但我们不允许entity的循环依赖。

有空间的entities对于update来说是需要特殊考虑的。

### Entity Update

Entity的更新工作是交于“System”的。

在entity system方面，我们与标准ECS概念类似。

一个entity system是一个component管理者。用于在特定类型的components之间追踪，更新和传送数据

Entity systems是唯一的update。

Kruger使用固定数量的工作线程（物理核心数-1），设置友好来避免频繁的线程切换。（Set core affinity to prevent threads jumping around cores）

在一帧里，有固定数量的update阶段，Frame Start，Pre-Physics，Physics，Post-Physics，Frame End，所有的引擎systems都在这些阶段中顺序执行。

无Task Graphs（用于任务多线程并行的系统），Task依赖链等。

### Entity Systems

Entity Systems指定需要在那些阶段进行update。

当注册一个update阶段的时候，同样需要指定其在这个更新阶段中的优先级。很容易控制Update更新顺序，而不需要显式依赖。

Entity Systems不允许引用或依赖其他entity systems。

### 本地和全局Entity Systems

Kruger有两种不同的entity systems，Per-Entity Systems（Local），Per-World Systems（Global）。

Entity Systems概念上等同于单例，每个entity/world对于特定类型的system只允许指定一个。

下图中，绿色为本地system，蓝色为全局system。

![image-20210112111656287](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20210112111656.png)

我们总是先update entities（本地entity system），再update全局entity systems

![image-20210112113411844](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20210112113411.png)

### 本地Entity Systems

Systems被手动添加到一个entity上。

只能操作这个entity上的的components。

可以具有临时的运行时状态，同样可以有每个实例的设置。

一个entity update只是对其local system的更新。

我们会预计算和缓存per-stage system的update list

![image-20210112113107103](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20210112113107.png)

### 全局Entity Systems

那些被自动创建在entity world的systems。

更像传统的ECS系统。

可以对当前世界所有特定类型的components进行处理，

可以有临时的运行时状态。

首要目的是处理全局的世界状态，其次是在entities之间传送数据。

![image-20210112113043382](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20210112113043.png)

### 再叙Entity Update

Entities不允许直接依赖其他entities（空间结构上的依赖除外）。

空间结构上的依赖导致父entity会在其子entities update之前调度，这让父entity在其子entities之前更新他自己的transform以及其他数据。

正是因为这些严格的限制，可以让我们对entities的update并行化。

![image-20210112113902194](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20210112113902.png)

### Entity Life-Cycle

![image-20210112114053044](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20210112114053.png)

Unloaded：所有的components都被unloaded

Loaded：所有的componemts被loaded，也有可能一些动态添加的components仍在loading

Activated：Entity正式在world启用，Entity components已经被注册到全部相应的systems

### Entity Activation

Entities在所有components被loaded和initialized后被激活。

激活过程做了以下事情：

- 将每个component注册到所有本地systems
- 创建一个per-stage本地systems的update列表
- 将每个component注册到所有全局systems
- 创建entity附加项（如果需要的话）

失活过程做了相反的事情

### 并行化 Activation

若果成千上万的entities尝试在同一帧激活/失活会是一个十分昂贵的操作。

我们尝试并行化这个激活过程，但我们必须非常小心，因为只有本地操作（local system，update list，spatial attachment）是线程安全的，之所以是线程安全的，是因为他们没有额外的依赖。

本地entity操作可以分组并行，我们只是简单的依据核心数将本地entity分组。

![image-20210112122118369](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20210112122118.png)

对于全局system注册，我们需要倒置逻辑，我们为每个全局system创建一个task，并填充所有的entity components到每个system，这可以让我们避免使用锁和面临线程竞争的问题。

![image-20210112122423147](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20210112122423.png)

### 为什么要划分本地和全局System

为了解决不同的问题。

本地systems被用于entity内部状态的更新，不能更改其他entities，首先被用作一个update方法和一个数据传送方法。

全局system管理世界状态和跨entity进行状态更新。在两个entities之间进行数据传递，基于entities管理世界的状态。

### 本地Entity System示例

![image-20210112123045264](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20210112123045.png)

Animation System负责执行animation tasks和传递结果到skeletal mesh component。

特定领域的逻辑仍然保持在单个组件中。例如Anim Update，Pose generation，Set Pose等

需要更新和传递数据的的逻辑才在system里。

![image-20210112124737048](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20210112124737.png)

![image-20210112124808731](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20210112124808.png)

### 全局Entity System示例

![image-20210112125010357](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20210112125010.png)

我们有一个全局的Static Mesh System，他会去查找所有Static Mesh Component，并判别是静态的还是动态的，然后统一执行操作，最后统一渲染。

全局systems不需要直接操作component data。

Component仅仅被用于表示某个特定事物存在的信息。

Systems可以根据他们特殊的需要创建一个自定义的内部状态。

### 全局Entity Systems：代理

![image-20210112130201402](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20210112130201.png)

因为数据的正确性由component保证，所以我们可以放心的进行代理。从而更方便的开发。

### 全局Entity Systems：数据传送

![image-20210112131511416](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20210112131511.png)

### 局部+全局Entity Systems

![image-20210112132410161](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20210112132410.png)
