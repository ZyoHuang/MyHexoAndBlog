---
title: Unity Shader入门精要学习笔记：高级纹理
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

# 高级纹理
## 立方体纹理
立方体纹理是环境映射的一种实现方式，和之前使用二维纹理坐标不同，对立方体纹理采样我们需要提供一个三维的纹理坐标，这个三维纹理坐标表示了我们在世界空间下的一个3D方向，这个方向矢量从立方体的中心出发，当它向外部延伸时就会和立方体的6个纹理之一发生相交，而采样得到的结果就是由该交点得来的。
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/10/QQ截图20191014204155.png)
它的实现简单快速，而且得到的效果也比较好，但它也有一些缺点，例如当场景发生变化时，我们就需要重新生成立方体纹理，并且立方体纹理也仅可以反射环境，但不能反射使用了该立方体纹理的物体本身。
`我们应该尽量对凸面体而不要对凹面体使用立方体纹理。（因为凹面体会反射自身）。`
### 天空盒
天空盒是在所有不透明物体之后渲染的，而背后使用的网格是一个立方体或者是一个细分后的球体。
### 创建用于环境映射的立方体纹理
第一种方法是直接由一些特殊布局的纹理创建。
我们只需要把纹理的Texture Type设置为Cubemap即可。可以对纹理数据进行压缩，而且可以支持边缘修正，光滑反射（glossy reflection）和HDR功能。
第二种方法是由脚本生成。
我们往往希望根据物体在场景中位置的不同，生成他们各自不同的立方体纹理。
这可以通过Unity提供的Camera.RenderToCubemap实现，Camera.RenderToCubemap可以把从任意位置观察到的场景图像存储到6张图像中，从而创建出该位置上对应的立方体纹理。
```csharp
//------------------------------------------------------------
// Author: 烟雨迷离半世殇
// Mail: 1778139321@qq.com
// Data: 2019年10月14日 20:53:49
//------------------------------------------------------------

using UnityEngine;
using UnityEditor;

//ScriptableWizard是EditorWindow的简化版
public class RenderCubemapWizard : ScriptableWizard 
{
	
    public Transform renderFromPosition;
    public Cubemap cubemap;
	
    void OnWizardUpdate () 
	{
        helpString = "选择要生成立方体纹理的物体和Cubemap资源";
        isValid = (renderFromPosition != null) && (cubemap != null);
    }
	
    void OnWizardCreate () 
	{
        GameObject go = new GameObject( "CubemapCamera");
        go.AddComponent<Camera>();
        go.transform.position = renderFromPosition.position;
        go.GetComponent<Camera>().RenderToCubemap(cubemap);
        //这个最好只在编辑器模式下使用
        DestroyImmediate( go );
    }
	
    [MenuItem("GameObject/Render into Cubemap")]
    static void RenderCubemap () 
	{
        ScriptableWizard.DisplayWizard<RenderCubemapWizard>(
            "Render cubemap", "Render!");
    }
}
```
### 反射
使用了反射效果的物体看起来像是镀了一层金属，想要模拟反射效果很简单，我们只需要`通过入射光线的方向和表面法线方向来计算反射方向，再利用反射方向对立方体纹理采样即可。`
```GLSL
// Upgrade NOTE: replaced '_Object2World' with 'unity_ObjectToWorld'
// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'

Shader "Unity Shaders Book/Chapter 10/Reflection"
{
    Properties
    {
        _Color("Color Tint", Color) = (1,1,1,1)
        //反射颜色
        _ReflectColor ("Reflection Color", Color) = (1,1,1,1)
        //反射程度
        _ReflectAmount ("Reflect Amount", Range(0,1)) = 1
        //环境映射纹理
        _Cubemap ("Reflection Cubemap", Cube) = "_Skybox"{}
    }
    SubShader
    {
        Tags { "RenderType"="Opaque" }

        Pass
        {
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #pragma multi_compile_fwdbase

            #include "Lighting.cginc"
            #include "AutoLight.cginc"
            fixed4 _Color;
            fixed4 _ReflectColor;
            fixed _ReflectAmount;
            samplerCUBE _Cubemap;

            struct a2v
            {
                float4 vertex : POSITION;
                float3 normal : NORMAL;
            };

            struct v2f
            {
                float4 pos : SV_POSITION;
                float3 worldPos : TEXCOORD0;
                fixed3 worldNormal : TEXCOORD1;
                fixed3 worldViewDir : TEXCOORD2;
                fixed3 worldRefl : TEXCOORD3;
                SHADOW_COORDS(4)
            };


            v2f vert (a2v v)
            {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                o.worldNormal = UnityObjectToWorldNormal(v.normal);
                o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;
                o.worldViewDir = UnityWorldSpaceViewDir(o.worldPos);
                //计算顶点处的反射方向
                o.worldRefl = reflect(-o.worldViewDir, o.worldNormal);
                
                TRANSFER_SHADOW(o);
                
                return o;
            }

            fixed4 frag (v2f i) : SV_Target
            {
                fixed3 worldNormal = normalize(i.worldNormal);
                fixed3 worldLightDir = normalize(UnityWorldSpaceLightDir(i.worldPos));
                fixed3 worldViewDir = normalize(i.worldViewDir);
                fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;
                fixed3 diffuse = _LightColor0.rgb * _Color.rgb * max(0, dot(worldNormal, worldLightDir));
                //利用反射方向来对立方体纹理采样
                fixed3 reflection = texCUBE(_Cubemap, i.worldRefl).rgb * _ReflectColor.rgb;
                
                UNITY_LIGHT_ATTENUATION(atten, i ,i.worldPos);
                
                fixed3 color = ambient + lerp(diffuse, reflection, _ReflectAmount)* atten;
                
                return fixed4(color, 1.0);
            }
            ENDCG
        }
    }
    
    Fallback "Reflective/VertexLit"
}

```
### 折射
我们可以使用`斯涅耳定律`来计算反射角。
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/10/QQ截图20191014213612.png)
$$\eta_1\sin\left(\theta_1\right)\;=\;\eta_2\sin\left(\theta_2\right)$$
```GLSL
// Upgrade NOTE: replaced '_Object2World' with 'unity_ObjectToWorld'
// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'

Shader "Unity Shaders Book/Chapter 10/Refraction" 
{
	Properties 
	{
		_Color ("Color Tint", Color) = (1, 1, 1, 1)
		_RefractColor ("Refraction Color", Color) = (1, 1, 1, 1)
		_RefractAmount ("Refraction Amount", Range(0, 1)) = 1
		//不同介质的透射比，用以计算折射方向
		_RefractRatio ("Refraction Ratio", Range(0.1, 1)) = 0.5
		_Cubemap ("Refraction Cubemap", Cube) = "_Skybox" {}
	}
	SubShader 
	{
		Tags { "RenderType"="Opaque" "Queue"="Geometry"}
		
		Pass 
		{ 
			Tags { "LightMode"="ForwardBase" }
		
			CGPROGRAM
			
			#pragma multi_compile_fwdbase	
			
			#pragma vertex vert
			#pragma fragment frag
			
			#include "Lighting.cginc"
			#include "AutoLight.cginc"
			
			fixed4 _Color;
			fixed4 _RefractColor;
			float _RefractAmount;
			fixed _RefractRatio;
			samplerCUBE _Cubemap;
			
			struct a2v {
				float4 vertex : POSITION;
				float3 normal : NORMAL;
			};
			
			struct v2f {
				float4 pos : SV_POSITION;
				float3 worldPos : TEXCOORD0;
				fixed3 worldNormal : TEXCOORD1;
				fixed3 worldViewDir : TEXCOORD2;
				fixed3 worldRefr : TEXCOORD3;
				SHADOW_COORDS(4)
			};
			
			v2f vert(a2v v) 
			{
				v2f o;
				o.pos = UnityObjectToClipPos(v.vertex);
				
				o.worldNormal = UnityObjectToWorldNormal(v.normal);
				
				o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;
				
				o.worldViewDir = UnityWorldSpaceViewDir(o.worldPos);
				
				//计算折射方向
				//参数一是归一化后的入射光线方向
				//参数二是故意化后的表面法线方向
				//参数三是入射光线所在介质的折射率和折射光线所在介质的折射率之间的比值
				o.worldRefr = refract(-normalize(o.worldViewDir), normalize(o.worldNormal), _RefractRatio);
				
				TRANSFER_SHADOW(o);
				
				return o;
			}
			
			fixed4 frag(v2f i) : SV_Target 
			{
				fixed3 worldNormal = normalize(i.worldNormal);
				fixed3 worldLightDir = normalize(UnityWorldSpaceLightDir(i.worldPos));
				fixed3 worldViewDir = normalize(i.worldViewDir);
								
				fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;
				
				fixed3 diffuse = _LightColor0.rgb * _Color.rgb * max(0, dot(worldNormal, worldLightDir));
				
				// 使用在世界空间的折射方向来对立方体纹理采样
				fixed3 refraction = texCUBE(_Cubemap, i.worldRefr).rgb * _RefractColor.rgb;
				
				UNITY_LIGHT_ATTENUATION(atten, i, i.worldPos);
				
				//把折射颜色和漫反射颜色混合
				fixed3 color = ambient + lerp(diffuse, refraction, _RefractAmount) * atten;
				
				return fixed4(color, 1.0);
			}
			
			ENDCG
		}
	} 
	FallBack "Reflective/VertexLit"
}
```
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/10/QQ截图20191014213732.png)
### 菲涅尔反射
菲涅尔反射描述了一种光学现象，即当光线照射到物体表面时，一部分发生反射，一部分进入物体内部，发生散射或者折射。
被反射的光和入射光之间存在一定的比率关系，这个比率关系可以用菲涅尔等式计算。
`Schlick菲涅尔近似等式`：
F是一个反射系数，用于控制菲涅尔反射强度，v是视角方向，n是表面法线。
$$F_{schlick}(v,n)\;=\;F_0\;+\;(1\;-\;F_0){(1\;-\;v\cdot n)}^5$$
`Empricial菲涅尔近似等式`：
bias，scale，power是控制项
$$F_{Empricial}(v,n)\;=\;max(0,min(1,bias+scale\times{(1-v\cdot n)}^{power}))$$
使用Schlick菲涅尔近似等式：
```GLSL
// Upgrade NOTE: replaced '_Object2World' with 'unity_ObjectToWorld'
// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'

Shader "Unity Shaders Book/Chapter 10/Fresnel" 
{
	Properties 
	{
		_Color ("Color Tint", Color) = (1, 1, 1, 1)
		_FresnelScale ("Fresnel Scale", Range(0, 1)) = 0.5
		_Cubemap ("Reflection Cubemap", Cube) = "_Skybox" {}
	}
	SubShader 
	{
		Tags { "RenderType"="Opaque" "Queue"="Geometry"}
		
		Pass 
		{ 
			Tags { "LightMode"="ForwardBase" }
		
			CGPROGRAM
			
			#pragma multi_compile_fwdbase
			
			#pragma vertex vert
			#pragma fragment frag
			
			#include "Lighting.cginc"
			#include "AutoLight.cginc"
			
			fixed4 _Color;
			fixed _FresnelScale;
			samplerCUBE _Cubemap;
			
			struct a2v 
			{
				float4 vertex : POSITION;
				float3 normal : NORMAL;
			};
			
			struct v2f 
			{
				float4 pos : SV_POSITION;
				float3 worldPos : TEXCOORD0;
  				fixed3 worldNormal : TEXCOORD1;
  				fixed3 worldViewDir : TEXCOORD2;
  				fixed3 worldRefl : TEXCOORD3;
 	 			SHADOW_COORDS(4)
			};
			
			v2f vert(a2v v) 
			{
				v2f o;
				o.pos = UnityObjectToClipPos(v.vertex);
				//世界空间下的法线方向
				o.worldNormal = UnityObjectToWorldNormal(v.normal);
				//世界空间下的位置
				o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;
				//世界空间下的视角方向
				o.worldViewDir = UnityWorldSpaceViewDir(o.worldPos);
				//世界空间下的反射方向
				o.worldRefl = reflect(-o.worldViewDir, o.worldNormal);
				
				TRANSFER_SHADOW(o);
				
				return o;
			}
			
			fixed4 frag(v2f i) : SV_Target 
			{
				fixed3 worldNormal = normalize(i.worldNormal);
				//世界空间下的光照方向（归一化）
				fixed3 worldLightDir = normalize(UnityWorldSpaceLightDir(i.worldPos));
				fixed3 worldViewDir = normalize(i.worldViewDir);
				
				fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;
				
				UNITY_LIGHT_ATTENUATION(atten, i, i.worldPos);
				
				fixed3 reflection = texCUBE(_Cubemap, i.worldRefl).rgb;
				
				//使用Schlick菲涅尔近似等式来计算fresnel变量
				fixed fresnel = _FresnelScale + (1 - _FresnelScale) * pow(1 - dot(worldViewDir, worldNormal), 5);
				
				fixed3 diffuse = _LightColor0.rgb * _Color.rgb * max(0, dot(worldNormal, worldLightDir));
				
				fixed3 color = ambient + lerp(diffuse, reflection, saturate(fresnel)) * atten;
				
				return fixed4(color, 1.0);
			}
			
			ENDCG
		}
	} 
	FallBack "Reflective/VertexLit"
}
```
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/10/QQ截图20191014221213.png)
## 渲染纹理
一个摄像机的渲染结果会输出到颜色缓冲中，并显示到我们的屏幕上。
现代的GPU允许我们把整个三维场景渲染到一个`中间缓冲`中，即`渲染目标纹理`（Render Target Texture，RTT），而不是传统的帧缓冲或后备缓冲（back buffer）。与之相关的是`多重渲染目标纹理`（Multiple Render Target，MRT）这种技术指的是GPU允许我们把场景同时渲染到多个渲染目标纹理中，而不再需要为每个渲染目标纹理单独渲染完整的场景。
延迟渲染就是使用多重渲染目标的一个应用。
Unity为渲染目标纹理定义了一种专门的纹理类型——渲染纹理（Render Texture）。
使用方式有二

1. 直接创建一个渲染纹理，然后把某个摄像机的渲染目标设置成该渲染纹理。
2. 在屏幕后处理时使用GrabPass命令或OnRenderImage函数来获取当前屏幕图像，Unity会把这个屏幕图像放到一张和屏幕分辨率等同的渲染纹理中，下面我们可以在自定义Pass中把他们当成普通的纹理来处理，从而实现各种屏幕特效。

### 镜子效果
创建一个shader，用于左右反转uv，创建一个Render Texture，用以接收摄像机传来的纹理，额外创建一个摄像机，用以向Render Texture传递纹理。
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/10/QQ截图20191019140950.png)
### 玻璃效果
在Shader中定义一个GrabPass后，Unity会把当前屏幕的图像绘制在一张纹理中，以便我们在后续的Pass中继续访问它。
使用GrabPass可以让我们对该物体后面的图像进行更加复杂的处理，例如使用法线来模拟折射效果，而不再是简单的和原屏幕颜色进行混合。
在使用GrabPass时，我们需要额外小心`物体的渲染队列设置`。GrabPass通常用于渲染透明物体，尽管代码里并不包含混合指令，但我们往往仍然需要把物体的渲染队列设置成透明队列（"Queue" = "Transparent"）。这样才可以保证当渲染该物体时，所有不透明物体都已经被绘制到屏幕上。
大体思路：首先使用一张法线纹理来修改模型的法线信息，然后通过一个Cubemap来模拟玻璃的反射，而在模拟折射时，则使用了GrabPass获取玻璃后的屏幕图像，并使用切线空间下的法线对屏幕纹理坐标偏移后，再对屏幕图像进行采样来模拟近似的折射效果。
```GLSL
// Upgrade NOTE: replaced '_Object2World' with 'unity_ObjectToWorld'
// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'

Shader "Unity Shaders Book/Chapter 10/Glass Refraction" 
{
	Properties 
	{
	    //玻璃的材质纹理
		_MainTex ("Main Tex", 2D) = "white" {}
		//玻璃的法线纹理
		_BumpMap ("Normal Map", 2D) = "bump" {}
		//用于模拟反射的环境纹理
		_Cubemap ("Environment Cubemap", Cube) = "_Skybox" {}
		//控制模拟折射时图像的扭曲程度
		_Distortion ("Distortion", Range(0, 100)) = 10
		//控制折射程度
		_RefractAmount ("Refract Amount", Range(0.0, 1.0)) = 1.0
	}
	SubShader 
	{
	    //渲染队列设置成Transparent保证物体渲染时，其他不透明物体已经被渲染到屏幕上了
	    //渲染类型设置为不透明，使用着色器替换时，该物体可以在需要时被正确渲染。
	    //这通常发生在我们需要得到摄像机的深度和法线贴图时
		Tags { "Queue"="Transparent" "RenderType"="Opaque" }
		
		//获取屏幕图像存到名为_RefractionTex纹理中
		GrabPass { "_RefractionTex" }
		
		Pass 
		{		
			CGPROGRAM
			
			#pragma vertex vert
			#pragma fragment frag
			
			#include "UnityCG.cginc"
			
			sampler2D _MainTex;
			float4 _MainTex_ST;
			sampler2D _BumpMap;
			float4 _BumpMap_ST;
			samplerCUBE _Cubemap;
			float _Distortion;
			fixed _RefractAmount;
			sampler2D _RefractionTex;
			//纹素大小
			//一个266X512的纹理，它的纹素是(1/256,1/512)
			//我们需要在对屏幕图像采样坐标进行偏移时使用该变量
			float4 _RefractionTex_TexelSize;
			
			struct a2v 
			{
				float4 vertex : POSITION;
				float3 normal : NORMAL;
				float4 tangent : TANGENT; 
				float2 texcoord: TEXCOORD0;
			};
			
			struct v2f 
			{
				float4 pos : SV_POSITION;
				float4 scrPos : TEXCOORD0;
				float4 uv : TEXCOORD1;
				float4 TtoW0 : TEXCOORD2;  
			    float4 TtoW1 : TEXCOORD3;  
			    float4 TtoW2 : TEXCOORD4; 
			};
			
			v2f vert (a2v v) 
			{
				v2f o;
				o.pos = UnityObjectToClipPos(v.vertex);
				//得到被抓取的屏幕图像的采样坐标
				o.scrPos = ComputeGrabScreenPos(o.pos);
				//计算_MainTex和_BumpMap的采样坐标
				o.uv.xy = TRANSFORM_TEX(v.texcoord, _MainTex);
				o.uv.zw = TRANSFORM_TEX(v.texcoord, _BumpMap);
				//切线空间到世界空间变换矩阵，把每一行分别存在TtoW0,TtoW1,TtoW2中
				float3 worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;  
				//计算世界空间下顶点切线
				fixed3 worldNormal = UnityObjectToWorldNormal(v.normal);  
				//计算世界空间下顶点副切线
				fixed3 worldTangent = UnityObjectToWorldDir(v.tangent.xyz);  
				//计算世界空间下顶点法线
				fixed3 worldBinormal = cross(worldNormal, worldTangent) * v.tangent.w; 
				//w分量是世界空间下顶点位置（充分利用寄存器空间）
				o.TtoW0 = float4(worldTangent.x, worldBinormal.x, worldNormal.x, worldPos.x);  
				o.TtoW1 = float4(worldTangent.y, worldBinormal.y, worldNormal.y, worldPos.y);  
				o.TtoW2 = float4(worldTangent.z, worldBinormal.z, worldNormal.z, worldPos.z);  
				
				return o;
			}
			
			fixed4 frag (v2f i) : SV_Target 
			{		
			    //解译我们的顶点世界坐标
				float3 worldPos = float3(i.TtoW0.w, i.TtoW1.w, i.TtoW2.w);
				//得到该片元对应的视角方向
				fixed3 worldViewDir = normalize(UnityWorldSpaceViewDir(worldPos));
				
				//对法线纹理进行采样，得到切线空间下的法线方向
				fixed3 bump = UnpackNormal(tex2D(_BumpMap, i.uv.zw));	
				
				//对屏幕图像采样坐标进行偏移，模拟折射效果
				float2 offset = bump.xy * _Distortion * _RefractionTex_TexelSize.xy;

				i.scrPos.xy = offset * i.scrPos.z + i.scrPos.xy;
				//对抓取的屏幕图像_RefractionTex进行采样，得到模拟的折射颜色
				//进行透视除法，得到真正的屏幕坐标
				fixed3 refrCol = tex2D(_RefractionTex, i.scrPos.xy/i.scrPos.w).rgb;
				
				//法线方向从切线空间转世界空间
				bump = normalize(half3(dot(i.TtoW0.xyz, bump), dot(i.TtoW1.xyz, bump), dot(i.TtoW2.xyz, bump)));
				//视角方向相对于法线方向的反射方向
				fixed3 reflDir = reflect(-worldViewDir, bump);
				//对主纹理进行采样
				fixed4 texColor = tex2D(_MainTex, i.uv.xy);
				//对Cubemap进行采样，把结果和主纹理颜色相乘后得到反射颜色
				fixed3 reflCol = texCUBE(_Cubemap, reflDir).rgb * texColor.rgb;
				//反射颜色和折射颜色进行混合，作为最终输出颜色
				fixed3 finalColor = reflCol * (1 - _RefractAmount) + refrCol * _RefractAmount;
				
				return fixed4(finalColor, 1);
			}
			
			ENDCG
		}
	}
	
	FallBack "Diffuse"
}

```
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/10/QQ截图20191019164719.png)
GrabPass支持两种形式

- 直接使用GrabPass{}，然后在后续的Pass中直接使用_GrabTexture来访问屏幕图像。但是如果场景中多个物体都使用这个形式，会造成较大性能消耗。因为对于每一个使用它的物体，Unity都会为它单独进行一次昂贵的屏幕抓取操作。
- 使用GrabPass{"TextureName"}，可以在后续的Pass中使用TextureName来访问屏幕图像。Unity只会在每一帧时为第一个使用名为TextureName的纹理的物体进行一次抓取屏幕，同样可以在其他Pass中被访问。即所有使用该命令的物体都是同样的屏幕图像。

### 渲染纹理对比GrabPass
从效率上来说，渲染纹理效率往往要好于GrabPass，使用渲染纹理我们可以自定义渲染纹理的大小。而使用GrabPass获取到的屏幕分辨率和显示屏幕是一致的，这意味着在一些高分辨率设备上可能会造成严重的带宽影响。
在移动设备上，GrabPass虽然不会重新渲染场景，但它往往`需要CPU直接读取后备缓冲(back buffer)`，破坏了CPU和GPU并行性，这往往是比较耗时的。`一些移动设备不支持`。
Unity引入了`命令缓冲(Command Buffers)`来允许我们拓展Unity的渲染流水线。

## 程序纹理
程序纹理是指那些由计算机生成的图像，我们通常使用一些特定的算法来创建个性化团或非常真实的自然元素，例如木头，石子。
好处是我们可以通过各种参数来控制纹理的外观，而这些属性不仅仅是那些衍射属性，甚至可以是完全不同类型的图案属性。
### 在Unity中实现简单的程序纹理
```csharp
using UnityEngine;
using System.Collections;
using System.Collections.Generic;

[ExecuteInEditMode]
public class ProceduralTextureGeneration : MonoBehaviour 
{

	public Material material = null;

	#region Material properties
	[SerializeField, SetProperty("textureWidth")]
	private int m_textureWidth = 512;
	public int textureWidth 
	{
		get 
		{
			return m_textureWidth;
		}
		set 
		{
			m_textureWidth = value;
			_UpdateMaterial();
		}
	}

	[SerializeField, SetProperty("backgroundColor")]
	private Color m_backgroundColor = Color.white;
	public Color backgroundColor 
	{
		get 
		{
			return m_backgroundColor;
		}
		set 
		{
			m_backgroundColor = value;
			_UpdateMaterial();
		}
	}

	[SerializeField, SetProperty("circleColor")]
	private Color m_circleColor = Color.yellow;
	public Color circleColor 
	{
		get 
		{
			return m_circleColor;
		}
		set 
		{
			m_circleColor = value;
			_UpdateMaterial();
		}
	}

	[SerializeField, SetProperty("blurFactor")]
	private float m_blurFactor = 2.0f;
	public float blurFactor 
	{
		get 
		{
			return m_blurFactor;
		}
		set 
		{
			m_blurFactor = value;
			_UpdateMaterial();
		}
	}
	#endregion

	private Texture2D m_generatedTexture = null;

	// Use this for initialization
	void Start () 
	{
		if (material == null) 
		{
			Renderer renderer = gameObject.GetComponent<Renderer>();
			if (renderer == null) 
			{
				Debug.LogWarning("Cannot find a renderer.");
				return;
			}

			material = renderer.sharedMaterial;
		}

		_UpdateMaterial();
	}

	private void _UpdateMaterial() 
	{
		if (material != null) 
		{
			m_generatedTexture = _GenerateProceduralTexture();
			material.SetTexture("_MainTex", m_generatedTexture);
		}
	}

	/// <summary>
	/// 颜色混合，模糊边界
	/// </summary>
	/// <param name="color0"></param>
	/// <param name="color1"></param>
	/// <param name="mixFactor"></param>
	/// <returns></returns>
	private Color _MixColor(Color color0, Color color1, float mixFactor) 
	{
		Color mixColor = Color.white;
		mixColor.r = Mathf.Lerp(color0.r, color1.r, mixFactor);
		mixColor.g = Mathf.Lerp(color0.g, color1.g, mixFactor);
		mixColor.b = Mathf.Lerp(color0.b, color1.b, mixFactor);
		mixColor.a = Mathf.Lerp(color0.a, color1.a, mixFactor);
		return mixColor;
	}

	/// <summary>
	/// 生成纹理
	/// </summary>
	/// <returns></returns>
	private Texture2D _GenerateProceduralTexture() 
	{
		Texture2D proceduralTexture = new Texture2D(textureWidth, textureWidth);

		// 圆边距
		float circleInterval = textureWidth / 4.0f;
		// 圆半径
		float radius = textureWidth / 10.0f;
		// 模糊系数
		float edgeBlur = 1.0f / blurFactor;

		for (int w = 0; w < textureWidth; w++) 
		{
			for (int h = 0; h < textureWidth; h++) 
			{
				// 使用背景颜色进行初始化
				Color pixel = backgroundColor;
				
				for (int i = 0; i < 3; i++) 
				{
					for (int j = 0; j < 3; j++) 
					{
						// 计算当前绘制圆的圆心位置
						Vector2 circleCenter = new Vector2(circleInterval * (i + 1), circleInterval * (j + 1));

						// 计算当前像素与圆心的距离
						float dist = Vector2.Distance(new Vector2(w, h), circleCenter) - radius;

						// 模糊圆的边界
						Color color = _MixColor(circleColor, new Color(pixel.r, pixel.g, pixel.b, 0.0f), Mathf.SmoothStep(0f, 1.0f, dist * edgeBlur));

						// 与之前的颜色进行混合
						pixel = _MixColor(pixel, color, color.a);
					}
				}
				//写入纹理
				proceduralTexture.SetPixel(w, h, pixel);
			}
		}

		proceduralTexture.Apply();

		return proceduralTexture;
	}
}

```
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/10/QQ截图20191019172133.png)
### Unity的程序材质
使用Substance Sesigner制作。
