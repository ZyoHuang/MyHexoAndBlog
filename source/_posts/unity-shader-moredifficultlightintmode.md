---
title: Unity Shader入门精要学习笔记：更复杂的光照
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

# 更复杂的光照
## Unity的渲染路径
`Unity的渲染路径决定了光照是如何应用到Unity Shader中的。`
Unity支持多种类型的渲染路径，主要有三种：前向渲染路径(Forward Rendering Path)，延迟渲染路径(Deferred Rendering Path)，顶点照明渲染路径(Vertex Lit Rendering Path)（在Unity 5.0后被抛弃）。
如果当前显卡不支持所选择的渲染路径，Unity就会自动使用更低一级的渲染路径。
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/10/QQ截图20191010183353.png)
通俗来讲，指定渲染路径是我们和Unity的底层渲染引擎的一次重要沟通。
`如果不指定渲染路径，一些光照变量很可能不会被正确赋值。`
### 前向渲染路径
#### 前向渲染路径的原理
每进行一次完整的前向渲染，我们需要渲染该对象的渲染图元，并计算连个缓冲区的信息：颜色缓冲区和深度缓冲区。我们利用深度缓冲来决定一个片元是否可见，如果可见就更新颜色缓冲区的颜色值。
对于每个逐像素光源，我们都需要进行上面一次完整的渲染流程。如果一个物体在多个逐像素光源的影响区内，那么该物体就需要执行多个Pass，每个Pass计算一个逐像素光源的光照结果，然后在帧缓冲中把这些光照结果结合起来得到最终的颜色值。
假设场景中有N个物体，每个物体受M个光源的影响，那么渲染整个场景义工需要N*M个Pass。
#### Unity中的前向渲染
一个Pass不仅仅可以用来计算逐像素光照，他也可以用来计算逐顶点等其他光照。
在Unity中，前向渲染路径有3种处理光照（即照亮物体）的方式：`逐顶点处理`，`逐像素处理`，`球谐函数处理(Spherical Harmonics,SH)`。
决定一个光源使用哪种处理模式取决于它的类型和渲染模式。
光源类型是指该光源是平行光还是其他类型的光源
光源的渲染模式是指该光源是否是重要的。如果我们把一个光照的模式设置为Important，Unity会把它当成一个逐像素光源来处理。
可以在光源的`Light组件`中设置这些属性。
在前向渲染中，Unity会根据场景中各个光源的设置以及这些光源对物体的影响程度进行重要度排序，一定数目的光源会按逐像素方式处理，然后`最多有4个光源按逐顶点方式处理`。最后剩下的光源可以用SH方式处理.
Unity判断规则如下

- 场景中最亮的平行光总是按逐像素处理的。
- 渲染模式被设置成Not Important的光源，会按照逐顶点或者SH处理
- 渲染模式被设置成Important的光源，会按照逐像素处理
- 如果根据以上规则得到的逐像素光源数量小于Quality Setting中的逐像素光源数量（Pixel Light Count），会有更多光源以逐像素的方式渲染。

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/10/QQ截图20191010213420.png)

- 只有分别为Base Pass和Additional Pass使用图中两个预编译指令才会在相关Pass得到正确光照变量，例如光照衰减值。
- 对于前向渲染来说，一个Unity Shader通常会定义一个Base Pass（也可以定义多次，例如需要双面渲染）以及一个Additional Pass。一个Base Pass仅会执行一次。（定义多个Base Pass情况除外）。一个Additional Pass会根据影响该物体的其他逐像素光源的数目被多次调用，即每个逐像素光源会执行一次Additional Pass。

上图给出的是在Pass中光照计算的`通常做法`，完全可以按照自己的想法进行光照计算。
#### 内置的光照变量和函数
Unity会根据我们使用的渲染路径（即Pass标签中的LightMode的值）把不同的光照变量传递给Shader。
在Unity5中，对于前向渲染（即LightMode为ForwardBase或ForwardAdd），`前向渲染中可以访问的光照变量。（非完整版）`
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/10/QQ截图20191011194904.png)
`前向渲染中可以使用的内置光照函数。（非完整版）`
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/10/QQ截图20191011194904-gim.png)
### 顶点照明渲染路径
`顶点照明渲染路径对硬件配置要求最低，运算性能最高，相应的效果最差，它不支持那些逐像素才能得到的效果，例如阴影，法线映射，高精度的高光反射等`。
是前向渲染路径子集，能用顶点照明渲染路径的用前向渲染路径依旧可以。
顶点照明渲染路径只是使用了逐顶点的方式来计算光照。Unity也只会填充那些逐顶点相关的光源变量。我们也`不可以使用一些逐像素光照变量`。
### 延迟渲染路径
前向渲染的问题是：当场景中包含大量实时光源时，前向渲染性能会急速下降。因为如果一个物体被M个实时光照射，就要执行M个Pass，物体多起来，性能就下去了。但事实上很多计算是重复的。
延迟渲染除了使用深度缓冲和颜色缓冲外，还会使用G缓冲（G-Buffer），G缓冲区存储了我们所关心表面的其他信息，例如该表面法线，位置，用于光照计算的材质属性。
#### 延迟渲染的原理
主要包含两个Pass，在第一个Pass中，我们不进行任何光照计算，仅计算那些片元是可见的（深度测试），如果一个片元可见，就把它相关信息存储到G缓冲区中。在第二个Pass中，利用G缓冲区各个片元信息进行真正的光照计算。
`延迟渲染效率不依赖于场景复杂度，而是和我们屏幕空间大小有关，这是因为信息都存储在缓冲区中，而这些缓冲区可以理解成一张张2D图像，我们计算实际上就是这些图像空间中进行的。`
#### Unity中的延迟渲染
适合在光源数目很多，如果使用前向渲染会造成性能瓶颈情况下使用。延迟渲染路径中每个光源都可以按逐像素方式处理。
但是，延迟渲染也有缺点

- 不支持真正抗锯齿(anti-aliasing)功能
- 不能处理半透明物体
- 对显卡有一定要求

#### 可访问的内置变量
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/10/QQ截图20191011202910.png)
## Unity中的光源类型
Unity一共支持四种光源类型：平行光，点光源，聚光灯和面光源。
### 光源类型有什么影响
我们最常使用光源属性有`光源的位置`，`方向`（更具体的说是到某点的方向），`颜色`，`强度`以及`衰减`。
#### 平行光
他只有方向和颜色，他到场景中所有点方向都是一样的，没有具体的位置，也没有衰减的概念，光照强度不会随着距离改变而改变。
#### 点光源
有位置，方向，颜色，强度，光照衰减。
#### 聚光灯
有位置，方向，颜色，强度，光照衰减。
### 在前向渲染中处理不同光照类型
#### 实践
本部分内容建立在使用前向渲染基础上
```GLSL
// Upgrade NOTE: replaced '_LightMatrix0' with 'unity_WorldToLight'
// Upgrade NOTE: replaced '_Object2World' with 'unity_ObjectToWorld'
// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'

// Upgrade NOTE: replaced '_Object2World' with 'unity_ObjectToWorld'
// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'

Shader "Unity Shaders Book/Chapter9/ForwardRendering"
{
    Properties
    {
        _Diffuse("Diffuse",Color) = (1,1,1,1)
        _Specular("Specular",Color) = (1,1,1,1)
        _Gloss("Gloss",Range(8.0,256)) = 20
    }
    SubShader
    {
        Tags { "RenderType"="Opaque" }
        //如果场景中有多个平行光，Unity会把最亮的那个交给Base Pass逐像素处理
        //其余的交给Additional Pass
        Pass
        {
            Tags{"LightMode" = "ForwardBase"}
            CGPROGRAM
            //保证在Shader中使用的光照衰减等光照变量可以正确赋值
            #pragma multi_compile_fwdbase
            #pragma vertex vert
            #pragma fragment frag

            #include "Lighting.cginc"
            
            fixed4 _Diffuse;
            fixed4 _Specular;
            float _Gloss;

            struct a2v
            {
                float4 vertex : POSITION;
                float3 normal : NORMAL;
            };

            struct v2f
            {
                float4 pos : SV_POSITION;
                float3 worldNormal : TEXCOORD0;
                float3 worldPos : TEXCOORD1;
            };

            v2f vert (a2v v)
            {
                v2f o;
                //把顶点位置从模型空间转换到裁剪空间中
                o.pos = UnityObjectToClipPos(v.vertex);
                //计算出世界空间法线方向
                o.worldNormal = UnityObjectToWorldNormal(v.normal);
                //计算出世界空间位置
                o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;
                return o;
            }

            fixed4 frag (v2f i) : SV_Target
            {
                fixed3 worldNormal = normalize(i.worldNormal);
                fixed3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz);
                //计算环境光
                fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;
                
                fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * max(0,dot(worldNormal,worldLightDir));
                
                fixed3 viewDir = normalize(_WorldSpaceLightPos0.xyz - i.worldPos.xyz);
                fixed3 halfDir = normalize(worldLightDir+viewDir);
                fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(max(0,dot(worldNormal,halfDir)),_Gloss);
                //平行光衰减值总是1.0
                fixed atten = 1.0;
                return fixed4(ambient+(diffuse+specular)*atten,1.0);
            }
            ENDCG
        }
        
        //场景中其他逐像素光源
        Pass 
		{
			Tags { "LightMode"="ForwardAdd" }
			//希望与帧缓存中其他光照结果进行叠加混合
			Blend One One
		
			CGPROGRAM
			
			#pragma multi_compile_fwdadd
			
			#pragma vertex vert
			#pragma fragment frag
			
			#include "Lighting.cginc"
			#include "AutoLight.cginc"
			
			fixed4 _Diffuse;
			fixed4 _Specular;
			float _Gloss;
			
			struct a2v 
			{
				float4 vertex : POSITION;
				float3 normal : NORMAL;
			};
			
			struct v2f 
			{
				float4 pos : SV_POSITION;
				float3 worldNormal : TEXCOORD0;
				float3 worldPos : TEXCOORD1;
			};
			
			v2f vert(a2v v) 
			{
				v2f o;
				o.pos = UnityObjectToClipPos(v.vertex);
				
				o.worldNormal = UnityObjectToWorldNormal(v.normal);
				
				o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;
				
				return o;
			}
			
			fixed4 frag(v2f i) : SV_Target 
			{
				fixed3 worldNormal = normalize(i.worldNormal);
				//不同光源方向
				#ifdef USING_DIRECTIONAL_LIGHT
					fixed3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz);
				#else
					fixed3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz - i.worldPos.xyz);
				#endif
				
				fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * max(0, dot(worldNormal, worldLightDir));
				
				fixed3 viewDir = normalize(_WorldSpaceCameraPos.xyz - i.worldPos.xyz);
				fixed3 halfDir = normalize(worldLightDir + viewDir);
				fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(max(0, dot(worldNormal, halfDir)), _Gloss);
				
				//不同光源衰减度
				#ifdef USING_DIRECTIONAL_LIGHT
					fixed atten = 1.0;
				#else
					#if defined (POINT)
				        float3 lightCoord = mul(unity_WorldToLight, float4(i.worldPos, 1)).xyz;
				        fixed atten = tex2D(_LightTexture0, dot(lightCoord, lightCoord).rr).UNITY_ATTEN_CHANNEL;
				    #elif defined (SPOT)
				        float4 lightCoord = mul(unity_WorldToLight, float4(i.worldPos, 1));
				        fixed atten = (lightCoord.z > 0) * tex2D(_LightTexture0, lightCoord.xy / lightCoord.w + 0.5).w * tex2D(_LightTextureB0, dot(lightCoord, lightCoord).rr).UNITY_ATTEN_CHANNEL;
				    #else
				        fixed atten = 1.0;
				    #endif
				#endif

				return fixed4((diffuse + specular) * atten, 1.0);
			}
			
			ENDCG
		}
    }
	FallBack "Specular"
}

```
## Unity的光照衰减
在Unity中使用一张纹理作为查找表来在片元着色器中计算逐像素光照的衰减。
好处是不用靠复杂的数学公式，可以提升一定的性能，并且效果看的过去。
缺点是

- 需要预处理得到采样纹理，而且纹理的大小也会影响衰减的精度。
- 不直观，也不方便，一旦把数据存储到查找表中，我们就没办法使用其他数学公式计算衰减。

### 用于光照衰减的纹理
Unity在内部使用了一张名为_LightTexture0的纹理来计算光照衰减。
_LightTexture0对角线上的纹理颜色值表明在光源空间中不同位置的点的衰减值。
为了对_LightTexture0纹理采样得到给定点到该光源的衰减值，需要先得到该点在光源空间中的位置，这是通过_LightMatrix0变换矩阵（把顶点从世界空间变换到光源空间，在后续版本中被unity_WorldToLight替代）得到的。
## Unity的阴影
### 阴影是如何实现的
在实时渲染中，最常使用一种名为Shadow Map的技术，他会首先把摄像机的位置放在与光源重合的位置上，那么场景中的该光源的阴影区域就是那些摄像机看不到的地方。
如果场景中最重要的平行光开启了阴影，Unity就会为该光源计算它的阴影映射纹理（Shadowmap），`本质上是一张深度图`，他记录了从该光源位置出发，能看到的场景中距离它最近的表面位置（深度信息）。
如果使用正常的Pass（Base Pass和Additional Pass）会因为涉及过多不需要的光照计算而造成性能浪费。
Unity选择使用一个额外的Pass来专门更新光源的阴影映射纹理，这个Pass就是LightModel标签被设置为`ShadowCaster`的Pass。
Unity现在使用了新的阴影采样技术，即`屏幕空间的阴影映射技术（Screenspace Shadow Map）`。原本是延迟渲染中产生阴影的方法，所以并不是所有平台都支持这个阴影采样技术（需要显卡支持MRT）。

- 如果一个物体要接收来自其他物体的阴影，就必须在Shader中对阴影映射纹理（包括屏幕空间的阴影图）进行采样，把采样结果和最后的光照结果相乘来产生阴影效果。
- 如果一个物体要向其他物体投射阴影，就必须把该物体加入到光源的阴影映射纹理的计算中（通过执行LightMode为ShadowCaster的Pass实现）。如果使用了屏幕空间的投影映射技术，Unity还会使用这个Pass产生一张摄像机的深度纹理。
### 不透明物体的阴影
#### 让物体投射阴影
默认会通过Fallback内置的VertexLit（LightMode为ShadowCaster）来实现阴影。
默认剔除背面阴影计算，如果需要，可以在物体的Mesh Renderer的Cast Shadow设置为two side。
#### 让物体接收阴影
`SHADOW_COORDS，TRANSFER_SHADOW，SHADOW_ATTENUATION阴影三剑客（貌似叫暗影三剑客更帅一点）`
```GLSL
// Upgrade NOTE: replaced '_LightMatrix0' with 'unity_WorldToLight'
// Upgrade NOTE: replaced '_Object2World' with 'unity_ObjectToWorld'
// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'

// Upgrade NOTE: replaced '_Object2World' with 'unity_ObjectToWorld'
// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'

Shader "Unity Shaders Book/Chapter9/Shadow"
{
    Properties
    {
        _Diffuse("Diffuse",Color) = (1,1,1,1)
        _Specular("Specular",Color) = (1,1,1,1)
        _Gloss("Gloss",Range(8.0,256)) = 20
    }
    SubShader
    {
        Tags { "RenderType"="Opaque" }
        //如果场景中有多个平行光，Unity会把最亮的那个交给Base Pass逐像素处理
        //其余的交给Additional Pass
        Pass
        {
            Tags{"LightMode" = "ForwardBase"}
            CGPROGRAM
            //保证在Shader中使用的光照衰减等光照变量可以正确赋值
            #pragma multi_compile_fwdbase
            #pragma vertex vert
            #pragma fragment frag
            
            //提供用于计算阴影的宏
            #include "AutoLight.cginc"
            #include "Lighting.cginc"
            
            fixed4 _Diffuse;
            fixed4 _Specular;
            float _Gloss;

            struct a2v
            {
                float4 vertex : POSITION;
                float3 normal : NORMAL;
            };

            struct v2f
            {
                float4 pos : SV_POSITION;
                float3 worldNormal : TEXCOORD0;
                float3 worldPos : TEXCOORD1;
                //声明一个用于对阴影采样的坐标。
                //该参数需要下一个可用插值寄存器的索引值，在这里是2
                SHADOW_COORDS(2)
            };

            v2f vert (a2v v)
            {
                v2f o;
                //把顶点位置从模型空间转换到裁剪空间中
                o.pos = UnityObjectToClipPos(v.vertex);
                //计算出世界空间法线方向
                o.worldNormal = UnityObjectToWorldNormal(v.normal);
                //计算出世界空间位置
                o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;
                //在顶点着色器计算上一步中声明的阴影纹理坐标
                TRANSFER_SHADOW(o);
                return o;
            }

            fixed4 frag (v2f i) : SV_Target
            {
                fixed3 worldNormal = normalize(i.worldNormal);
                fixed3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz);
                //计算环境光
                fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;
                
                fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * max(0,dot(worldNormal,worldLightDir));
                
                fixed3 viewDir = normalize(_WorldSpaceLightPos0.xyz - i.worldPos.xyz);
                fixed3 halfDir = normalize(worldLightDir+viewDir);
                fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(max(0,dot(worldNormal,halfDir)),_Gloss);
                //平行光衰减值总是1.0
                fixed atten = 1.0;
                //计算阴影值
                fixed shadow = SHADOW_ATTENUATION(i);
                return fixed4(ambient+(diffuse+specular)*atten*shadow,1.0);
            }
            ENDCG
        }
        
        //场景中其他逐像素光源
        Pass 
		{
			Tags { "LightMode"="ForwardAdd" }
			//希望与帧缓存中其他光照结果进行叠加混合
			Blend One One
		
			CGPROGRAM
			
			#pragma multi_compile_fwdadd
			
			#pragma vertex vert
			#pragma fragment frag
			
			#include "Lighting.cginc"
			#include "AutoLight.cginc"
			
			fixed4 _Diffuse;
			fixed4 _Specular;
			float _Gloss;
			
			struct a2v 
			{
				float4 vertex : POSITION;
				float3 normal : NORMAL;
			};
			
			struct v2f 
			{
				float4 pos : SV_POSITION;
				float3 worldNormal : TEXCOORD0;
				float3 worldPos : TEXCOORD1;
			};
			
			v2f vert(a2v v) 
			{
				v2f o;
				o.pos = UnityObjectToClipPos(v.vertex);
				
				o.worldNormal = UnityObjectToWorldNormal(v.normal);
				
				o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;
				
				return o;
			}
			
			fixed4 frag(v2f i) : SV_Target 
			{
				fixed3 worldNormal = normalize(i.worldNormal);
				//不同光源方向
				#ifdef USING_DIRECTIONAL_LIGHT
					fixed3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz);
				#else
					fixed3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz - i.worldPos.xyz);
				#endif
				
				fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * max(0, dot(worldNormal, worldLightDir));
				
				fixed3 viewDir = normalize(_WorldSpaceCameraPos.xyz - i.worldPos.xyz);
				fixed3 halfDir = normalize(worldLightDir + viewDir);
				fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(max(0, dot(worldNormal, halfDir)), _Gloss);
				
				//不同光源衰减度
				#ifdef USING_DIRECTIONAL_LIGHT
					fixed atten = 1.0;
				#else
					#if defined (POINT)
				        float3 lightCoord = mul(unity_WorldToLight, float4(i.worldPos, 1)).xyz;
				        fixed atten = tex2D(_LightTexture0, dot(lightCoord, lightCoord).rr).UNITY_ATTEN_CHANNEL;
				    #elif defined (SPOT)
				        float4 lightCoord = mul(unity_WorldToLight, float4(i.worldPos, 1));
				        fixed atten = (lightCoord.z > 0) * tex2D(_LightTexture0, lightCoord.xy / lightCoord.w + 0.5).w * tex2D(_LightTextureB0, dot(lightCoord, lightCoord).rr).UNITY_ATTEN_CHANNEL;
				    #else
				        fixed atten = 1.0;
				    #endif
				#endif

				return fixed4((diffuse + specular) * atten, 1.0);
			}
			
			ENDCG
		}
    }
	FallBack "Specular"
}

```
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/10/QQ截图20191013215344.png)
### 统一管理光照衰减和阴影
```GLSL
// Upgrade NOTE: replaced '_Object2World' with 'unity_ObjectToWorld'
// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'

Shader "Unity Shaders Book/Chapter 9/Attenuation And Shadow Use Build-in Functions" 
{
	Properties 
	{
		_Diffuse ("Diffuse", Color) = (1, 1, 1, 1)
		_Specular ("Specular", Color) = (1, 1, 1, 1)
		_Gloss ("Gloss", Range(8.0, 256)) = 20
	}
	SubShader 
	{
		Tags { "RenderType"="Opaque" }
		
		Pass 
		{
			Tags { "LightMode"="ForwardBase" }
		
			CGPROGRAM
			
			#pragma multi_compile_fwdbase	
			
			#pragma vertex vert
			#pragma fragment frag
			
			#include "Lighting.cginc"
			#include "AutoLight.cginc"
			
			fixed4 _Diffuse;
			fixed4 _Specular;
			float _Gloss;
			
			struct a2v 
			{
				float4 vertex : POSITION;
				float3 normal : NORMAL;
			};
			
			struct v2f 
			{
				float4 pos : SV_POSITION;
				float3 worldNormal : TEXCOORD0;
				float3 worldPos : TEXCOORD1;
				//声明阴影坐标
				SHADOW_COORDS(2)
			};
			
			v2f vert(a2v v) 
			{
			 	v2f o;
			 	o.pos = UnityObjectToClipPos(v.vertex);
			 	
			 	o.worldNormal = UnityObjectToWorldNormal(v.normal);
			 	
			 	o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;
			 	
			 	//计算并向片元着色器传递阴影坐标
			 	TRANSFER_SHADOW(o);
			 	
			 	return o;
			}
			
			fixed4 frag(v2f i) : SV_Target 
			{
				fixed3 worldNormal = normalize(i.worldNormal);
				fixed3 worldLightDir = normalize(UnityWorldSpaceLightDir(i.worldPos));
				
				fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;
				
			 	fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * max(0, dot(worldNormal, worldLightDir));

			 	fixed3 viewDir = normalize(UnityWorldSpaceViewDir(i.worldPos));
			 	fixed3 halfDir = normalize(worldLightDir + viewDir);
			 	fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(max(0, dot(worldNormal, halfDir)), _Gloss);

				//使用内置宏计算光照衰减和阴影
				UNITY_LIGHT_ATTENUATION(atten, i, i.worldPos);
				
				return fixed4(ambient + (diffuse + specular) * atten, 1.0);
			}
			
			ENDCG
		}
	
		Pass 
		{
			Tags { "LightMode"="ForwardAdd" }
			
			Blend One One
		
			CGPROGRAM
			
			#pragma multi_compile_fwdadd
			#pragma vertex vert
			#pragma fragment frag
			
			#include "Lighting.cginc"
			#include "AutoLight.cginc"
			
			fixed4 _Diffuse;
			fixed4 _Specular;
			float _Gloss;
			
			struct a2v 
			{
				float4 vertex : POSITION;
				float3 normal : NORMAL;
			};
			
			struct v2f 
			{
				float4 pos : SV_POSITION;
				float3 worldNormal : TEXCOORD0;
				float3 worldPos : TEXCOORD1;
				SHADOW_COORDS(2)
			};
			
			v2f vert(a2v v) 
			{
			 	v2f o;
			 	o.pos = UnityObjectToClipPos(v.vertex);
			 	
			 	o.worldNormal = UnityObjectToWorldNormal(v.normal);
			 	
			 	o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;

			 	TRANSFER_SHADOW(o);
			 	
			 	return o;
			}
			
			fixed4 frag(v2f i) : SV_Target 
			{
				fixed3 worldNormal = normalize(i.worldNormal);
				fixed3 worldLightDir = normalize(UnityWorldSpaceLightDir(i.worldPos));
				
			 	fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * max(0, dot(worldNormal, worldLightDir));

			 	fixed3 viewDir = normalize(UnityWorldSpaceViewDir(i.worldPos));
			 	fixed3 halfDir = normalize(worldLightDir + viewDir);
			 	fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(max(0, dot(worldNormal, halfDir)), _Gloss);

				UNITY_LIGHT_ATTENUATION(atten, i, i.worldPos);
			 	
				return fixed4((diffuse + specular) * atten, 1.0);
			}
			
			ENDCG
		}
	}
	FallBack "Specular"
}
```
### 透明度物体的阴影
透明物体实现通常会使用透明度测试或者透明度混合，所以要小心设置他们的Fallback。

#### 透明度测试的物体阴影

计算提供相关阴影数据，额外提供名为_Cutoff属性，然后把Fallback设置为`Transparent/Cutout/VertexLit`
#### 透明度混合的物体阴影
把Fallback设为VertexLit来强制为半透明物体生成阴影（效果还说得过去）。
