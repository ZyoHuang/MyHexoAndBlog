---
title: Unity Shader入门精要学习笔记：开始Unity Shader学习之旅
date:
tags: 图形渲染
categories:
  - - 图形渲染
    - 理论知识
  - - 游戏引擎
    - Unity
keywords:
top_img:
cover: https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/08/QQ截图20190831102907.png
aplayer:
---
<meta name="referrer" content="no-referrer" />

## 一个最简单的顶点/片元着色器
### 顶点/片元着色器的基本结构
```
// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'
shader "Unity Shader Book/Chapter 5/Simple Shader"
{
    SubShader
    {
        Pass 
        {
            CGPROGRAM
            //表示vert函数是顶点着色器代码
            #pragma vertex vert 
            //表示fragment函数是片元着色器代码
            #pragma fragment frag
            
            float4 vert(float4 v : POSITION) : SV_POSITION
            {
                //Unity内置的模型·观察·投影矩阵
                return UnityObjectToClipPos (v);
            }
            
            fixed4 frag() : SV_Target
            {
                //返回一个颜色的fixed4类型变量
                return fixed4(0.3,0.4,1.0,1.0);
            }
            
            ENDCG
        }
    }
}

```
他长这样
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/08/QQ截图20190831102907.png)

### 模型数据从哪里来
在上面那个例子中，我们使用POSITION语义得到了模型的顶点位置。
如果我们想获得更多顶点数据呢。
现在，我们想得到模型上的每个顶点的纹理坐标和法线方向。我们需要使用纹理坐标来访问纹理，而法线可用于计算光照。因此，我们需要为顶点着色器定义一个新的输入参数，这个参数不再是一个简单的数据类型，而是一个结构体。
```
// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'
shader "Unity Shader Book/Chapter 5/Simple Shader"
{
    SubShader
    {
        Pass 
        {
            CGPROGRAM
            //表示vert函数是顶点着色器代码
            #pragma vertex vert 
            //表示fragment函数是片元着色器代码
            #pragma fragment frag
            
            //声明新的结构体包含顶点着色器需要的模型数据
            //a表示应用，v表示顶点着色器，a2v意思就是把数据从应用阶段传递到顶点着色器中
            struct a2v
            {
                // POSITION语义告诉Unity用模型空间的顶点坐标填充vertex变量
                float4 vertex : POSITION;
                // NORMAL语义告诉Unity，用模型空间的法线方向填充normal变量
                float3 normal : NORMAL;
                // TEXCOORD0语义告诉Unity，用模型的第一套纹理坐标填充texcoord变量
                float4 texcoord : TEXCOORD0;
            };
            
            float4 vert(a2v v) : SV_POSITION
            {
                //Unity内置的模型·观察·投影矩阵
                return UnityObjectToClipPos (v.vertex);
            }
            
            fixed4 frag() : SV_Target
            {
                //返回一个颜色的fixed4类型变量
                return fixed4(0.3,0.4,1.0,1.0);
            }
            
            ENDCG
        }
    }
}

```
那么，填充到POSITION,TANGENT,NORMAL这些语义中的数据究竟从哪里来的呢？在Unity中，他们是由该材质的`Mesh Render组件`提供的。在每帧调用Draw Call的时候，Mesh Render组件会把它负责渲染的模型数据发送给Unity Shader。我们知道，一个模型通常包含一组三角面片，每个三角面片由3个顶点构成，而`每个顶点又包含一些数据，例如顶点位置，法线，切线，纹理坐标，顶点颜色等`。通过上面的方法，我们就可以在顶点着色器中访问顶点的这些模型数据。
### 顶点着色器和片元着色器之间如何通信
我们需要再定义一个新的结构体。
```
// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'

// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'
shader "Unity Shader Book/Chapter 5/Simple Shader"
{
    SubShader
    {
        Pass 
        {
            CGPROGRAM
            //表示vert函数是顶点着色器代码
            #pragma vertex vert 
            //表示fragment函数是片元着色器代码
            #pragma fragment frag
            
            //声明新的结构体包含顶点着色器需要的模型数据
            //a表示应用，v表示顶点着色器，a2v意思就是把数据从应用阶段传递到顶点着色器中
            struct a2v
            {
                // POSITION语义告诉Unity用模型空间的顶点坐标填充vertex变量
                float4 vertex : POSITION;
                // NORMAL语义告诉Unity，用模型空间的法线方向填充normal变量
                float3 normal : NORMAL;
                // TEXCOORD0语义告诉Unity，用模型的第一套纹理坐标填充texcoord变量
                float4 texcoord : TEXCOORD0;
            };
            
            //使用一个结构体来定义顶点着色器的输出
            struct v2f
            {
                // SV_POSITION语义告诉Unity，pos里面包含了顶点在裁剪空间中的位置信息
                float4 pos : SV_POSITION;
                // COLOR0语义可以用于储存颜色信息
                fixed3 color : COLOR0;
            };
            
            v2f vert(a2v v)
            {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                o.color = v.normal * 0.5 + fixed3(0.5, 0.5, 0.5);
                return o;
            }
            
            fixed4 frag(v2f i) : SV_Target
            {
                //返回一个颜色的fixed4类型变量
                return fixed4(i.color, 1.0);
            }
            
            ENDCG
        }
    }
}

```
顶点着色器的输出结构中，必须包含一个变量，它的语义是SV_POSITION。否则，渲染器将无法得到裁剪空间中的顶点坐标，也无法将顶点渲染到屏幕上。
`SV_POSITION是DirectX 10中引入的系统数值语义。在绝大多数平台上，他和POSITION是等价的，但在某些平台上（例如PS4）上必须使用SV_POSITION来修饰顶点着色器的输出，否则无法让Shader正常工作。`
`Shader不规范，同事泪两行。`
片元着色器中的输入实际上是把顶点着色器的输出进行插值后得到的结果。
### 如何使用属性
我们想在材质面板显示一个颜色拾取器，从而可以直接控制模型在屏幕上显示的颜色。
```
// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'

// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'
shader "Unity Shader Book/Chapter 5/Simple Shader"
{
    Properties
    {
        _Color ("Color Tint", Color) = (0.5,0.6,0.2,1.0)
    }

    SubShader
    {
        Pass 
        {
            CGPROGRAM
            //表示vert函数是顶点着色器代码
            #pragma vertex vert 
            //表示fragment函数是片元着色器代码
            #pragma fragment frag
            
            //在CG代码中，我们需要定义一个与属性名称和类型都匹配的变量
            fixed4 _Color;
            
            //声明新的结构体包含顶点着色器需要的模型数据
            //a表示应用，v表示顶点着色器，a2v意思就是把数据从应用阶段传递到顶点着色器中
            struct a2v
            {
                // POSITION语义告诉Unity用模型空间的顶点坐标填充vertex变量
                float4 vertex : POSITION;
                // NORMAL语义告诉Unity，用模型空间的法线方向填充normal变量
                float3 normal : NORMAL;
                // TEXCOORD0语义告诉Unity，用模型的第一套纹理坐标填充texcoord变量
                float4 texcoord : TEXCOORD0;
            };
            
            //使用一个结构体来定义顶点着色器的输出
            struct v2f
            {
                // SV_POSITION语义告诉Unity，pos里面包含了顶点在裁剪空间中的位置信息
                float4 pos : SV_POSITION;
                // COLOR0语义可以用于储存颜色信息
                fixed3 color : COLOR0;
            };
            
            v2f vert(a2v v)
            {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                o.color = v.normal * 0.5 + fixed3(0.5, 0.5, 0.5);
                return o;
            }
            
            fixed4 frag(v2f i) : SV_Target
            {
                fixed3 c=i.color;
                c *= _Color.rgb;
                //返回一个颜色的fixed4类型变量
                return fixed4(c, 1.0);
            }
            
            ENDCG
        }
    }
}

```
有时我们会发现在CG变量前会有一个uniform关键字，这是CG中修饰变量和参数的一种修饰词，它仅仅用于提供一些关于该变量的初始值是如何指定和存储的相关信息，在Unity Shader中，uniform关键词可以省略。
## 强大的援手：Unity提供的内置文件和变量
### 内置的包含文件
包含文件是类似于C++中头文件的一种文件。在Unity中，他们文件后缀是.cginc。在编写Shader时，我们可以使用#include指令把这些文件包含进来，这样我们可以使用Unity为我们提供的一些非常有用的变量和帮助函数。
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/08/QQ截图20190831195429.png)
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/08/QQ截图20190831195600.png)
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/08/QQ截图20190831195702.png)
## Unity提供的CG/HLSL语义
在DirectX 10以后，有了一种新的语义类型，就是系统数值语义。这类语义以SV开头，SV代表的含义是系统数值。
### Unity支持的语义
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/08/QQ截图20190831200449.png)
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/08/QQ截图20190831200612.png)
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/08/QQ截图20190831200722.png)
## Debug
### 使用假彩色图像
### Visual Studio
### 帧调试器(Frame Debugger)
## 小心，渲染平台的差异
### 渲染纹理的坐标差异
注意，当开启抗锯齿之后，Unity不会像往常一样帮我们处理DirectX图像纹理翻转问题。
### Shader的语法差异
### Shader的语义差异
### 其他平台差异
## Shader整洁之道
### float，half还是fixed
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/08/QQ截图20190831203949.png)
上面给出的精度不一定是绝对正确的，尤其在不同平台和GPU上，他们实际精度可能和上面给出范围不一致。
- 大多数桌面GPU会把所有计算都按最高的浮点精度进行计算，也就是说，float，half，fixed在这些平台上实际上是等价的。这意味着，我们在PC上很难看出因为half和fixed精度而带来的不同。
- 在移动平台GPU上，他们的确会有不同的精度范围，而且不同精度的浮点值的运算速度也会有差异。因此，我们应该确保在真正的移动平台上验证我们的Shader。
- fixed精度实际上只在一些比较旧的移动平台上有用，在大多数现代GPU上，他们内部都把fixed和half当成同等精度来对待。
- 尽可能使用精度较低的类型，因为这样可以优化shader的性能。
### 规范语法
### 避免不必要的计算
### 慎用分支和循环语句
GPU使用了不同于CPU的技术来实现分支语句，在最坏的情况下，我们花在一个分支语句的时间相当于运行了所有分支语句的时间，因此我们不鼓励在Shader中使用流程控制语句。因为他们会降低GPU的并行处理操作。
我们应该尽量吧计算向流水线上端移动，例如把放在片元着色器中的计算放到顶点着色器中，或者直接在CPU中进行预计算，再把结果传递给Shader。
如果不可避免的使用分支语句的话
- 分支判断语句中使用的条件变量最好是常数，即在Shader运行过程中不会发生变化
- 每个分支中包含的操作指令数尽可能少
- 分支的嵌套层数尽可能少

### 不要除以0
