---
title: Unity Shader入门精要学习笔记：非真实感渲染
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

# 前言
尽管游戏渲染一般是以照相写实主义作为主要目标，但也有一些游戏使用了`非真实感渲染（Non-Photorealistic Rendering，NPR）`,例如卡通，水彩风格等。
# 卡通风格的渲染
要实现卡通渲染有很多方法，其中之一就是使用`基于色调的着色技术`。
在卡通风格中，模型高光往往是一块块分界明显的纯色区域。
[![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/11/QQ截图20191107192120.png)](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/11/QQ截图20191107192120.png)
## 渲染轮廓线
在《Real Time Rendering, third edition》一书中，作者把这些方法分成5种类型。

- 基于观察角度和表面法线的轮廓线渲染，这种方法使用视角方向和表面法线的点乘结果来得到轮廓线的信息，可以在一个Pass中就得到渲染结果，但局限性很大，效果也不尽如人意。
- 过程式集合轮廓线渲染，使用两个Pass渲染，第一个Pass渲染背面的面片，并使用某些技术让它轮廓可见，第二个Pass再正常渲染正面的面片，这种方法快速有效，并且适用于绝大多数表面平滑的模型，缺点是不适合类似于立方体这样平整的模型。
- 基于图像处理的轮廓线渲染，适应面较广，缺点是一些深度和法线变换很小的轮廓无法被检测出来，例如桌子上的纸张。
- 基于轮廓边检测的轮廓线渲染，前几个方法最大问题就是，无法控制轮廓线的风格渲染，但有时我们希望刻意渲染出独特风格的轮廓线，例如水墨风格，所以我们希望可以检测出精确的轮廓线，然后直接渲染他们。检测一条边是否是轮廓边的公式很简单，我们只需要检查和这条边相邻的两个三角面片是否满足下面的条件：$\left(n_0\cdot v>0\right)\;\neq\;\left(n_1\cdot v>0\right)$，其中$n_0$和$n_1$分别表示两个相邻三角面片的法向，v是从视角到该边上任意顶点的方向，上述公式的本质是检查两个相邻的三角面片是否一个朝正面，一个朝背面。缺点是在帧和帧之间会出现跳跃性。
- 最后一个种类就是混合了上述的几种渲染方法，例如，先找到精确的轮廓边，把模型和轮廓边渲染到纹理中，再使用图像处理的方法识别出轮廓线，并在图像空间下进行风格化渲染。

我们将使用两个Pass渲染模型，第一个Pass中，我们使用轮廓线颜色渲染整个背面的面片，并在视角空间下把模型顶点沿着法线方向向外扩张一段距离，以此让背部轮廓线可见。
`viewPos = view + viewNormal * _Outline`
但是，对于一些内凹的模型，就有可能发生背面面片遮挡正面面片情况，所以我们在扩张背面顶点之前，我们首先对顶点法线的z分量进行处理，让他们等于一个定值，然后把法线归一化后再对顶点进行扩张。这样扩展后的背面更加扁平化，从而降低了遮挡正面面片的可能性。
`viewNormal.z = -0.5;
viewNormal = normalize(viewNormal);
viewPos = viewPos + viewNormal * _OutLine;`
## 添加高光
卡通风格中的高光往往是模型上一块块分界明显的纯色区域，所以我们计算normal和halfDir点乘结果后需要与一个阈值进行比较，如果小于该阈值，高光反射系数为0，否则返回1。
`float spec = dot(worldNormal, worldHalfDir);
//平滑处理，对高光边缘抗锯齿，还可以使用fwidth函数得到邻域像素之间的近似导数值
spec = lerp(0, 1, smoothstep(-w, w, spec - threshold));`
[![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/11/QQ截图20191107223355.png)](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/11/QQ截图20191107223355.png)
## 实现
```GLSL
// Upgrade NOTE: replaced '_Object2World' with 'unity_ObjectToWorld'
// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'

Shader "Unity Shaders Book/Chapter 14/Toon Shading" 
{
	Properties 
	{
		_Color ("Color Tint", Color) = (1, 1, 1, 1)
		_MainTex ("Main Tex", 2D) = "white" {}
		//控制漫反射色调的渐变纹理
		_Ramp ("Ramp Texture", 2D) = "white" {}
		//控制轮廓线宽度
		_Outline ("Outline", Range(0, 1)) = 0.1
		//对应轮廓线颜色
		_OutlineColor ("Outline Color", Color) = (0, 0, 0, 1)
		//高光反射颜色
		_Specular ("Specular", Color) = (1, 1, 1, 1)
		//用于控制计算高光反射时使用的阈值
		_SpecularScale ("Specular Scale", Range(0, 0.1)) = 0.01
	}
    SubShader 
    {
		Tags { "RenderType"="Opaque" "Queue"="Geometry"}
		
		Pass 
		{
		    //定义名称，方便重用
			NAME "OUTLINE"
			
			Cull Front
			
			CGPROGRAM
			
			#pragma vertex vert
			#pragma fragment frag
			
			#include "UnityCG.cginc"
			
			float _Outline;
			fixed4 _OutlineColor;
			
			struct a2v 
			{
				float4 vertex : POSITION;
				float3 normal : NORMAL;
			}; 
			
			struct v2f 
			{
			    float4 pos : SV_POSITION;
			};
			
			v2f vert (a2v v) 
			{
				v2f o;
				//将顶点变换到视角空间下
				float4 pos = mul(UNITY_MATRIX_MV, v.vertex); 
				//将法线变换到视角空间下
				float3 normal = mul((float3x3)UNITY_MATRIX_IT_MV, v.normal);  
				//避免背面扩张后的顶点挡在正面的面片
				normal.z = -0.5;
				//顶点扩张
				pos = pos + float4(normalize(normal), 0) * _Outline;
				//变换到裁剪空间
				o.pos = mul(UNITY_MATRIX_P, pos);
				
				return o;
			}
			
			float4 frag(v2f i) : SV_Target 
			{ 
				return float4(_OutlineColor.rgb, 1);               
			}
			
			ENDCG
		}
		
		Pass 
		{
			Tags { "LightMode"="ForwardBase" }
			
			Cull Back
		
			CGPROGRAM
		
			#pragma vertex vert
			#pragma fragment frag
			
			#pragma multi_compile_fwdbase
		
			#include "UnityCG.cginc"
			#include "Lighting.cginc"
			#include "AutoLight.cginc"
			#include "UnityShaderVariables.cginc"
			
			fixed4 _Color;
			sampler2D _MainTex;
			float4 _MainTex_ST;
			sampler2D _Ramp;
			fixed4 _Specular;
			fixed _SpecularScale;
		
			struct a2v 
			{
				float4 vertex : POSITION;
				float3 normal : NORMAL;
				float4 texcoord : TEXCOORD0;
				float4 tangent : TANGENT;
			}; 
		
			struct v2f 
			{
				float4 pos : POSITION;
				float2 uv : TEXCOORD0;
				float3 worldNormal : TEXCOORD1;
				float3 worldPos : TEXCOORD2;
				//计算阴影所需的各个变量
				SHADOW_COORDS(3)
			};
			
			v2f vert (a2v v) 
			{
				v2f o;
				
				o.pos = UnityObjectToClipPos( v.vertex);
				o.uv = TRANSFORM_TEX (v.texcoord, _MainTex);
				o.worldNormal  = UnityObjectToWorldNormal(v.normal);
				o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;
				//计算阴影所需的各个变量
				TRANSFER_SHADOW(o);
				
				return o;
			}
			
			float4 frag(v2f i) : SV_Target 
			{ 
				fixed3 worldNormal = normalize(i.worldNormal);
				fixed3 worldLightDir = normalize(UnityWorldSpaceLightDir(i.worldPos));
				fixed3 worldViewDir = normalize(UnityWorldSpaceViewDir(i.worldPos));
				fixed3 worldHalfDir = normalize(worldLightDir + worldViewDir);
				
				fixed4 c = tex2D (_MainTex, i.uv);
				fixed3 albedo = c.rgb * _Color.rgb;
				
				fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz * albedo;
				//计算世界坐标下阴影值
				UNITY_LIGHT_ATTENUATION(atten, i, i.worldPos);
				
				fixed diff =  dot(worldNormal, worldLightDir);
				diff = (diff * 0.5 + 0.5) * atten;
				//最终的漫反射系数
				fixed3 diffuse = _LightColor0.rgb * albedo * tex2D(_Ramp, float2(diff, diff)).rgb;
				
				fixed spec = dot(worldNormal, worldHalfDir);
				//抗锯齿处理
				fixed w = fwidth(spec) * 2.0;
				fixed3 specular = _Specular.rgb * lerp(0, 1, smoothstep(-w, w, spec + _SpecularScale - 1)) * step(0.0001, _SpecularScale);
				
				return fixed4(ambient + diffuse + specular, 1.0);
			}
		
			ENDCG
		}
	}
	FallBack "Diffuse"
}

```

# 素描风格的渲染
Praum等人发表的一篇著名的论文，他们使用提前生成的素描纹理来实现实时的素描风格渲染，这些纹理组成了一个色调艺术映射。
[![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/11/QQ截图20191109230116.png)](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/11/QQ截图20191109230116.png)
从左到右的笔触逐渐增多，用于模拟不同光照下的漫反射效果，从上到下则对应每张纹理的多级渐远纹理（`并不是简单地对上一层纹理进行降采样，而是需要保持笔触之间的间隔，以便更加真实的模拟素描效果`）
我们首先在顶点着色器阶段计算逐顶点的光照，然后在片元着色器中根据权重来混合6张纹理的采样结果。
```GLSL
// Upgrade NOTE: replaced '_Object2World' with 'unity_ObjectToWorld'
// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'

///
///  Reference: 	Praun E, Hoppe H, Webb M, et al. Real-time hatching[C]
///						Proceedings of the 28th annual conference on Computer graphics and interactive techniques. ACM, 2001: 581.
///
Shader "Unity Shaders Book/Chapter 14/Hatching" 
{
	Properties 
	{
		_Color ("Color Tint", Color) = (1, 1, 1, 1)
		_TileFactor ("Tile Factor", Float) = 1
		_Outline ("Outline", Range(0, 1)) = 0.1
		_Hatch0 ("Hatch 0", 2D) = "white" {}
		_Hatch1 ("Hatch 1", 2D) = "white" {}
		_Hatch2 ("Hatch 2", 2D) = "white" {}
		_Hatch3 ("Hatch 3", 2D) = "white" {}
		_Hatch4 ("Hatch 4", 2D) = "white" {}
		_Hatch5 ("Hatch 5", 2D) = "white" {}
	}
	
	SubShader 
	{
		Tags { "RenderType"="Opaque" "Queue"="Geometry"}
		
		UsePass "Unity Shaders Book/Chapter 14/Toon Shading/OUTLINE"
		
		Pass 
		{
			Tags { "LightMode"="ForwardBase" }
			
			CGPROGRAM
			
			#pragma vertex vert
			#pragma fragment frag 
			
			#pragma multi_compile_fwdbase
			
			#include "UnityCG.cginc"
			#include "Lighting.cginc"
			#include "AutoLight.cginc"
			#include "UnityShaderVariables.cginc"
			//控制模型颜色
			fixed4 _Color;
			//纹理的平铺系数，值越大，模型上的素描线条越密
			float _TileFactor;
			sampler2D _Hatch0;
			sampler2D _Hatch1;
			sampler2D _Hatch2;
			sampler2D _Hatch3;
			sampler2D _Hatch4;
			sampler2D _Hatch5;
			
			struct a2v 
			{
				float4 vertex : POSITION;
				float4 tangent : TANGENT; 
				float3 normal : NORMAL; 
				float2 texcoord : TEXCOORD0; 
			};
			
			struct v2f 
			{
				float4 pos : SV_POSITION;
				float2 uv : TEXCOORD0;
				fixed3 hatchWeights0 : TEXCOORD1;
				fixed3 hatchWeights1 : TEXCOORD2;
				float3 worldPos : TEXCOORD3;
				SHADOW_COORDS(4)
			};
			
			v2f vert(a2v v)
			{
				v2f o;
				
				o.pos = UnityObjectToClipPos(v.vertex);
				//使用__TileFactor得到纹理采样坐标
				o.uv = v.texcoord.xy * _TileFactor;
				//计算6张纹理的混合权重之前，需要先计算逐顶点光照
				fixed3 worldLightDir = normalize(WorldSpaceLightDir(v.vertex));
				fixed3 worldNormal = UnityObjectToWorldNormal(v.normal);
				fixed diff = max(0, dot(worldLightDir, worldNormal));
                //权值初始化为0
				o.hatchWeights0 = fixed3(0, 0, 0);
				o.hatchWeights1 = fixed3(0, 0, 0);
				//把diff缩放到[0,7]
				float hatchFactor = diff * 7.0;
				//通过判断hatchFactor所处子区间来计算对应的纹理混合权重
				if (hatchFactor > 6.0) 
				{
					// Pure white, do nothing
				} 
				else if (hatchFactor > 5.0) 
				{
					o.hatchWeights0.x = hatchFactor - 5.0;
				} 
				else if (hatchFactor > 4.0) 
				{
					o.hatchWeights0.x = hatchFactor - 4.0;
					o.hatchWeights0.y = 1.0 - o.hatchWeights0.x;
				} 
				else if (hatchFactor > 3.0) 
				{
					o.hatchWeights0.y = hatchFactor - 3.0;
					o.hatchWeights0.z = 1.0 - o.hatchWeights0.y;
				} 
				else if (hatchFactor > 2.0) 
				{
					o.hatchWeights0.z = hatchFactor - 2.0;
					o.hatchWeights1.x = 1.0 - o.hatchWeights0.z;
				} 
				else if (hatchFactor > 1.0) 
				{
					o.hatchWeights1.x = hatchFactor - 1.0;
					o.hatchWeights1.y = 1.0 - o.hatchWeights1.x;
				} 
				else 
				{
					o.hatchWeights1.y = hatchFactor;
					o.hatchWeights1.z = 1.0 - o.hatchWeights1.y;
				}
				
				o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;
				//计算阴影纹理的采样坐标
				TRANSFER_SHADOW(o);
				
				return o; 
			}
			
			fixed4 frag(v2f i) : SV_Target 
			{			
				fixed4 hatchTex0 = tex2D(_Hatch0, i.uv) * i.hatchWeights0.x;
				fixed4 hatchTex1 = tex2D(_Hatch1, i.uv) * i.hatchWeights0.y;
				fixed4 hatchTex2 = tex2D(_Hatch2, i.uv) * i.hatchWeights0.z;
				fixed4 hatchTex3 = tex2D(_Hatch3, i.uv) * i.hatchWeights1.x;
				fixed4 hatchTex4 = tex2D(_Hatch4, i.uv) * i.hatchWeights1.y;
				fixed4 hatchTex5 = tex2D(_Hatch5, i.uv) * i.hatchWeights1.z;
				fixed4 whiteColor = fixed4(1, 1, 1, 1) * (1 - i.hatchWeights0.x - i.hatchWeights0.y - i.hatchWeights0.z - 
							i.hatchWeights1.x - i.hatchWeights1.y - i.hatchWeights1.z);
				
				fixed4 hatchColor = hatchTex0 + hatchTex1 + hatchTex2 + hatchTex3 + hatchTex4 + hatchTex5 + whiteColor;
				
				UNITY_LIGHT_ATTENUATION(atten, i, i.worldPos);
								
				return fixed4(hatchColor.rgb * _Color.rgb * atten, 1.0);
			}
			
			ENDCG
		}
	}
	FallBack "Diffuse"
}

```
[![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/11/QQ截图20191109230420.png)](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/11/QQ截图20191109230420.png)

# 拓展阅读
[https://blog.csdn.net/jvandc/article/details/81171250](https://blog.csdn.net/jvandc/article/details/81171250)
