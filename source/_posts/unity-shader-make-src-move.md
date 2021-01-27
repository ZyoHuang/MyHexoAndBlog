---
title: Unity Shader入门精要学习笔记：让画面动起来
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
cover: https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/10/9-rop.gif
aplayer:
---
<meta name="referrer" content="no-referrer" />

# Unity Shader中的时间变量
动画效果往往都是把时间添加到一些变量的计算中，以便在时间变化时，画面也可以随之变化。Unity Shader提供了一系列关于时间的内置变量来允许我们方便地在Shader中访问运行时间。
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/10/QQ截图20191019211235.png)
# 纹理动画
## 序列帧动画
要播放帧动画，我们需要计算出每个时刻需要播放的关键帧在纹理中的位置。
```GLSL
// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'

Shader "Unity Shaders Book/Chapter 11/Image Sequence Animation" 
{
	Properties 
	{
		_Color ("Color Tint", Color) = (1, 1, 1, 1)
		_MainTex ("Image Sequence", 2D) = "white" {}
    	_HorizontalAmount ("Horizontal Amount", Float) = 4
    	_VerticalAmount ("Vertical Amount", Float) = 4
    	_Speed ("Speed", Range(1, 100)) = 30
	}
	SubShader 
	{
	    //序列帧图像通常包含了透明通道，因此可以被当成一个半透明对象
		Tags {"Queue"="Transparent" "IgnoreProjector"="True" "RenderType"="Transparent"}
		
		Pass 
		{
			Tags { "LightMode"="ForwardBase" }
			//关闭深度写入
			ZWrite Off
			//开启混合
			Blend SrcAlpha OneMinusSrcAlpha
			
			CGPROGRAM
			
			#pragma vertex vert  
			#pragma fragment frag
			
			#include "UnityCG.cginc"
			
			fixed4 _Color;
			sampler2D _MainTex;
			float4 _MainTex_ST;
			float _HorizontalAmount;
			float _VerticalAmount;
			float _Speed;
			  
			struct a2v 
			{  
			    float4 vertex : POSITION; 
			    float2 texcoord : TEXCOORD0;
			};  
			
			struct v2f 
			{  
			    float4 pos : SV_POSITION;
			    float2 uv : TEXCOORD0;
			};  
			
			v2f vert (a2v v) 
			{  
				v2f o;  
				o.pos = UnityObjectToClipPos(v.vertex);  
				o.uv = TRANSFORM_TEX(v.texcoord, _MainTex);  
				return o;
			}  
			
			fixed4 frag (v2f i) : SV_Target 
			{
			    //返回小于等于模拟时间的最大整数
				float time = floor(_Time.y * _Speed);  
				//依据时间设置行列索引
				//商为行
				float row = floor(time / _HorizontalAmount);
				//余数为列
				float column = time - row * _HorizontalAmount;
				//竖直方向（行索引）需要做减法，
				//因为Unity纹理坐标竖直方向顺序（从下到上递增）和序列帧纹理（从上到下递增）
				half2 uv = i.uv + half2(column, -row);
				uv.x /=  _HorizontalAmount;
				uv.y /= _VerticalAmount;
				//取样
				fixed4 c = tex2D(_MainTex, uv);
				//颜色混合
				c.rgb *= _Color;
				
				return c;
			}
			
			ENDCG
		}  
	}
	FallBack "Transparent/VertexLit"
}

```
## 滚动的背景
```GLSL
// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'

Shader "Unity Shaders Book/Chapter 11/Scrolling Background" 
{
	Properties 
	{
		_MainTex ("Base Layer (RGB)", 2D) = "white" {}
		_DetailTex ("2nd Layer (RGB)", 2D) = "white" {}
		_ScrollX ("Base layer Scroll Speed", Float) = 1.0
		_Scroll2X ("2nd layer Scroll Speed", Float) = 1.0
		//控制纹理的整体亮度
		_Multiplier ("Layer Multiplier", Float) = 1
	}
	SubShader 
	{
		Tags { "RenderType"="Opaque" "Queue"="Geometry"}
		
		Pass 
		{ 
			Tags { "LightMode"="ForwardBase" }
			
			CGPROGRAM
			
			#pragma vertex vert
			#pragma fragment frag
			
			#include "UnityCG.cginc"
			
			sampler2D _MainTex;
			sampler2D _DetailTex;
			float4 _MainTex_ST;
			float4 _DetailTex_ST;
			float _ScrollX;
			float _Scroll2X;
			float _Multiplier;
			
			struct a2v 
			{
				float4 vertex : POSITION;
				float4 texcoord : TEXCOORD0;
			};
			
			struct v2f 
			{
				float4 pos : SV_POSITION;
				float4 uv : TEXCOORD0;
			};
			
			v2f vert (a2v v) 
			{
				v2f o;
				o.pos = UnityObjectToClipPos(v.vertex);
				
				o.uv.xy = TRANSFORM_TEX(v.texcoord, _MainTex) + frac(float2(_ScrollX, 0.0) * _Time.y);
				o.uv.zw = TRANSFORM_TEX(v.texcoord, _DetailTex) + frac(float2(_Scroll2X, 0.0) * _Time.y);
				
				return o;
			}
			
			fixed4 frag (v2f i) : SV_Target 
			{
				fixed4 firstLayer = tex2D(_MainTex, i.uv.xy);
				fixed4 secondLayer = tex2D(_DetailTex, i.uv.zw);
				
				fixed4 c = lerp(firstLayer, secondLayer, secondLayer.a);
				c.rgb *= _Multiplier;
				
				return c;
			}
			
			ENDCG
		}
	}
	FallBack "VertexLit"
}

```
# 顶点动画
## 流动的河流
```GLSL
// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'

Shader "Unity Shaders Book/Chapter 11/Water" 
{
	Properties 
	{
		_MainTex ("Main Tex", 2D) = "white" {}
		_Color ("Color Tint", Color) = (1, 1, 1, 1)
		_Magnitude ("Distortion Magnitude", Float) = 1
 		_Frequency ("Distortion Frequency", Float) = 1
 		_InvWaveLength ("Distortion Inverse Wave Length", Float) = 10
 		_Speed ("Speed", Float) = 0.5
	}
	SubShader 
	{
		// 批处理会合并所有相关模型，而这些模型各自模型空间就会丢失，所以要关闭批处理
		Tags {"Queue"="Transparent" "IgnoreProjector"="True" "RenderType"="Transparent" "DisableBatching"="True"}
		
		Pass 
		{
			Tags { "LightMode"="ForwardBase" }
			
			ZWrite Off
			Blend SrcAlpha OneMinusSrcAlpha
			//关闭剔除
			Cull Off
			
			CGPROGRAM  
			#pragma vertex vert 
			#pragma fragment frag
			
			#include "UnityCG.cginc" 
			
			sampler2D _MainTex;
			float4 _MainTex_ST;
			fixed4 _Color;
			float _Magnitude;
			float _Frequency;
			float _InvWaveLength;
			float _Speed;
			
			struct a2v 
			{
				float4 vertex : POSITION;
				float4 texcoord : TEXCOORD0;
			};
			
			struct v2f 
			{
				float4 pos : SV_POSITION;
				float2 uv : TEXCOORD0;
			};
			
			v2f vert(a2v v) 
			{
				v2f o;
				
				float4 offset;
				offset.yzw = float3(0.0, 0.0, 0.0);
				offset.x = sin(_Frequency * _Time.y + v.vertex.x * _InvWaveLength + v.vertex.y * _InvWaveLength + v.vertex.z * _InvWaveLength) * _Magnitude;
				o.pos = UnityObjectToClipPos(v.vertex + offset);
				
				o.uv = TRANSFORM_TEX(v.texcoord, _MainTex);
				o.uv +=  float2(0.0, _Time.y * _Speed);
				
				return o;
			}
			
			fixed4 frag(v2f i) : SV_Target 
			{
				fixed4 c = tex2D(_MainTex, i.uv);
				c.rgb *= _Color.rgb;
				
				return c;
			} 
			
			ENDCG
		}
	}
	FallBack "Transparent/VertexLit"
}

```
## 广告牌
广告牌技术会根据视角方向来旋转一个被纹理着色的多边形，使多边形看起来总是面对照相机。
本质是构建旋转矩阵，而我们知道一个变换矩阵需要三个基向量。
广告牌技术使用的基向量就是`表面法线`，`指向上的方向`以及`指向右的方向`，除此之外，还需要制定一个锚点，这个锚点在旋转过程中是固定不变的，以此来确定多边形在空间中的位置。
难点在于：`如何构建3个相互正交的基向量`
正常过程为：
通过初始计算得到目标的表面法线和指向上的方向
`两者往往不是垂直的，但是两者之一是固定的`
首先，我们根据初始的表面法线和指向上的方向来计算出目标方向的指向右的方向（通过叉积），并且归一化
再由法线方向和指向右的方向计算出正交的指向上的方向即可
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/10/QQ截图20191019231004.png)
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/10/QQ截图20191019231025.png)
```GLSL
// Upgrade NOTE: replaced '_World2Object' with 'unity_WorldToObject'
// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'

Shader "Unity Shaders Book/Chapter 11/Billboard" 
{
	Properties 
	{
		_MainTex ("Main Tex", 2D) = "white" {}
		_Color ("Color Tint", Color) = (1, 1, 1, 1)
		_VerticalBillboarding ("Vertical Restraints", Range(0, 1)) = 1 
	}
	SubShader 
	{
		// 因为顶点动画，需要关闭合批功能
		// 这里具体是因为我们需要使用物体的模型空间下的位置来作为锚点进行计算
		Tags {"Queue"="Transparent" "IgnoreProjector"="True" "RenderType"="Transparent" "DisableBatching"="True"}
		
		Pass 
		{ 
			Tags { "LightMode"="ForwardBase" }
			
			ZWrite Off
			Blend SrcAlpha OneMinusSrcAlpha
			Cull Off
		
			CGPROGRAM
			
			#pragma vertex vert
			#pragma fragment frag
			
			#include "Lighting.cginc"
			
			sampler2D _MainTex;
			float4 _MainTex_ST;
			fixed4 _Color;
			fixed _VerticalBillboarding;
			
			struct a2v 
			{
				float4 vertex : POSITION;
				float4 texcoord : TEXCOORD0;
			};
			
			struct v2f 
			{
				float4 pos : SV_POSITION;
				float2 uv : TEXCOORD0;
			};
			
			v2f vert (a2v v) 
			{
				v2f o;
				
				// 选择模型空间的原点作为广告牌的锚点
				float3 center = float3(0, 0, 0);
				// 利用内置变量获取模型空间下的视角位置
				float3 viewer = mul(unity_WorldToObject,float4(_WorldSpaceCameraPos, 1));
				// 根据观察位置和锚点计算目标法线方向
				float3 normalDir = viewer - center;
				// 根据_VerticalBillboarding属性来控制垂直方向上的约束度
				normalDir.y =normalDir.y * _VerticalBillboarding;
				// 归一化得到单位矢量
				normalDir = normalize(normalDir);
				// 粗略向上的方向，为了防止法线方向和向上方向平行（如果平行，叉积得到的结果将会是错误的）
				float3 upDir = abs(normalDir.y) > 0.999 ? float3(0, 0, 1) : float3(0, 1, 0);
				// 根据法线方向和粗略向上方向得到向右方向
				float3 rightDir = normalize(cross(upDir, normalDir));
				// 根据准确的法线方向和向右方向得到最后的向上方向
				upDir = normalize(cross(normalDir, rightDir));
				
				// 相对于锚点的偏移量
				float3 centerOffs = v.vertex.xyz - center;
				// 计算新的顶点位置
				float3 localPos = center + rightDir * centerOffs.x + upDir * centerOffs.y + normalDir * centerOffs.z;
                // 模型空间到裁剪空间
				o.pos = UnityObjectToClipPos(float4(localPos, 1));
				o.uv = TRANSFORM_TEX(v.texcoord,_MainTex);

				return o;
			}
			
			fixed4 frag (v2f i) : SV_Target 
			{
				fixed4 c = tex2D (_MainTex, i.uv);
				c.rgb *= _Color.rgb;
				
				return c;
			}
			
			ENDCG
		}
	} 
	FallBack "Transparent/VertexLit"
}

```
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/10/9-rop.gif)
## 注意事项
`如果我们在模型空间下进行了一些顶点动画，那么批处理往往会破坏这种动画效果。需要DisableBatching标签来强制取消。`
这样会导致Draw Call增加，性能下降。
应该尽量避免使用模型空间下的一些绝对位置和方向来进行计算。
就像广告牌例子，我们可以使用顶点颜色来存储每个顶点到锚点的距离值。
`如果要对包含了顶点动画的物体添加阴影，需要提供一个自定义的ShadowCaster Pass，因为Unity内置的并没有进行顶点动画，会导致阴影还是基于之前的顶点数据。`
```GLSL
Pass 
		{
			Tags { "LightMode" = "ShadowCaster" }
			
			CGPROGRAM
			
			#pragma vertex vert
			#pragma fragment frag
			
			#pragma multi_compile_shadowcaster
			
			#include "UnityCG.cginc"
			
			float _Magnitude;
			float _Frequency;
			float _InvWaveLength;
			float _Speed;
			
			struct v2f 
			{ 
			    V2F_SHADOW_CASTER;
			};
			
			v2f vert(appdata_base v) 
			{
				v2f o;
				
				float4 offset;
				offset.yzw = float3(0.0, 0.0, 0.0);
				offset.x = sin(_Frequency * _Time.y + v.vertex.x * _InvWaveLength + v.vertex.y * _InvWaveLength + v.vertex.z * _InvWaveLength) * _Magnitude;
				v.vertex = v.vertex + offset;

				TRANSFER_SHADOW_CASTER_NORMALOFFSET(o)
				
				return o;
			}
			
			fixed4 frag(v2f i) : SV_Target 
			{
			    SHADOW_CASTER_FRAGMENT(i)
			}
			ENDCG
		}
```
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/10/9.gif)
