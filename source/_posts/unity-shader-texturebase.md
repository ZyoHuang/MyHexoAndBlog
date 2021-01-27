---
title: Unity Shader入门精要学习笔记：基础纹理
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
cover: https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/10/QQ截图20191004231336.png
katex: true
aplayer:
---
<meta name="referrer" content="no-referrer" />

## 前言
纹理最初的目的就是使用一张图片来控制模型的外观。
在美术人员建模的时候，通常会在建模软件中利用纹理展开技术，把纹理映射坐标存储在每个顶点上。纹理映射坐标定义了该顶点在纹理中对应的2D坐标。
通常这些坐标使用一个二维变量(u,v)来表示，其中u是横向坐标，而v是纵向坐标，因此纹理映射坐标也被称为UV坐标。
但顶点UV坐标的范围通常都被归一化到[0,1]范围内。纹理采样时使用的纹理坐标不一定是在[0,1]范围内。实际上，这种不在[0,1]范围内的纹理坐标又是会非常有用，与之关系紧密的是纹理的平铺模式，它将决定渲染引擎在遇到不在[0,1]范围内的纹理坐标时如何进行纹理采样。
## 单张纹理
我们通常会使用一张纹理来代替物体的漫反射颜色。
### 实践
我们使用Blinn-Phong光照模型来计算光照。
```GLSL
// Upgrade NOTE: replaced '_Object2World' with 'unity_ObjectToWorld'
// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'

Shader "Unity Shaders Book/Chapter7/SingleTexture"
{
    //添加纹理属性
    Properties
    {
        //为了控制物体的整体色调，声明_Color属性
        _Color ("Color Tint",Color) = (1,1,1,1)
        //一个全白的纹理
        _MainTex ("Main Tex",2D) = "white"{}
        _Specular("Specular",Color) = (1,1,1,1)
        _Gloss("Gloss",Range(8.0,256)) = 20
    }
    SubShader
    {
        Pass
        {
            //指定Pass的光照模式
            Tags { "LightMode" = "ForwardBase" }
            CGPROGRAM
            
            #pragma vertex vert
            #pragma fragment frag
            
            #include "Lighting.cginc"
            
            //声明匹配类型
            fixed4 _Color;
            //Unity Texture Importor设置会影响这里的变量
            //即此变量包含采样时所需要的所有设置
            sampler2D _MainTex;
            //在Unity中，我们需要使用 纹理名_ST 的方式来声明某个纹理的属性。其中ST是缩放和平移的缩写
            //_MainTex_ST.xy存储的是缩放值，而__MainTex_ST.zw存储的是偏移值
            float4 _MainTex_ST;
            fixed4 _Specular;
            float _Gloss;
            
            //顶点着色器的输入和输出结构体
            struct a2v
            {   
                float4 vertex : POSITION;
                float3 normal : NORMAL;
                //Unity会将模型的第一组纹理坐标存储到该变量中
                float4 texcoord : TEXCOORD0;
            };
            
            struct v2f
            {
                float4 pos : SV_POSITION;
                float3 worldNormal : TEXCOORD0;
                float3 worldPos : TEXCOORD1;
                //用于存储纹理坐标，以便在片元着色器中使用该坐标进行纹理采样。
                float2 uv : TEXCOORD2;
            };
            
            //顶点着色器
            v2f vert(a2v v)
            {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                o.worldNormal = UnityObjectToWorldNormal(v.normal);
                o.worldPos = mul(unity_ObjectToWorld,v.vertex).xyz;
                //使用纹理的属性值_MainTex_ST来对顶点纹理坐标进行变换，得到最终的纹理坐标
                //先使用缩放属性__MainTex_ST.xy对顶点纹理坐标进行缩放
                //再使用偏移属性__MainTex_ST.zw对结果进行偏移
                //Unity提供了一个内置宏TRANSFORM_TEX来帮我们计算上述过程
                o.uv = v.texcoord.xy * _MainTex_ST.xy+_MainTex_ST.zw;
                return o;
            }
            
            //实现片元着色器，并在计算漫反射时使用纹理中的纹素值
            fixed4 frag(v2f i):SV_Target
            {   
                //世界空间下的法线方向
                fixed3 worldNormal = normalize(i.worldNormal);
                //世界空间下的光照方向
                fixed3 worldLightDir  =normalize(UnityWorldSpaceLightDir(i.worldPos));
                //对纹理进行采样，返回计算得到的纹素值
                //使用采样结果和颜色属性_Color乘积来作为材质的反射率(albedo)
                fixed3 albedo = tex2D(_MainTex, i.uv).rgb * _Color.rgb;
                //得到环境光部分
                fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz*albedo;
                //计算漫反射光照效果
                fixed3 diffuse = _LightColor0.rgb * albedo*max(0,dot(worldNormal,worldLightDir));
                //计算高光反射部分
                fixed3 viewDir = normalize(UnityWorldSpaceLightDir(i.worldPos));
                fixed3 halfDir = normalize(worldLightDir+viewDir);
                fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(max(0, dot(worldNormal,halfDir)),_Gloss);
                //相加返回
                return fixed4(ambient+diffuse+specular,1.0);
            }

            ENDCG
        }
    }
    Fallback "Specular"
}

```
### Unity中纹理的基本知识点
#### 纹理的基本设置
`Wrap Mode`，它决定了当纹理坐标超过[0,1]范围后将会如何被平铺。Wrap Mode有两种模式：
一种是`Repeat`，在这种模式下，如果纹理坐标超过了1，那么它的整数部分将会被舍弃，而直接使用小数部分进行采样，这样的结果是纹理将会不断重复。
另一种是`Clamp`，在这种模式下，如果纹理坐标大于1，那么将会被截取到1，如果小于0，将会截取到0。
纹理缩放更加复杂的原因是在于我们往往需要处理`抗锯齿`问题，一个方法就是使用`多级渐远纹理技术（mipmapping）`。
多级渐远纹理技术将原纹理提前用滤波处理来得到很多更小的图像，形成一个图像金字塔，每一层都是对上一层图像降采样的结果。这样在实时运行时，就可以快速得到结果像素。
在Unity中，我们可以在纹理导入面板中，勾选Generate Mip Maps即可开启`多级渐远纹理技术`，同时，我们还可以选择生成多级渐远纹理时是否使用线性空间（用于伽马校正）以及采用的滤波器等。
对于`Filter Mode`（滤波模式）（假设同时使用多级渐远纹理技术）
Point模式使用了最近邻滤波，在放大或缩小时，它的采样像素数目通常只有一个，因此图像会看起来有种像素风的效果。
Bilinear滤波使用了线性滤波，对于每个目标像素，他会找4个临近数据，然后对他们进行线性插值混合后得到最终像素，因此图像看起来像被模糊了。
Trilinear滤波几乎和Bilinear一样，但是他还会在多级渐远纹理之间进行混合，如果一张纹理没有使用多级渐远纹理技术，那么Trilinear得到的结果和Bilinear就是一样的。
#### 纹理的最大尺寸和纹理模式
如果导入纹理大小超过Max Texture Size中的设置值，那么Unity将会把该纹理缩放为这个最大分辨率，理想情况下，导入的纹理可以是正方形的，但`长宽大小应该是2的幂`，否则（NPOT）将会占用更多的内存空间，而且GPU读取该纹理的速度也会有所下降。
Format决定了Unity内部使用哪种格式来存储该纹理。使用的纹理精度越高，占用的内存空间越大，但是得到的效果也越好（如果开启多级渐远技术也会增加纹理的内存占用）。所以对于一些不需要使用很高精度的纹理（例如用于漫反射颜色的纹理），我们应该尽量使用压缩格式。（Compression/Compressed）
## 凹凸映射
`凹凸映射的目的是使用一张纹理来修改模型表面的法线`，以便为模型提供更多的细节。这种方法不会真的改变模型的顶点位置，只是让模型看起来高低不平，但是可以从模型的轮廓看出破绽。
两种方法使用凹凸映射
一：使用一张`高度纹理`来模拟表面位移,然后得到一个修改后的法线值，这种方法也被称为`高度映射`。
二：使用一张`法线纹理`来直接存储表面法线，这种方法又被称为`法线映射`。
### 高度纹理
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/10/QQ截图20191004231336.png)
高度图中存储的是强度值，它用于表示模型表面局部的海拔高度，因此，颜色越浅表明该位置的表面越向外凸起，而颜色越深，表明该位置越向里凹。`优点是非常直观，缺点是计算更加复杂，在实时计算时不能直接得到表面法线，需要由像素灰度值计算而得，因此需要消耗更多性能`。
高度图通常会和发现映射一起使用，用于给出表面凹凸的额外信息，我们`通常会使用法线映射来修改光照`。

### 法线纹理
法线纹理中存储的就是表面的法线方向。由于法线方向分量范围在[-1,1],而像素分量范围为[0,1]，所以需要一个映射
$$pixel\;=\;\frac{normal+1}2$$
这也意味着，我们在Shader中对法线纹理进行纹理采样后，还需要对结果进行一次反映射过程，来得到原先的法线方向。
$$normal\;=\;pixel\times2-1$$
对于模型顶点自带的法线，它们是定义在模型空间中的，因此一种直接的想法就是将修改后的模型空间中的表面法线存储在一张纹理中——`模型空间的法线纹理`
然鹅，我们往往会选择另一种坐标空间——`模型的切线空间`来存储法线。对于模型的每个顶点，它们都有一个属于自己的切线空间，这个`切线空间的原点就是该顶点本身，而z轴是顶点的法线方向，x轴是顶点的切线方向，y轴可由法线和切线叉积而得，也被称为副切线或者副法线。`
下面是切线空间示意图
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/10/QQ截图20191004231502.png)
这种纹理被称为`切线空间的法线纹理`。
下面是模型空间法线纹理和切线空间法线纹理示意图
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/10/QQ截图20191005143528.png)
模型空间下的法线纹理看起来是“五颜六色的”，这是因为所有法线所在坐标空间都是同一个模型空间，即个点法线方向不同，经过映射后存储到纹理后对应颜色也不同。
切线空间下的法线纹理几乎都是蓝色的，因为每个法线所在坐标空间都是不同的切线空间，这种法线纹理存储了每个点在各自切线空间中法线扰动方向——z轴方向，`如果一个点的法线方向不变，值为(0,0,1)，所以映射过来都是蓝色。`这些蓝色部分表明法线纹理中的法线和模型本身法线一样，不需要改变。
#### 模型空间法线纹理和切线空间法线纹理对比
##### 模型空间法线纹理
- 实现简单，更加直观。
- 在纹理坐标的缝合处和尖锐的边角部分，可见突变（缝隙）少，可以提供平滑的边界。
- 记录的是绝对法线信息，仅可用于创建它的那个模型。
- 不可以进行UV动画
- 不可压缩，法线纹理每个方向都有可能，因此必须存储3个方向的值，不可压缩。
##### 切线空间法线纹理
- 自由度高，可以用于不同的模型。
- 可以进行UV动画
- 可压缩，由于切线空间下的法线纹理中的法线的Z方向总是正方向，因此我们可以仅存储XY方向，然后推导Z方向。
### 实践

1. 在切线空间下进行光照计算，效率比第二种方法高，因为可以在顶点着色器中就可以完成对光照方向和视角方向的变换。而第二种方法要先对法线纹理进行采样。所以变换过程必须在片元着色器计算，意味着一次矩阵操作。
2. 在世界空间下进行光照计算，通用性比第一种方法高，有时需要在世界空间下进行一些计算。具体就是在世界空间计算光照模型，在片元着色器把法线方向从切线空间变换到世界空间下（在顶点着色器把这个矩阵传递给片元着色器），最后在片元着色器中把法线纹理中的法线方向从切线空间变换到世界空间即可。尽管这种方法需要更多的计算，但是在需要使用Cubemap进行环境映射等情况下，我们就需要使用这种方法。

#### 在切线空间下进行计算
基本思路：在片元着色器中通过纹理采样得到切线空间下的法线，然后再与切线空间下的视角方向，光照方向等进行计算。为此要现在顶点着色器中把视角方向和光照方向从模型空间变换到切线空间，即模型空间到切线空间的变换矩阵。
```GLSL
// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'

Shader "Unity Shaders Book/Chapter 7/Normal Map In TangentSpace"
{
    Properties
    {
        _Color("Color Tint",Color) = (1,1,1,1)
        _MainTex("Main Tex",2D) = "White"{}
        //法线纹理属性
        _BumpMap("Normal Map",2D) = "bump"{}
        //控制凹凸程度，当为0时，意味着该法线纹理不会对光照产生任何影响
        _BumpScale("Bump Scale",Float) = 1.0
        _Specular("Specular",Color) = (1,1,1,1)
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

            #include "Lighting.cginc"
            #include "UnityCG.cginc"
                
            //声明对应属性变量
            fixed4 _Color;
            sampler2D _MainTex;
            //为了得到纹理平铺和偏移系数
            float4 _MainTex_ST;
            sampler2D _BumpMap;
            //为了得到纹理平铺和偏移系数
            float4 _BumpMap_ST;
            float _BumpScale;
            fixed4 _Specular;
            float _Gloss;

            //切线空间是由顶点法线和切线构建出的一个坐标空间，因此我们需要得到顶点切线信息
            struct a2v
            {
                float4 vertex : POSITION;
                float3 normal : NORMAL;
                float4 tangent : TANGENT;
                float4 texcoord : TEXCOORD0;
            };

            struct v2f
            {
                float4 pos : SV_POSITION;
                float4 uv : TEXCOORD0;
                float3 lightDir : TEXCOORD1;
                float3 viewDir : TEXCOORD2;
            };


            v2f vert (a2v v)
            {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                
                o.uv.xy = v.texcoord.xy * _MainTex_ST.xy + _MainTex_ST.zw;
                o.uv.zw = v.texcoord.xy * _BumpMap_ST.xy + _BumpMap_ST.zw;       
               
                //把模型空间下的切线方向，副切线方向和法线方向按行排列得到从模型空间到切线空间的变换矩阵rotation
                TANGENT_SPACE_ROTATION;
                
                //得到模型空间下的光照和视角方向
                //然后利用rotation矩阵把他们从模型空间变换到切线空间中
                o.lightDir = mul(rotation,ObjSpaceLightDir(v.vertex)).xyz;
                o.viewDir = mul(rotation,ObjSpaceViewDir(v.vertex)).xyz;
                         
                return o;
            }
            
            fixed4 frag(v2f i) : SV_Target
            {
                fixed3 tangentLightDir = normalize(i.lightDir);
                fixed3 tangentViewDir = normalize(i.viewDir);
                //对法线纹理进行采样
                fixed4 packedNormal = tex2D(_BumpMap,i.uv.zw);
                fixed3 tangentNormal;
                
                //反映射到原本的法线方向
                //我们应该在Unity就把该法线纹理类型设置为Normal map
                //Unity会根据平台选择不同压缩方法
                //使用Unity的内置函数UnpackNormal得到正确的法线方向
                tangentNormal = UnpackNormal(packedNormal);
                //乘以控制凹凸度变量
                tangentNormal.xy *= _BumpScale;
                //由于法线都是单位向量，所以可以计算出z
                tangentNormal.z = sqrt(1.0 - saturate(dot(tangentNormal.xy,tangentNormal.xy)));
                //对另一张纹理进行采样
                fixed3 albedo = tex2D(_MainTex,i.uv).rgb * _Color.rgb;
                //得到环境光部分
                fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz * albedo;
                //计算漫反射光照效果
                fixed3 diffuse = _LightColor0.rgb * albedo * max(0,dot(tangentNormal,tangentLightDir));
                //计算高光反射部分
                fixed3 halfDir = normalize(tangentLightDir + tangentViewDir);
                fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(max(0,dot(tangentNormal,halfDir)),_Gloss);
                return fixed4(ambient + diffuse + specular,1.0);
            }

            ENDCG
        }
    }
    
    Fallback "Specular"
}

```
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/10/QQ截图20191005163659.png)

#### 在世界空间下计算
```GLSL
// Upgrade NOTE: replaced '_Object2World' with 'unity_ObjectToWorld'

Shader "Unlit/Chapter-NormalMapWorldSpace"
{
     Properties
    {
        _Color("Color Tint",Color) = (1,1,1,1)
        _MainTex("Main Tex",2D) = "White"{}
        //法线纹理属性
        _BumpMap("Normal Map",2D) = "bump"{}
        //控制凹凸程度，当为0时，意味着该法线纹理不会对光照产生任何影响
        _BumpScale("Bump Scale",Float) = 1.0
        _Specular("Specular",Color) = (1,1,1,1)
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

            #include "Lighting.cginc"
            #include "UnityCG.cginc"
                
            //声明对应属性变量
            fixed4 _Color;
            sampler2D _MainTex;
            //为了得到纹理平铺和偏移系数
            float4 _MainTex_ST;
            sampler2D _BumpMap;
            //为了得到纹理平铺和偏移系数
            float4 _BumpMap_ST;
            float _BumpScale;
            fixed4 _Specular;
            float _Gloss;

            //切线空间是由顶点法线和切线构建出的一个坐标空间，因此我们需要得到顶点切线信息
            struct a2v
            {
                float4 vertex : POSITION;
                float3 normal : NORMAL;
                float4 tangent : TANGENT;
                float4 texcoord : TEXCOORD0;
            };


            struct v2f
            {
                float4 pos : SV_POSITION;
                float4 uv : TEXCOORD0;
                //包含从切线空间到世界空间的变换矩阵
                //一个插值寄存器最多只能存储float4大小的变量
                //对于矩阵这样的变量，我们可以把它们按行拆分成多个变量再进行存储
                float4 TtoW0 : TEXCOORD1;
                float4 TtoW1 : TEXCOORD2;
                float4 TtoW2 : TEXCOORD3;
            };


            //计算从切线空间到世界空间的变换矩阵
            v2f vert (a2v v)
            {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                
                o.uv.xy = v.texcoord.xy * _MainTex_ST.xy + _MainTex_ST.zw;
                o.uv.zw = v.texcoord.xy * _BumpMap_ST.xy + _BumpMap_ST.zw;       
               
                float3 worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;
                fixed3 worldNormal = UnityObjectToWorldNormal(v.normal);
                fixed3 worldTangent = UnityObjectToWorldDir(v.tangent.xyz);
                fixed3 worldBinormal = cross(worldNormal,worldTangent)*v.tangent.w;
                
                o.TtoW0 = float4(worldTangent.x,worldBinormal.x,worldNormal.x,worldPos.x);
                o.TtoW1 = float4(worldTangent.y,worldBinormal.y,worldNormal.y,worldPos.y);
                o.TtoW2 = float4(worldTangent.z,worldBinormal.z,worldNormal.z,worldPos.z);         
                return o;
            }
            
            fixed4 frag(v2f i) : SV_Target
            {
                //先从TtoW0,TtoW1,TtoW2的w分量中构建世界空间下的坐标
                float3 worldPos = float3(i.TtoW0.w,i.TtoW1.w,i.TtoW2.w);
                //通过内置UnityWorldSpaceLightDir和UnityWorldSpaceViewDir得到世界空间下的光照和视角方向
                fixed3 lightDir = normalize(UnityWorldSpaceLightDir(worldPos));
                fixed3 viewDir = normalize(UnityWorldSpaceViewDir(worldPos));

                //对法线纹理进行采样和解码（需要把法线纹理的格式标示为Normal Map）
                fixed3 bump = UnpackNormal(tex2D(_BumpMap,i.uv.zw));
                bump.xy *= _BumpScale;
                bump.z = sqrt(1.0 - saturate(dot(bump.xy,bump.xy)));
                //转换到世界空间
                bump = normalize(half3(dot(i.TtoW0.xyz,bump),dot(i.TtoW1.xyz,bump),dot(i.TtoW2.xyz,bump)));

                //对另一张纹理进行采样
                fixed3 albedo = tex2D(_MainTex,i.uv).rgb * _Color.rgb;
                //得到环境光部分
                fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz * albedo;
                //计算漫反射光照效果
                fixed3 diffuse = _LightColor0.rgb * albedo * max(0,dot(bump,lightDir));
                //计算高光反射部分
                fixed3 halfDir = normalize(lightDir + viewDir);
                fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(max(0,dot(bump,halfDir)),_Gloss);
                return fixed4(ambient + diffuse + specular,1.0);
            }

            ENDCG
        }
    }
    
    Fallback "Specular"
}

```
### Unity中的法线纹理类型
当把法线纹理标识成Normal map，可以使用Unity的内置函数UnpackNormal来得到正确的法线方向。
当我们需要使用那些包含了法线映射的内置Unity Shader时，必须把使用的法线纹理按上面的方式标识成Normal map才能得到正确结果。
这可以让Unity根据不同平台对纹理进行压缩。
### 渐变纹理
一种常见用法就是使用渐变纹理来控制漫反射光照结果。
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/10/QQ截图20191005182402.png)
```GLSL
// Upgrade NOTE: replaced '_Object2World' with 'unity_ObjectToWorld'
// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'

Shader "Unity Shaders Book/Chapter 7/Ramp Texture"
{
    Properties
    {
        _Color("Color Tint",Color) = (1,1,1,1)
        _RampTex("Ramp Tex",2D) = "White"{}
        _Specular("Specular",Color) = (1,1,1,1)
        _Gloss("Gloss",Range(8.0,256)) = 20
    }
    SubShader
    {
        //指定光照模式
        Tags{"LightMode"="ForwardBase"}

        Pass    
        {
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            #include "Lighting.cginc"

            fixed4 _Color;
            sampler2D _RampTex;
            float4 _RampTex_ST;
            fixed4 _Specular;
            float _Gloss;

            struct a2v
            {
                float4 vertex : POSITION;
                float3 normal : NORMAL;
                float4 texcoord : TEXCOORD0;
            };

            struct v2f
            {
                float4 pos : SV_POSITION;
                float3 worldNormal : TEXCOORD0;
                float3 worldPos : TEXCOORD1;
                float2 uv : TEXCOORD2;
            };

            v2f vert (a2v v)
            {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                o.worldNormal = UnityObjectToWorldNormal(v.normal);
                o.worldPos = mul(unity_ObjectToWorld,v.vertex).xyz;
                //计算经过平铺和偏移后的纹理坐标
                o.uv = TRANSFORM_TEX(v.texcoord, _RampTex);
                return o;
            }

            fixed4 frag (v2f i) : SV_Target
            {
                fixed3 worldNormal = normalize(i.worldNormal);
                fixed3 worldLightDir = normalize(UnityWorldSpaceLightDir(i.worldPos));
                
                fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;
                
                //半兰伯特模型
                fixed halfLambert = 0.5*dot(worldNormal,worldLightDir)+0.5;

                //_RampTex实际上是一个一维纹理，他在纵轴方向颜色不变，
                //因此u,v都使用了半兰伯特，最后与_Color相乘得到最后的漫反射颜色
                fixed3 diffuseColor = tex2D(_RampTex,fixed2(halfLambert,halfLambert)).rgb*_Color.rgb;
                
                //计算漫反射和高光反射
                fixed3 diffuse = _LightColor0.rgb * diffuseColor;
                fixed3 viewDir = normalize(UnityWorldSpaceViewDir(i.worldPos));
                fixed3 halfDir = normalize(worldLightDir+viewDir);
                fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(max(0,dot(worldNormal,halfDir)),_Gloss);
                return fixed4(ambient + diffuse +specular,1.0);
            }
            ENDCG
        }
    }
    
    Fallback "Specular"
}

```
需要注意，我们需要把渐变纹理的Wrap Mode设置为Clamp模式，防止对纹理进行采样时由于浮点精度而造成的问题。
使用Repeat模式会舍弃整数部分，只保留小数部分，有时会造成部分黑点。
换成Clamp模式即可解决这种问题。
### 遮罩纹理
遮罩允许我们可以保护某些区域，使他们免于修改。
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/10/QQ截图20191005190806.png)
```GLSL
// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'

Shader "Unity Shaders Book/Chapter7/MaskTexture"
{
    Properties
    {
        _Color("Color Tint",Color) = (1,1,1,1)
        _MainTex("Main Tex",2D) = "white"{}
        _BumpMap("Normal Map",2D) = "bump"{}
        _BumpScale("Bump Scale",Float) = 1.0
        //使用高光反射遮罩纹理
        _SpecularMask("Specular Mask",2D) = "white"{}
        //控制遮罩影响度系数
        _SpecularScale("Specular Scale",Float) = 1.0
        _Specular ("Specular",Color) = (1,1,1,1)
        _Gloss("Gloss",Range(8.0,256)) = 20
    }
    SubShader
    {
        //指定光照模式
        Tags{"LightMode"="ForwardBase"}
        Pass
        {
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            #include "Lighting.cginc"

            fixed4 _Color;
            //主纹理
            sampler2D _MainTex;
            //法线纹理
            sampler2D _BumpMap;
            float _BumpScale;
            //遮罩纹理
            sampler2D _SpecularMask;
            float _SpecularScale;
            //三个纹理共同使用的纹理属性变量
            float4 _MainTex_ST;
            fixed4 _Specular;
            float _Gloss;

            struct a2v
            {
                float4 vertex : POSITION;
                float3 normal : NORMAL;
                float4 tangent : TANGENT;
                float4 texcoord : TEXCOORD0;
            };

            struct v2f
            {
                float4 pos : SV_POSITION;
                float2 uv : TEXCOORD0;
                float3 lightDir : TEXCOORD1;
                float3 viewDir : TEXCOORD2;
            };


            v2f vert (a2v v)
            {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                
                o.uv.xy = v.texcoord.xy * _MainTex_ST.xy + _MainTex_ST.zw;
                //模型空间到切线空间转换矩阵
                TANGENT_SPACE_ROTATION;
                //光照方向和视角方向也转换到切线空间，便于在片元着色器中和法线进行光照计算
                o.lightDir = mul(rotation,ObjSpaceLightDir(v.vertex)).xyz;
                o.viewDir = mul(rotation,ObjSpaceViewDir(v.vertex)).xyz;
                
                return o;
            }

            fixed4 frag (v2f i) : SV_Target
            {
                fixed3 tangentLightDir = normalize(i.lightDir);
                fixed3 tangentViewDir = normalize(i.viewDir);
                
                fixed3 tangentNormal = UnpackNormal(tex2D(_BumpMap,i.uv));
                tangentNormal.xy *= _BumpScale;
                tangentNormal.z = sqrt(1.0 - saturate(dot(tangentNormal.xy,tangentNormal.xy)));
                
                fixed3 albedo = tex2D(_MainTex,i.uv).rgb * _Color.rgb;
                fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz * albedo;
                
                fixed3 diffuse = _LightColor0.rgb*albedo*max(0,dot(tangentNormal,tangentLightDir));
                
                fixed3 halfDir = normalize(tangentLightDir + tangentViewDir);
                
                fixed specularMask = tex2D(_SpecularMask,i.uv).r*_SpecularScale;
                fixed3 specular = _LightColor0.rgb * _Specular.rgb*pow(max(0,dot(tangentNormal,halfDir)),_Gloss) * specularMask;
                
                return fixed4(ambient + diffuse + specular,1.0);
            }
            ENDCG
        }
    }
    Fallback "Specular"
}

```
### 其他遮罩纹理
在真实的游戏制作过程中，遮罩纹理已经不止限于保护某些区域使他们免于某些修改，而是可以存储任何我们希望逐像素控制的表面属性。我们会充分利用一张纹理RGBA四个通道，用于存储不同属性。
