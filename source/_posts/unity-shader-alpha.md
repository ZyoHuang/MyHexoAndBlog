---
title: Unity Shader入门精要学习笔记：透明效果
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
cover: https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/10/QQ截图20191008224344.png
aplayer:
---
<meta name="referrer" content="no-referrer" />

## 透明效果
透明是游戏中经常要用的一种效果，在实时渲染中要实现透明效果，通常会在渲染模型时控制它的透明通道(Alpha Channel)。当开启透明混合后，当一个物体被渲染到屏幕上时，每个片元除了颜色值和深度值外，他还有另外一个属性——`透明度`。当透明度为1时。表示该像素是完全不透明的，而当其为0时，则表示改像素完全不会显示。
在Unity中，我们通常使用`透明度测试`（这种方法其实无法得到真正的的半透明效果）和`透明度混合`实现透明效果。
对于不透明物体，不考虑他们渲染顺序也能得到正确的排序效果，这是由于强大深度缓冲（depth buffer也被称为z-buffer）。
在实时渲染中，深度缓冲是用于解决可见性问题的，核心思想：根据深度缓冲中的值来判断该片元距离摄像机的距离，当渲染一个片元时，需要把它的深度值和已经存在与深度缓冲中的值进行比较（如果开启了深度测试），`如果它的值距离摄像机更远，则不需要渲染到屏幕（有物体挡住了它），否则这个片元将会覆盖掉此时颜色缓冲中的像素值，并把它的深度值更新到深度缓冲中（如果开启深度写入）。`
但是如果要实现透明效果，当使用透明度混合时，我们关闭了深度写入（ZWrite）。
简单来说，透明度测试和透明度混合的基本原理如下。
**透明度测试**
只要一个片元的透明度不满足条件，那么它对应的片元就会被舍弃，并且不会再做任何处理，也不会影响颜色缓冲。`透明度测试不需要关闭深度写入。`只会产生两个结果，要么完全透明看不到，要么完全不透明。
**透明度混合**
使用当前片元的透明度作为混合因子，与已经存储在颜色缓冲中的颜色值进行混合，得到新的颜色。但是`透明度混合需要关闭深度写入。这是因为，如果不关闭深度写入，一个半透明表面背后的表面本来是可以透过它被我们看到的，但是由于深度测试时判断结果是该半透明表面距离摄像机更近，导致后面的表面将会被剔除，我们也无法通过半透明表面看到后面的物体了。`
透明度混合只关闭了深度写入，没有关闭深度测试，即`当使用透明度混合渲染一个片元时，还是会比较它的深度值与当前深度缓冲中的深度值，如果它的深度值距离摄像机更远，就不会进行混合操作`。所以，当一个不透明物体出现在一个透明物体前面，而我们先渲染了不透明物体，他仍然可以正常遮挡住透明物体。
`总结：对于透明度混合来说，深度缓冲是只读的。`

### 为什么渲染顺序很重要

因为透明度关闭了深度写入，如果不保证渲染顺序的话，会导致错误的渲染结果。
比如半透明A（离摄像机近），不透明B（离摄相机远），如果先渲染A，不会在深度缓冲区写入数据，但会写入颜色缓冲区。再渲染B，会在深度缓冲区写入深度值，然后写入颜色缓冲区，即覆盖A的颜色，看起来物体B在A前面了。
再比如半透明A（离摄像机近），半透明B（离摄像机远），如果先渲染A，不会在深度缓冲区写入数据，但会写入颜色缓冲区。再渲染B，不会在深度缓冲区写入深度值，会与颜色缓冲区中的A进行颜色混合，看起来物体B在A前面了。
因此，渲染引擎一般会对物体进行排序，再渲染，常用方法是：
先渲染所有不透明物体，并开启他们深度测试和深度写入。
把半透明物体按他们距离摄像机远近进行排序，然后按照从后往前的顺序渲染这些半透明物体，并开启他们的深度测试，关闭深度写入。
但是如果仅此而已，还是不够的，因为会有一些特殊情况。
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/10/QQ截图20191008205021.png)
而我们是根据他们离摄像机远近来排序的，而距离摄像机远近依靠的是深度值（像素级，即每个像素都有一个深度值），但我们现在是对单个物体级进行排序，所以要么A全在B前面，要么A全在B后面。
### Unity Shader的渲染顺序
Unity为了解决渲染顺序的问题，提供了渲染队列（render queue）这一解决方案、我们可以使用SubShader的Queue标签来决定我们模型将归于哪个渲染队列。
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/10/QQ截图20191008205257.png)
### 透明度测试
```GLSL
// Upgrade NOTE: replaced '_Object2World' with 'unity_ObjectToWorld'

Shader "Unity Shaders Book/Chapter8/AlphaTest"
{
    Properties
    {
        _Color("Main Tint",Color) = (1,1,1,1)
        _MainTex ("Texture", 2D) = "white" {}
        //决定我们调用clip进行透明度测试时使用的判断条件。
        //它的范围是[0,1]这是因为纹理像素的透明度就是在此范围内。
        _Cutoff("Alpha Cutoff",Range(0,1)) = 0.5
    }
    SubShader
    {
        //Queue定义归属渲染队列
        //IgnoreProjector为true表示这个Shader不会受到投影器影响
        //RenderType可以让Unity把这个Shader归入到提前定义的组
        Tags { "Queue" = "AlphaTest" "IgnoreProjector" = "True" "RenderType" = "TransparentCutout"}
        
        //关闭剔除，可以看到背面
        Cull Off
 
        Pass
        {
            //定义光照模式
            Tags { "LightMode" = "ForwardBase"}
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            #include "Lighting.cginc"

            fixed4 _Color;
            sampler2D _MainTex;
            float4 _MainTex_ST;
            fixed _Cutoff;

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
                //把顶点位置从模型空间转换到裁剪空间中
                o.pos = UnityObjectToClipPos(v.vertex);
                //计算出世界空间法线方向
                o.worldNormal = UnityObjectToWorldNormal (v.normal);
                //计算出世界空间顶点位置
                o.worldPos = mul(unity_ObjectToWorld,v.vertex).xyz;
                //计算经过平铺和偏移后的纹理坐标
                o.uv = TRANSFORM_TEX(v.texcoord, _MainTex);
                return o;
            }

            fixed4 frag (v2f i) : SV_Target
            {
                fixed3 worldNormal = normalize(i.worldNormal);
                fixed3 worldLightDir = normalize(UnityWorldSpaceLightDir(i.worldPos));
                //对纹理进行采样
                fixed4 texColor = tex2D(_MainTex,i.uv);
                //会判断texColor.a - _Cutoff是否为负，如果是就会舍弃该片元的输出
                //当texColor.a小于材质参数_Cutoff时，该片元就会产生完全透明的效果
                clip(texColor.a - _Cutoff);
                //得到环境光
                fixed3 albedo = texColor.rgb * _Color.rgb;
                //计算高光反射
                fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz * albedo;
                //计算漫反射
                fixed3 diffuse = _LightColor0.rgb * albedo*max(0,dot(worldNormal,worldLightDir));
                
                return fixed4(ambient + diffuse,1.0);
            }
            ENDCG
        }

    }
    Fallback "Transparent/Cutout/VertexLit"
}

```
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/10/QQ截图20191008224344.png)
### 透明度混合
为了进行混合，我们需要使用Unity提供的混合命令——Blend。Blend是Unity提供的设置混合模式的命令。想要实现半透明效果就需要把当前自身的颜色和已经存在于颜色缓冲中的颜色值进行混合。混合时使用的函数就是由该指令决定的。
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/10/QQ截图20191009202503.png)
```GLSL
// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'

// Upgrade NOTE: replaced '_Object2World' with 'unity_ObjectToWorld'

Shader "Unity Shaders Book/Chapter8/AlphaBlend"
{
    Properties
    {
        _Color ("Color Tint", Color) = (1, 1, 1, 1)
        _MainTex ("Main Tex", 2D) = "white" {}
        //_AlphaScale用于在透明纹理的基础上控制整体的透明度
        _AlphaScale ("Alpha Scale", Range(0, 1)) = 1
    }
    SubShader
    {
        //Queue定义归属渲染队列，因为是透明度混合，所以是Transparent队列
        //IgnoreProjector为true表示这个Shader不会受到投影器影响
        //RenderType可以让Unity把这个Shader归入到提前定义的组
        Tags {"Queue"="Transparent" "IgnoreProjector"="True" "RenderType"="Transparent"}
        
        //关闭剔除，可以看到背面
        Cull Off
 
        Pass
        {
            //定义光照模式
            Tags { "LightMode" = "ForwardBase"}
            //关闭深度写入
            ZWrite Off
            //将该片元着色器产生的颜色混合因子设置为SrcAlpha
            //将已经存在与颜色缓冲中的颜色混合因子设置为OneMinusSrcAlpha
            Blend SrcAlpha OneMinusSrcAlpha
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            #include "Lighting.cginc"

            fixed4 _Color;
            sampler2D _MainTex;
            float4 _MainTex_ST;
            fixed _AlphaScale;

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
                //把顶点位置从模型空间转换到裁剪空间中
                o.pos = UnityObjectToClipPos(v.vertex);
                //计算出世界空间法线方向
                o.worldNormal = UnityObjectToWorldNormal (v.normal);
                //计算出世界空间顶点位置
                o.worldPos = mul(unity_ObjectToWorld,v.vertex).xyz;
                //计算经过平铺和偏移后的纹理坐标
                o.uv = TRANSFORM_TEX(v.texcoord, _MainTex);
                return o;
            }

            fixed4 frag (v2f i) : SV_Target
            {
                fixed3 worldNormal = normalize(i.worldNormal);
                fixed3 worldLightDir = normalize(UnityWorldSpaceLightDir(i.worldPos));
                //对纹理进行采样
                fixed4 texColor = tex2D(_MainTex,i.uv);
                //得到环境光
                fixed3 albedo = texColor.rgb * _Color.rgb;
                //计算高光反射
                fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz * albedo;
                //计算漫反射
                fixed3 diffuse = _LightColor0.rgb * albedo*max(0,dot(worldNormal,worldLightDir));
                //设置片元着色器返回值的透明通道
                return fixed4(ambient + diffuse,texColor.a*_AlphaScale);
            }
            ENDCG
        }

    }
    Fallback "Transparent/VertexLit"
}

```
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/10/QQ截图20191009212734.png)
但是会有一种情况，得到错误的渲染结果
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/10/QQ截图20191009212948.png)
### 开启深度写入的半透明效果
为了解决上述问题，我们通常使用两个Pass进行渲染，第一个Pass开启深度写入，但不输出颜色，它的目的仅仅是为了把模型的深度值写入深度缓冲中，第二个Pass进行正常的透明度混合，由于上一个Pass已经得到了逐像素的正确的深度信息，该Pass就可以按照像素级别的深度排序结果进行透明渲染。但是`这样会消耗一定的额外性能`。
### ShaderLab的混合命令
混合的实现原理：当片元着色器产生一个颜色时，可以选择与颜色缓存中的颜色进行混合，然后重新写入颜色缓冲。（注意，我们平时谈论混合中的颜色都是RGBA四个通道的）。
在Unity中，当我们使用Blend（Blend Off命令除外）时，除了设置混合状态外，也开启了混合，但是`在其他图形API中我们是需要手动开启的`。
#### 混合等式和参数
混合是一个逐片元操作，而且他是不可编程的，但却是高度可配置的。
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/10/QQ截图20191009215205.png)
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/10/QQ截图20191009221706.png)
#### 混合操作
当把原颜色和目标颜色与他们对应的混合因子相乘后，我们都是把他们的结果加起来作为输出颜色的。我们也可以使用别的混合操作：
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/10/QQ截图20191009215205-j7x.png)
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/10/QQ截图20191009221950.png)
混合操作命令通常是与混合因子命令一起工作的。但需要注意的是，当使用Min或Max混合操作时，混合因子实际上是不起任何作用的，他们仅仅会判断原始的源颜色和目的颜色之间的比较结果。
#### 常见的混合类型
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/10/QQ截图20191009222732.png)
### 双面渲染的透明效果
和透明度测试相比。想让透明度混合实现双面渲染会更加复杂一点，这是因为关闭了深度写入。如果我们仍然直接关闭剔除功能，那么我们就无法保证同一个物体的正面和背面图元的渲染顺序。`就有可能得到错误的半透明效果。`
为此我们把双面渲染工作分成两个Pass——第一个Pass只渲染背面，第二个Pass只渲染正面，我们可以保证背面总是在正面被渲染前渲染，从而保证正确的深度渲染关系。
