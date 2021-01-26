---
title: Unity Shader入门精要学习笔记：Unity中的基础光照
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
## 我们如何看待这个世界的

通常，我们要模拟真实的光照环境来生成一张图像，需要考虑三种物理现象。
- 首先，光纤从光源中被发射出来
- 然后，光线与场景中的一些物体相交，一些光线被物体吸收了，而另一些光线被散射到其他方向。
- 最后，摄像机吸收了一些光，产生了一张图像。
### 光源
在实时渲染中，我们通常把光源当成一个没有体积的点，用I表示他的方向。用`辐照度`来量化光。对于平行光，它的辐照度可通过计算在垂直于I的单位面积上单位时间内穿过的能量来得到。
在计算光照模型时，我们需要知道一个物体表面的辐照度，而物体表面往往是和I不垂直的，那么如何计算这样的表面辐照度呢？我们可以使用光源方向I与表面发现n之间的夹角余弦值来得到。默认方向矢量模为1.
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/09/QQ截图20190901092840.png)
### 吸收和散射
光线由光源发射出来后，就会与一些物体相交。通常相交结果有两个：`散射`和`吸收`
`散射只改变光线方向，但不改变光线的密度和颜色。而吸收只改变光线的密度和颜色，但是不改变光线方向。`
光线在物体表面经过散射后，有两种方向，一种会散射到物体内部，这种现象被称为`折射`或者`透射`。另一种会散射到外部，这种现象被称为`反射`。
为了区分这两种不同的散射方向，我们在光照模型中使用不同的部分来计算他们：`高光反射`部分表示物体表面是如何反射光线的。`漫反射`部分则表示有多少光线会被`折射，吸收和散射出表面`。
根据入射光线的数量和方向，我们可以计算出射光线的数量和方向，我们通常使用`出射度`来表述他。
辐照度和出射度之间满足线性关系，而他们之间的比值就是材质的漫反射和高光反射属性。
### 着色
着色指的是，根据材质属性（如漫反射属性等），光源信息（如光源方向，辐照度等），使用一个等式去计算沿某个观察方向的出射度的过程。
我们也把这个等式称为`光照模型`。
### BRDF光照模型
BRDF表述了一个表面是如何和光照进行交互的。
当给定模型表面上的一个点时，BRDF包含了该点外观的完整的描述。
当给定入射光线的方向和辐射度后，BRDF可以给出在某个出射方向的光照能量分布。
计算机图形学第一定律：如果他看起来是对的，那么他就是对的。
有时我们希望更加真实地模拟光和物体的交互，这就出现了基于物理的BRDF模型。
## 标准光照模型
存在于BRDF概念提出之前。
它的基本方法是：把进入摄像机内部的光线分为4个部分，每个部分使用一种方法来计算它的贡献度

- 自发光，用$c_{emisssive}$表示。这个部分用于描述当给定一个方向时，一个表面本身会向该方向发射多少辐射量。如果没有使用全局光照，这些自发光的表面并不会真的照亮周围物体，而是它本身看起来更亮了而已，
- 高光反射部分，用$c_{specular}$表示，这个部分用于描述当光线从光源照射到模型表面时，该表面会在完全镜面反射方向散射多少辐射量。
- 漫反射部分，用$c_{diffuse}$表示，这个部分用于描述当光线从光源照射到模型表面时，该表面会向每个方向散射多少辐射量。
- 环境光部分，用$c_{ambient}$来表示，用于描述其他所有的间接光照。

### 环境光
它通常是一个全局变量，即场景中的所有物体都使用这个环境光。
$$c_{ambient} = g_{ambient}$$
### 自发光
光线直接由光源发射进入摄像机，不需要经过任何物体反射。直接使用该材质自发光颜色。
$$c_{emisssive} = m_{emisssive}$$
### 漫反射
漫反射光照是用于对那些被物体表面随机散射到各个方向的辐射度进行建模的，在漫发射中，视角的位置是不重要的，因为反射是完全随机的。可以认为在任何反射方向上的分布都是一样的，但是入射光线角度很重要。
漫反射光照符合`兰伯特定律`。：反射光线强度与表面法线和光源方向之间夹角余弦值成正比，因此漫反射部分计算如下
$$c_{diffuse}\;=\;(c_{light}\cdot m_{diffuse})max(0,n\cdot I)$$
其中，n是表面法线，I是指向光源的单位矢量，$m_{diffuse}$是材质的漫反射颜色，$c_{light}$是光源颜色，`我们需要防止法线和光源方向点乘的结果为负值。为此，我们使用取最大值的函数来将其截取到0，这可以防止物体被从后面来的光源照亮`。
### 高光反射
这里的高光反射是一种经验模型，也就是说，它并不完全符合真实世界中的高光反射现象。他可以用于计算那些沿着完全镜面反射方向被反射的光线，这可以让物体看起来是有光泽的，例如金属材质。
计算高光反射需要知道的信息比较多，如表面法线，视角方向，光源方向，反射方向等。
我们可以通过前三个变量来求出第四个变量。
$$r\;=\;2(\widehat n\cdot I)\widehat n\;-\;I$$
我们可以利用Phong模型来计算高光反射的部分。
$$c_{specular}\;=\;(c_{light}\cdot m_{specular})max(0,\widehat v\cdot r)^{m_{gloss}}$$
其中，$m_{gloss}$是材质的光泽度，也被称为反光度，它用于控制高光区域的“亮点有多宽”，$m_{gloss}$越大，亮点越小，$m_{specular}$是材质的高光反射颜色。用于控制材质对于高光反射的强度和颜色。$c_{light}$则是光源的颜色和强度。同样，这里也要防止法线与光源点乘为负。
与上述Phong模型相比，Blinn提出了一个简单的修改方法来得到类似的效果、它的基本思想是，避免计算反射方向$\widehat r$，为此Blinn模型引入一个新的矢量$\widehat h$，它是通过对$\widehat v$和$\widehat I$取平均后再归一化得到的。即
$$\widehat h=\frac{\widehat v\;+\;I}{\left|\widehat v\;+\;I\right|}$$
然后使用$\widehat n$和$\widehat h$夹角进行计算。
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/09/QQ截图20190901105303.png)
Blinn模型公式如下
$$c_{specular}\;=\;(c_{light}\cdot m_{specular})max(0,\widehat n\cdot\widehat h)^{m_{gloss}}$$
在硬件实现时，如果摄像机和光源距离模型足够远的话，Blinn模型会快于Phong模型。这是因为，此时可以认为$\widehat v$和$\widehat I$都是定值，，因此$\widehat h$是一个常量。但是如果$\widehat v$和$\widehat I$不是定值，可能Phong更快。这两种光照模型都是经验模型，我们不应该认为Blinn模型是对“正确”Phong模型的近似。实际上，在一些情况下，Blinn模型更符合实验结果。
### 逐像素还是逐顶点
我们在两个地方可以用到这些光照模型。
`片元着色器：也叫逐像素光照
顶点着色器：也叫逐顶点光照`
在逐像素光照中，我们以每个像素为基础，得到他的法线，然后进行光照模型计算。这种在面片之间对顶点法线进行插值的技术被称为`Phong着色`。也称为Phong插值或者法线插值着色技术。`不同于我们之前讲的Phong光照模型。`
在逐顶点光照中，也被称为`高洛德着色`。我们在每个顶点上计算光照，然后会在渲染图元内部进行线性插值。最后输出成像素颜色。`当光照模型中有非线性计算（例如高光反射）时，逐顶点光照会出现问题`。
### 总结
标准光照模型（Blinn-Phong模型）有局限性，很多重要的物理现象无法用Blinn-Phong模型表现出来，例如菲涅尔反射。其次，`Blinn-Phong模型是各向同性`。也就是说，当我们固定视角和光源方向旋转这个表面时，反射不会发生任何改变。但有些表面时具有`各向异性`的，比如拉丝金属，毛发等。
## Unity中的环境光和自发光
如果需要计算自发光，只需要在片元着色器输出最后颜色之前，把材质的自发光颜色添加到输出颜色上即可。
## 在Unity Shader中实现漫反射光照模型
此部分效果图在`对漫反射和高光反射各种实现图的汇总`中
### 实践：逐顶点光照
```glsl
// Upgrade NOTE: replaced '_World2Object' with 'unity_WorldToObject'

// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'

Shader "Unity Shaders Book/Chapter 6/Diffuse Vertex-Level"
{
    Properties
    {
        _Diffuse("Diffuse",Color) = (1,1,1,1)
    }
    SubShader
    {   
        Pass
        {
            //指定光照模式
            Tags{"LightMode"="ForwardBase"}
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            
            //包含Unity的内置文件Lighting.cginc
            #include "Lighting.cginc"
            //在shader中使用Properties语义块中声明的属性
            fixed4 _Diffuse;
            //定义顶点着色器输入和输出结构体（输出结构体同时也是片元着色器的输入结构体）
            struct a2v
            {
                float4 vertex : POSITION;
                //告诉unity要把模型顶点的法线信息存储到normal变量中
                float3 normal : NORMAL;
            };
            struct v2f
            {
                float4 pos:SV_POSITION;
                //把在顶点着色器中计算得到的光照颜色传递给片元着色器
                fixed3 color:COLOR;
            };
            
            v2f vert(a2v v)
            {
                v2f o;
                //把顶点位置从模型空间转换到裁剪空间中
                o.pos = UnityObjectToClipPos(v.vertex);
                //得到环境光
                fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;
                //使用顶点变换矩阵的逆转置矩阵对法线进行相同的变换，
                //因此我们要先得到模型空间到世界空间的变换矩阵的逆矩阵__World2Object
                //然后通过调换他在mul函数中的位置，得到和转置矩阵相同的矩阵乘法。
                //由于法线是一个三维矢量，我们只需要截取__World2Object前三行的前3列即可
                fixed3 worldNormal = normalize(mul(v.normal,(float3x3)unity_WorldToObject));
                fixed3 worldLight = normalize(_WorldSpaceLightPos0.xyz);
                //_LightColor0访问该Pass处理的光源颜色和强度信息以及光源方向(来自_WorldSpaceLightPos0)
                fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * saturate(dot(worldNormal,worldLight));
                o.color = ambient + diffuse;
                return o;
            }
            
            fixed4 frag(v2f i) : SV_Target
            {
                return fixed4(i.color, 1.0);
            }
            
            ENDCG
        }
    }
    Fallback "Diffuse"
}
```
### 逐像素光照
```glsl
// Upgrade NOTE: replaced '_World2Object' with 'unity_WorldToObject'

// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'

Shader "Unity Shaders Book/Chapter 6/Diffuse Vertex-Level"
{
    Properties
    {
        _Diffuse("Diffuse",Color) = (1,1,1,1)
    }
    SubShader
    {   
        Pass
        {
            //指定光照模式
            Tags{"LightMode"="ForwardBase"}
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            
            //包含Unity的内置文件Lighting.cginc
            #include "Lighting.cginc"
            //在shader中使用Properties语义块中声明的属性
            fixed4 _Diffuse;
            //定义顶点着色器输入和输出结构体（输出结构体同时也是片元着色器的输入结构体）
            struct a2v
            {
                float4 vertex : POSITION;
                //告诉unity要把模型顶点的法线信息存储到normal变量中
                float3 normal : NORMAL;
            };
            struct v2f
            {
                float4 pos:SV_POSITION;
                //把在顶点着色器中计算得到的光照颜色传递给片元着色器
                float3 worldNormal:TEXCOORD0;
            };
            
            //顶点着色器不需要计算光照模型，只需要把世界空间下的法线传递给片元着色器即可
            v2f vert(a2v v)
            {
                v2f o;
                //把顶点位置从模型空间转换到裁剪空间中
                o.pos = UnityObjectToClipPos(v.vertex);
                //使用顶点变换矩阵的逆转置矩阵对法线进行相同的变换，
                //因此我们要先得到模型空间到世界空间的变换矩阵的逆矩阵__World2Object
                //然后通过调换他在mul函数中的位置，得到和转置矩阵相同的矩阵乘法。
                //由于法线是一个三维矢量，我们只需要截取__World2Object前三行的前3列即可
                o.worldNormal = mul(v.normal, (float3x3)unity_WorldToObject);
                return o;
            }
            
            //片元着色器需要计算漫反射光照模型
            fixed4 frag(v2f i) : SV_Target
            {
                fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;
                
                fixed3 worldNormal = normalize(i.worldNormal);
                
                fixed3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz);
                //半兰伯特模型                
                //fixed halfLambert = dot(worldNormal,worldLightDir)*0.5+0.5;
                //fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * halfLambert;
                fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * saturate(dot(worldNormal,worldLightDir));
                fixed3 color = ambient + diffuse;
                
                return fixed4(color,1.0);
            }
            
            ENDCG
        }
    }
    Fallback "Diffuse"
}
```
逐像素光照可以得到更加平滑的光照效果，但是即使使用了逐像素漫反射光照，有一个问题仍然存在，在光照无法到达的区域，模型的外观通常是全黑的，没有任何变化，这回事模型背光去看起来像一个平面一样，失去了模型细节表现。实际上我们可以通过添加换金光来得到非全黑效果但即使这样仍然无法解决背光面明暗一样的缺点。为此，有一种改善技术被提出来，这就是`半兰伯特光照模型`。
### 半兰伯特模型
上面的逐顶点光照计算也被称为`兰伯特光照模型——在平面某点漫反射光的光强与该反射点的法向量和入射光角度余弦值成正比`。
而`半兰伯特光照模型`如下：
$$c_{diffuse}\;=\;(c_{light}\cdot m_{diffuse})(\alpha(\widehat n\cdot I)+\beta)$$
一般情况下$\alpha$和$\beta$都是`0.5`
这样一来，我们可以把$\widehat n\cdot I$结果范围从$\begin{bmatrix}-1,&1\end{bmatrix}$映射到$\begin{bmatrix}0,&1\end{bmatrix}$
```glsl
// Upgrade NOTE: replaced '_World2Object' with 'unity_WorldToObject'

// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'

Shader "Unity Shaders Book/Chapter 6/Diffuse Vertex-Level_halfLambert"
{
    Properties
    {
        _Diffuse("Diffuse",Color) = (1,1,1,1)
    }
    SubShader
    {   
        Pass
        {
            //指定光照模式
            Tags{"LightMode"="ForwardBase"}
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            
            //包含Unity的内置文件Lighting.cginc
            #include "Lighting.cginc"
            //在shader中使用Properties语义块中声明的属性
            fixed4 _Diffuse;
            //定义顶点着色器输入和输出结构体（输出结构体同时也是片元着色器的输入结构体）
            struct a2v
            {
                float4 vertex : POSITION;
                //告诉unity要把模型顶点的法线信息存储到normal变量中
                float3 normal : NORMAL;
            };
            struct v2f
            {
                float4 pos:SV_POSITION;
                //把在顶点着色器中计算得到的光照颜色传递给片元着色器
                float3 worldNormal:TEXCOORD0;
            };
            
            //顶点着色器不需要计算光照模型，只需要把世界空间下的法线传递给片元着色器即可
            v2f vert(a2v v)
            {
                v2f o;
                //把顶点位置从模型空间转换到裁剪空间中
                o.pos = UnityObjectToClipPos(v.vertex);
                //使用顶点变换矩阵的逆转置矩阵对法线进行相同的变换，
                //因此我们要先得到模型空间到世界空间的变换矩阵的逆矩阵__World2Object
                //然后通过调换他在mul函数中的位置，得到和转置矩阵相同的矩阵乘法。
                //由于法线是一个三维矢量，我们只需要截取__World2Object前三行的前3列即可
                o.worldNormal = mul(v.normal, (float3x3)unity_WorldToObject);
                return o;
            }
            
            //片元着色器需要计算漫反射光照模型
            fixed4 frag(v2f i) : SV_Target
            {
                fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;
                
                fixed3 worldNormal = normalize(i.worldNormal);
                
                fixed3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz);
                //半兰伯特模型                
                fixed halfLambert = dot(worldNormal,worldLightDir)*0.5+0.5;
                fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * halfLambert;
                fixed3 color = ambient + diffuse;
                
                return fixed4(color,1.0);
            }
            
            ENDCG
        }
    }
    Fallback "Diffuse"
}
```
## 在Unity中实现高光反射光照模型
此部分效果图在`对漫反射和高光反射各种实现图的汇总`中
### 逐顶点光照
```glsl
// Upgrade NOTE: replaced '_Object2World' with 'unity_ObjectToWorld'
// Upgrade NOTE: replaced '_World2Object' with 'unity_WorldToObject'

// Upgrade NOTE: replaced '_World2Object' with 'unity_WorldToObject'

// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'

Shader "Unity Shaders Book/Chapter 6/Specular Vertex-Level"
{
    Properties
    {
        _Diffuse("Diffuse",Color) = (1,1,1,1)
        //控制高光反射颜色
        _Specular("Specular",Color) = (1,1,1,1)
        //控制高光区域大小
        _Gloss("Gloss",Range(8.0,256)) = 20
    }
    SubShader
    {   
        Pass
        {
            //指定光照模式
            Tags{"LightMode"="ForwardBase"}
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            
            //包含Unity的内置文件Lighting.cginc
            #include "Lighting.cginc"
            //在shader中使用Properties语义块中声明的属性
            fixed4 _Diffuse;
            fixed4 _Specular;
            float _Gloss;
            //定义顶点着色器输入和输出结构体（输出结构体同时也是片元着色器的输入结构体）
            struct a2v
            {
                float4 vertex : POSITION;
                //告诉unity要把模型顶点的法线信息存储到normal变量中
                float3 normal : NORMAL;
            };
            struct v2f
            {
                float4 pos:SV_POSITION;
                //把在顶点着色器中计算得到的光照颜色传递给片元着色器
                fixed3 color:COLOR;
            };
            
            //计算包含高光反射的光照模型
            v2f vert(a2v v)
            {
                v2f o;
                //把顶点位置从模型空间转换到裁剪空间中
                o.pos = UnityObjectToClipPos(v.vertex);
                fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;
                fixed3 worldNormal = normalize(mul(v.normal, (float3x3)unity_WorldToObject));
                fixed3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz);
                fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * saturate(dot(worldNormal,worldLightDir));
                //高光反射部分
                fixed3 reflectDir = normalize(reflect(-worldLightDir,worldNormal));
                fixed3 viewDir = normalize(_WorldSpaceCameraPos.xyz - mul(unity_ObjectToWorld, v.vertex).xyz);
                fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(saturate(dot(reflectDir, viewDir)),_Gloss);
                o.color = ambient + diffuse + specular;
                return o;
            }
            
            //直接返回顶点颜色
            fixed4 frag(v2f i) : SV_Target
            {
                return fixed4(i.color, 1.0);
            }
            
            ENDCG
        }
    }
    Fallback "Specular"
}
```
可以发现高光部分其实很不平滑，这是因为高光反射部分的计算是非线性的，而在顶点着色器中计算光照再进行插值的过程是线性的，破坏了原计算的非线性关系。所以我们需要`使用逐像素的方法来计算高光反射`。
### 逐像素光照
```glsl
// Upgrade NOTE: replaced '_Object2World' with 'unity_ObjectToWorld'

// Upgrade NOTE: replaced '_Object2World' with 'unity_ObjectToWorld'
// Upgrade NOTE: replaced '_World2Object' with 'unity_WorldToObject'

// Upgrade NOTE: replaced '_World2Object' with 'unity_WorldToObject'

// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'

Shader "Unity Shaders Book/Chapter 6/Specular Pixel-Level"
{
    Properties
    {
        _Diffuse("Diffuse",Color) = (1,1,1,1)
        //控制高光反射颜色
        _Specular("Specular",Color) = (1,1,1,1)
        //控制高光区域大小
        _Gloss("Gloss",Range(8.0,256)) = 20
    }
    SubShader
    {   
        Pass
        {
            //指定光照模式
            Tags{"LightMode"="ForwardBase"}
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            
            //包含Unity的内置文件Lighting.cginc
            #include "Lighting.cginc"
            //在shader中使用Properties语义块中声明的属性
            fixed4 _Diffuse;
            fixed4 _Specular;
            float _Gloss;
            //定义顶点着色器输入和输出结构体（输出结构体同时也是片元着色器的输入结构体）
            struct a2v
            {
                float4 vertex : POSITION;
                //告诉unity要把模型顶点的法线信息存储到normal变量中
                float3 normal : NORMAL;
            };
            struct v2f
            {
                float4 pos:SV_POSITION;
                float3 worldNormal : TEXCOORD0;
                float3 worldPos : TEXCOORD1;
            };
            
            //计算包含高光反射的光照模型
            v2f vert(a2v v)
            {
                v2f o;
                //把顶点位置从模型空间转换到裁剪空间中
                o.pos = UnityObjectToClipPos(v.vertex);
                o.worldNormal = mul(v.normal, (float3x3)unity_WorldToObject);
                o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;
                return o;
            }
            
            //直接返回顶点颜色
            fixed4 frag(v2f i) : SV_Target
            {
                fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;
                fixed3 worldNormal = normalize(i.worldNormal);
                fixed3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz);
                fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * saturate(dot(worldNormal,worldLightDir));
                //高光反射部分
                fixed3 reflectDir = normalize(reflect(-worldLightDir,worldNormal));
                fixed3 viewDir = normalize(_WorldSpaceCameraPos.xyz - i.worldPos.xyz);
                fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(saturate(dot(reflectDir, viewDir)),_Gloss);
                return fixed4(ambient+diffuse+specular,1.0);
            }
            
            ENDCG
        }
    }
    Fallback "Specular"
}
```
按照逐像素的方式处理光照可以得到更加平滑的高光效果。
至此，我们就实现了一个完整的`Phong光照模型`。
### Blinn-Phong光照模型
```glsl
// Upgrade NOTE: replaced '_Object2World' with 'unity_ObjectToWorld'

// Upgrade NOTE: replaced '_Object2World' with 'unity_ObjectToWorld'
// Upgrade NOTE: replaced '_World2Object' with 'unity_WorldToObject'

// Upgrade NOTE: replaced '_World2Object' with 'unity_WorldToObject'

// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'

Shader "Unity Shaders Book/Chapter 6/Specular Pixel-Level_Blinn-Phong"
{
    Properties
    {
        _Diffuse("Diffuse",Color) = (1,1,1,1)
        //控制高光反射颜色
        _Specular("Specular",Color) = (1,1,1,1)
        //控制高光区域大小
        _Gloss("Gloss",Range(8.0,256)) = 20
    }
    SubShader
    {   
        Pass
        {
            //指定光照模式
            Tags{"LightMode"="ForwardBase"}
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            
            //包含Unity的内置文件Lighting.cginc
            #include "Lighting.cginc"
            //在shader中使用Properties语义块中声明的属性
            fixed4 _Diffuse;
            fixed4 _Specular;
            float _Gloss;
            //定义顶点着色器输入和输出结构体（输出结构体同时也是片元着色器的输入结构体）
            struct a2v
            {
                float4 vertex : POSITION;
                //告诉unity要把模型顶点的法线信息存储到normal变量中
                float3 normal : NORMAL;
            };
            struct v2f
            {
                float4 pos:SV_POSITION;
                float3 worldNormal : TEXCOORD0;
                float3 worldPos : TEXCOORD1;
            };
            
            //计算包含高光反射的光照模型
            v2f vert(a2v v)
            {
                v2f o;
                //把顶点位置从模型空间转换到裁剪空间中
                o.pos = UnityObjectToClipPos(v.vertex);
                o.worldNormal = mul(v.normal, (float3x3)unity_WorldToObject);
                o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;
                return o;
            }
            
            //直接返回顶点颜色
            fixed4 frag(v2f i) : SV_Target
            {
                fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;
                fixed3 worldNormal = normalize(i.worldNormal);
                fixed3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz);
                fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * saturate(dot(worldNormal,worldLightDir));
                //高光反射部分
                fixed3 viewDir = normalize(_WorldSpaceCameraPos.xyz - i.worldPos.xyz);
                fixed3 halfDir = normalize(worldLightDir + viewDir);
                fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(max(0,dot(worldNormal, halfDir)),_Gloss);
                return fixed4(ambient+diffuse+specular,1.0);
            }
            
            ENDCG
        }
    }
    Fallback "Specular"
}
```
`大多数情况下我们会选择Blinn-Phong光照模型。`
## 对漫反射和高光反射各种实现图的汇总
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/09/QQ截图20190901161031.png)
## 使用Unity内置函数
我们在上述例子中，都是通过代码计算的光源方向和视角方向（我们的处理方案只适合平行光），如果要处理更复杂的光照类型，比如点光源和聚光灯，我们计算光源方向的方法就是错误的。这需要我们在代码中先判断光源类型，再计算它的光源信息。
Unity提供了一些内置函数来帮助我们计算这些信息。
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/09/QQ截图20190901161612.png)
需要注意的是，`这些函数都没有保证得到的方向矢量是单位矢量，因此，我们需要在使用前把他们归一化。`
计算光源方向的三个函数：WorldSpaceLightDir,UnityWorldSpaceLightDir和ObjSpaceLightDir`仅可以用于前向渲染`，这是因为，只有在前向渲染时，这三个函数里使用的内置变量_WorldSpaceLightPos0等才会被正确赋值。
