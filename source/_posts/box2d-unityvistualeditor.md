---
title: Box2D篇：在Unity开发Box2D可视化编辑器拓展
tags: []
id: '1510'
categories:
  - - GamePlay
  - - '%e6%b8%b8%e6%88%8f%e5%bc%95%e6%93%8e'
    - Unity
date: 2019-07-17 18:00:06
---

<meta name="referrer" content="no-referrer" />



## 前言

我们需要把Box2D的碰撞体数据导入到服务端，然后让服务端读取这些数据，重建2D物理世界。那么，这些碰撞数据从哪来，怎么做的呢？ Box2D官方手册里有提到两个比较出名的编辑器。 `PhysicsEditor`和`RUBE` 这两个编辑器都很出色，但是他们都有一个共同的痛点——编辑的碰撞体对于Unity不是所见即所得的。 还有比较特殊的应用场景，比如做Moba这种伪3D游戏，物理世界完全是可以用Box2D做的，但是，游戏里许多模型都是3D的，而PhysicsEditor和RUBE都只支持导入图片，所以也不妥当。 在仔细研究了官方文档后对比Unity自带的2D物理系统有很多共同点，我就想能不能借助Unity自身物理系统的力量做一个Box2D编辑器呢，这样可以所见即所得。 其实硬要说技术含量的话，在Odin的帮助下，这个可视化编辑器还真没啥技术含量。所以我在这主要说一下思路和历程，希望能对大家以后解决类似问题有帮助。文中代码尽量采用伪代码的形式，方便大家理解。

## 正文

先放出一张UML图，方便大家理解下面的代码 ![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/07/QQ截图20190717175843.png) **涉及到的相关知识点** - 编辑器拓展插件：Odin - 序列化反序列化：Bson - 多边形分割：Antoan Angelov的Separator - 矩阵变换相关知识 - Gizmos相关知识 - Box2D的相关知识\[toc\] 要做这种需要导出数据的编辑器拓展，我一般都遵循数据逻辑分离的原则，这样开发时干净利落，序列化反序列化时没有冗余信息。 这里就以最为复杂的多边形碰撞体数据构建为例 `本博客代码中所有特性都是Odin插件提供的支持，如果没接触过Odin的，就把他理解为利用特性把这个字段可视化了。`

### 数据结构的选定

首先是所有类型碰撞体都有的数据结构

```csharp
    public class B2S_ColliderDataStructureBase
    {
        [LabelText("碰撞体ID")]
        public long id;

        [LabelText("是否为触发器")]
        public bool isSensor;

        [LabelText("碰撞体类型")]
        public B2S_ColliderType b2SColliderType;

        [LabelText("碰撞体偏移信息")]
        public CostumVector2 offset = new CostumVector2(0, 0);
    }
```

这里为什么不用Unity的Vector2或者System的Vector2呢，因为Bson不支持反序列化结构体。。。，所以只能自己写一个class了 而且，我们做这种导出数据给服务端读取的编辑器的时候，尽量不要引用到Unity的变量，不然你还得需要在服务端做一些操作匹配上这些数据类型才行。 然后是多边形的数据结构

```csharp
    public class B2S_PolygonColliderDataStructure: B2S_ColliderDataStructureBase
    {
        [LabelText("碰撞体所包含的顶点信息(顺时针),可能由多个多边形组成")]
        public List<List<CostumVector2>> points = new List<List<CostumVector2>>();

        [LabelText("总顶点数")]
        public int pointCount;
    }
```

数据结构选好了，我们需要做一个数据载体，来承载所有的数据

```csharp
    public class ColliderDataSupporter
    {
        [BsonDictionaryOptions(DictionaryRepresentation.ArrayOfArrays)]
        public Dictionary<long, B2S_ColliderDataStructureBase> colliderDataDic = new Dictionary<long, B2S_ColliderDataStructureBase>();
    }
```

### 借助Unity设置碰撞体信息，并绘制出图形

数据层我们选好了，接下来要处理显示层，我选择了使用`Gizmos`来绘制线段

```csharp
public class B2S_PolygonColliderVisualHelper
{
    [LabelText("Unity的多边形碰撞体")]
    public PolygonCollider2D mCollider2D;
    [LabelText("多边形碰撞体数据")]
    public B2S_PolygonColliderDataStructure MB2S_PolygonColliderDataStructure = new B2S_PolygonColliderDataStructure();

    //新建矩阵变换，确保绘制的线段与Unity碰撞体线段位置信息一致
    //从mCollider2D读取碰撞体信息，并赋值给一个临时List<Vector2>

    //由于Box2D默认支持的多边形最多只能有8个顶点，而且还得是凸多边形，
    //所以我们需要使用Separator来进行多边形的分割

    //使用Gizmos画线，注意使用矩阵的MultiplyPoint方法对顶点进行变换
}
```

由于我们使用了Gizmos，所以我们必须得需要一个挂载MonoBehaviour脚本的游戏物体进行驱动画线

```csharp
        private void OnDrawGizmos()
        {
            foreach (var VARIABLE in this.MB2SColliderVisualHelpers)
            {
                if (VARIABLE.canDraw)
                    VARIABLE.OnDrawGizmos();
            }
        }
```

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/07/QQ截图20190717174309.png)

### 序列化反序列化信息与可视化

同样利用了Odin提供的特性，在EditorWindow做了数据的可视化，主要是方便开发调试时Debug 我们需要在打开EditorWindow的时候读取所有碰撞数据 ![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/07/QQ截图20190717174321.png)