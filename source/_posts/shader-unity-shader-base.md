---
title: Unity Shader入门精要学习笔记：Unity Shader基础
date:
updated:
tags: 图形渲染
categories:
  - - 图形渲染
    - 理论知识
  - - 游戏引擎
    - Unity
keywords:
top_img:
cover: https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/08/QQ截图20190828180847.png
aplayer:
---
<meta name="referrer" content="no-referrer" />

##  Unity Shader概述
### 材质和Unity Shader
在Unity中需要配合使用材质和Unity Shader才能达到需要的效果
## Unity Shader基础：ShaderLab
#### 什么是ShaderLab
Unity Shader是Unity为开发者提供的高层级的渲染抽象层。
在Unity中，所有的Unity Shader都是使用ShaderLab来编写的。
ShaderLab是Unity提供的编写Unity Shader的一种说明性语言。
Unity在背后会根据使用的平台来把这些结构编译成真正的代码和Shader文件，而开发者只需要和Unity Shader打交道即可。
## Unity Shader的结构
```glsl
Shader "Shader的名字，通过增加'/'来细分种类"
{
    Properties
    {
		//Name:属性的名字
		//display name：显示在材质面板上的名字
		//PropertyType：类型
		//DefaultValue：默认值
		Name("display name",PropertyType) = DefaultValue
    }
    SubShader
    {
		//可选标签，例如：Tags{"Queue"="Transparent"}
		[Tags]
		//可选状态
		[RenderSetup]
		//通道，每个通道定义了一次完整的渲染流程，但是如果数目过多会造成渲染性能的下降。
		Pass
		{
			[Name]
			[Tags]
			[RenderSetup]
		}
		//其他通道
		//Other Pass
    }
    Fallback "VertexLit"
}
```
### Unity Shader名称
通过增加'/'来细分种类
### 材质与Unity Sahder的桥梁Properties
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/08/QQ截图20190828180847.png)
### SubShader
每个Unity Shader可以包含多个SubShader语义块，但至少要有一个。
当Unity加载这个Shader时，会扫描所有SubShader语义块，然后选择第一个能在目标平台运行的SubShader，如果都不支持的话，Unity就会使用Fallback语义指定的Unity Shader。
#### 状态设置
这些指令可以设置显卡的各种状态，例如是否开启混合/深度测试等。
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/08/QQ截图20190828182445.png)
当在SubShader块中设置了上述渲染状态时。`将会应用到所有的Pass。
如果不想这样，我们可以在Pass中单独设置状态。`
#### 标签
SubShader的标签是一个`键值对`。键值都是字符串类型。
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/08/QQ截图20190828184214.png)
注意，上述标签只可以在SubShader中声明，不可以在Pass中声明，Pass块虽然也可以定义标签，但是这些标签是不同于SubShader的标签类型。
#### Pass语义块
Name：在用`UserPass`命令时必须使用大写形式的名字。
Tags：也是用于高速渲染引擎我们希望怎样来渲染该物体。
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/08/QQ截图20190828184628.png)
UsePass：用以复用其他UnityShader的Pass
GrabPass：负责抓取屏幕并将结果存储在一张纹理中，用于后续的Pass处理。
#### Fallback
Fallback "Name"
Fallback Off
`注意，为每个Unity Shader正确设置Fallback是非常重要的，这会影响阴影投射。`
#### 还有其他的
可以使用CostumEditor语义来拓展编辑界面。（注意，和平时拓展Inspector面板不一样哦）。
## Unity Shader的形式
### 表面着色器（Surface Shader）
Unity提供的一种着色器代码类型，是一个黑箱子，Unity在后面还是会把他转换成顶点片元着色器。
### 顶点/片元着色器（Vertex/Fragment Shader）
它更复杂，但灵活性也更高。
### 固定函数着色器
为了支持老式设备，已被淘汰。
## 答疑解惑
### Unity Shader != 真正的Shader
- 在传统Shader里，我们只可以编写特定类型Shader，而在Unity Shader里，我们可以在同一个文件里同时包含需要的顶点着色器和片元着色器代码。
- 在传统Shader里，我们无法设置一些渲染设置，例如是否开启混合，深度测试等。在Unity Shader中，我们通过一行特定指令就可以完成这些设置。
- 在传统Shader里，我们需要编写冗长的代码来设置着色器的输入和输出，在Unity Shader中，我们只需要在特定语句块中声明一些属性，就可以依靠材质来方便地改变这些属性。
### Unity Shader和CG/HLSL之间的关系
Unity Shader是用ShaderLab编写的，但是对于表面着色器和顶点片元着色器，我们可以在ShaderLab内部嵌套CG/HLSL语言来编写这些着色器代码。这些CG/HLSL代码是嵌套在CGPROGRAM和ENDCG之间的。在Unity中，CG和HLSL是等价的。
通常，CG代码片段位于Pass语义块内部的。
