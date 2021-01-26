---
title: Unity Shader入门精要学习笔记：使用噪声
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

# 消融效果
常见于游戏中角色死亡，地图烧毁等效果，这些效果中，消融往往从不同区域开始，并向看似随机的方向扩张，最后整个物体消失不见。
下面是来自《英魂之刃口袋版》的一个击杀特效：
[![来自《英魂之刃口袋版》的一个击杀特效](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/11/10.gif)](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/11/10.gif)
原理非常简单，就是`噪声纹理+透明度测试`。我们使用对噪声纹理采样的结果和某个控制消融程度的阈值比较，如果小于阈值，就是用clip函数把它对应的像素裁剪掉，也就是被“烧毁的部分”，`镂空区域边缘的烧焦效果则是将两种颜色混合，再用pow函数处理后，与原纹理颜色混合后的结果`。

```csharp
using UnityEngine;
using System.Collections;

/// <summary>
/// 控制消融动画的辅助脚本
/// </summary>
public class BurnHelper : MonoBehaviour
{
    public Material material;

    [Range(0.01f, 1.0f)] public float burnSpeed = 0.3f;

    private float burnAmount = 0.0f;

    // Use this for initialization
    void Start()
    {
        if (material == null)
        {
            Renderer renderer = gameObject.GetComponentInChildren<Renderer>();
            if (renderer != null)
            {
                material = renderer.material;
            }
        }

        if (material == null)
        {
            this.enabled = false;
        }
        else
        {
            material.SetFloat("_BurnAmount", 0.0f);
        }
    }

    // Update is called once per frame
    void Update()
    {
        burnAmount = Mathf.Repeat(Time.time * burnSpeed, 1.0f);
        material.SetFloat("_BurnAmount", burnAmount);
    }
}
```
```GLSL
// Upgrade NOTE: replaced '_Object2World' with 'unity_ObjectToWorld'
// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'

Shader "Unity Shaders Book/Chapter 15/Dissolve" 
{
	Properties 
	{
	    //控制消融程度
		_BurnAmount ("Burn Amount", Range(0.0, 1.0)) = 0.0
		//控制模拟烧焦效果时的线宽
		_LineWidth("Burn Line Width", Range(0.0, 0.2)) = 0.1
		//漫反射纹理
		_MainTex ("Base (RGB)", 2D) = "white" {}
		//法线纹理
		_BumpMap ("Normal Map", 2D) = "bump" {}
		//火焰边缘的两种颜色
		_BurnFirstColor("Burn First Color", Color) = (1, 0, 0, 1)
		_BurnSecondColor("Burn Second Color", Color) = (1, 0, 0, 1)
		//噪声纹理
		_BurnMap("Burn Map", 2D) = "white"{}
	}
	SubShader 
	{
		Tags { "RenderType"="Opaque" "Queue"="Geometry"}
		
		Pass 
		{
			Tags { "LightMode"="ForwardBase" }
            //只渲染正面会出现错误的结果
			Cull Off
			
			CGPROGRAM
			
			#include "Lighting.cginc"
			#include "AutoLight.cginc"
			
			#pragma multi_compile_fwdbase
			
			#pragma vertex vert
			#pragma fragment frag
			
			fixed _BurnAmount;
			fixed _LineWidth;
			sampler2D _MainTex;
			sampler2D _BumpMap;
			fixed4 _BurnFirstColor;
			fixed4 _BurnSecondColor;
			sampler2D _BurnMap;
			
			float4 _MainTex_ST;
			float4 _BumpMap_ST;
			float4 _BurnMap_ST;
			
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
				float2 uvMainTex : TEXCOORD0;
				float2 uvBumpMap : TEXCOORD1;
				float2 uvBurnMap : TEXCOORD2;
				float3 lightDir : TEXCOORD3;
				float3 worldPos : TEXCOORD4;
				SHADOW_COORDS(5)
			};
			
			v2f vert(a2v v) 
			{
				v2f o;
				o.pos = UnityObjectToClipPos(v.vertex);
				//计算三张纹理对应纹理坐标，再把光源方向从模型空间变换到了切线空间
				o.uvMainTex = TRANSFORM_TEX(v.texcoord, _MainTex);
				o.uvBumpMap = TRANSFORM_TEX(v.texcoord, _BumpMap);
				o.uvBurnMap = TRANSFORM_TEX(v.texcoord, _BurnMap);
				
				TANGENT_SPACE_ROTATION;
  				o.lightDir = mul(rotation, ObjSpaceLightDir(v.vertex)).xyz;
  				
  				o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;
  				//为了得到阴影信息，计算了世界空间下的顶点位置和阴影纹理的采样坐标
  				TRANSFER_SHADOW(o);
				
				return o;
			}
			
			fixed4 frag(v2f i) : SV_Target 
			{
			    //对噪声纹理采样
				fixed3 burn = tex2D(_BurnMap, i.uvBurnMap).rgb;
				//如果结果小于零就舍弃这个像素
				clip(burn.r - _BurnAmount);
				
				float3 tangentLightDir = normalize(i.lightDir);
				fixed3 tangentNormal = UnpackNormal(tex2D(_BumpMap, i.uvBumpMap));
				//得到材质的反射率
				fixed3 albedo = tex2D(_MainTex, i.uvMainTex).rgb;
				//计算环境光照
				fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz * albedo;
				//得到漫反射光照
				fixed3 diffuse = _LightColor0.rgb * albedo * max(0, dot(tangentNormal, tangentLightDir));
                //平滑插值，计算混合系数
                //t为1说明该像素位于消融的边界处
                //t为0说明该像素为正常的模型颜色
				fixed t = 1 - smoothstep(0.0, _LineWidth, burn.r - _BurnAmount);
				//计算烧焦颜色
				fixed3 burnColor = lerp(_BurnFirstColor, _BurnSecondColor, t);
				//使效果更加逼真（颜色加深？）
				burnColor = pow(burnColor, 5);
				
				UNITY_LIGHT_ATTENUATION(atten, i, i.worldPos);
				fixed3 finalColor = lerp(ambient + diffuse * atten, burnColor, t * step(0.0001, _BurnAmount));
				
				return fixed4(finalColor, 1);
			}
			
			ENDCG
		}
		
		// 为了让物体的阴影也能配合透明度测试产生正确的效果，需要自定义一个投射阴影Pass
		// 阴影投射重点是需要按正常Pass的处理来剔除片元或进行顶点动画
		// 以便阴影可以和物体正常渲染的结果相匹配
		Pass 
		{
			Tags { "LightMode" = "ShadowCaster" }
			
			CGPROGRAM
			
			#pragma vertex vert
			#pragma fragment frag
			
			#pragma multi_compile_shadowcaster
			
			#include "UnityCG.cginc"
			
			fixed _BurnAmount;
			sampler2D _BurnMap;
			float4 _BurnMap_ST;
			
			struct v2f 
			{
			    //定义阴影投射所需要定义的变量
				V2F_SHADOW_CASTER;
				float2 uvBurnMap : TEXCOORD1;
			};
			
			v2f vert(appdata_base v) 
			{
				v2f o;
				//填充V2F_SHADOW_CASTER中的变量
				TRANSFER_SHADOW_CASTER_NORMALOFFSET(o)
				//计算噪声纹理的采样坐标
				o.uvBurnMap = TRANSFORM_TEX(v.texcoord, _BurnMap);
				
				return o;
			}
			
			fixed4 frag(v2f i) : SV_Target 
			{
				fixed3 burn = tex2D(_BurnMap, i.uvBurnMap).rgb;
				//使用噪声纹理的采样结果剔除片元
				clip(burn.r - _BurnAmount);
				//完成阴影投射，把结果输出到深度图和阴影映射纹理中
				SHADOW_CASTER_FRAGMENT(i)
			}
			ENDCG
		}
	}
	FallBack "Diffuse"
}

```

[![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/11/9.gif)](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/11/9.gif)

# 水波效果
在模拟实时水面的过程中，往往也会使用噪声纹理，此时，`噪声贴图通常会用作一个高度图，以不断修改睡眠的法线方向，为了模拟水不断流动的效果，会使用和时间相关的变量来对噪声纹理进行采样，当得到法线信息后，再进行正常的反射+折射计算，得到最后的水面波动效果`。
[![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/11/QQ截图20191111212214.png)](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/11/QQ截图20191111212214.png)
为了模拟折射效果，我们使用GrabPass来获取当前屏幕的渲染纹理，并使用`切线空间下的法线方向`对`像素的屏幕坐标`进行偏移，再使用该坐标对`渲染纹理进行屏幕采样`，从而模拟出近似的折射效果。水波纹的法线纹理是由一张噪声纹理生成而得，并且会随着时间变化不断平移，模拟波光粼粼的效果。`使用了菲涅尔系数来动态决定混合系数`。
$$fresnel\;=\;pow(1\;-\;max(0,\;v\cdot n),4)$$
其中v和n分别对应了视角方向和法线方向。他们夹角越小，fresnel值越小，反射越弱，折射越强。`菲涅尔系数还经常会用于边缘光照的计算中。`

```GLSL
// Upgrade NOTE: replaced '_Object2World' with 'unity_ObjectToWorld'
// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'

Shader "Unity Shaders Book/Chapter 15/Water Wave" 
{
	Properties 
	{
	    //水面颜色
		_Color ("Main Color", Color) = (0, 0.15, 0.115, 1)
		//水面波纹材质纹理
		_MainTex ("Base (RGB)", 2D) = "white" {}
		//噪声纹理生成的法线纹理
		_WaveMap ("Wave Map", 2D) = "bump" {}
		//用于模拟反射的立方体纹理
		_Cubemap ("Environment Cubemap", Cube) = "_Skybox" {}
		_WaveXSpeed ("Wave Horizontal Speed", Range(-0.1, 0.1)) = 0.01
		_WaveYSpeed ("Wave Vertical Speed", Range(-0.1, 0.1)) = 0.01
		//控制模拟折射时图像扭曲程度
		_Distortion ("Distortion", Range(0, 100)) = 10
	}
	
	SubShader 
	{
	    //Transparent确保该物体渲染时，其他所有不透明物体都已经被渲染到屏幕上了
	    //Opaque使用着色器替换时物体可以在需要时被正确渲染
	    //这通常发生在我们需要的到摄像机的深度和法线纹理时
		Tags { "Queue"="Transparent" "RenderType"="Opaque" }
		
		Cull Off
		
		//决定抓取到的屏幕图像将会被存入哪个纹理中
		GrabPass { "_RefractionTex" }
		
		Pass 
		{
			Tags { "LightMode"="ForwardBase" }
			
			CGPROGRAM
			
			#include "UnityCG.cginc"
			#include "Lighting.cginc"
			
			#pragma multi_compile_fwdbase
			
			#pragma vertex vert
			#pragma fragment frag
			
			fixed4 _Color;
			sampler2D _MainTex;
			float4 _MainTex_ST;
			sampler2D _WaveMap;
			float4 _WaveMap_ST;
			samplerCUBE _Cubemap;
			fixed _WaveXSpeed;
			fixed _WaveYSpeed;
			float _Distortion;	
			sampler2D _RefractionTex;
			//纹素大小
			//一个256*512的纹理，纹素大小就是1/256*1/512
			//对屏幕图像的采样坐标进行偏移时使用该变量
			float4 _RefractionTex_TexelSize;
			
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
				float4 scrPos : TEXCOORD0;
				float4 uv : TEXCOORD1;
				float4 TtoW0 : TEXCOORD2;  
				float4 TtoW1 : TEXCOORD3;  
				float4 TtoW2 : TEXCOORD4; 
			};
			
			v2f vert(a2v v) 
			{
				v2f o;
				o.pos = UnityObjectToClipPos(v.vertex);
				//得到对应被抓取屏幕图像的采样坐标
				o.scrPos = ComputeGrabScreenPos(o.pos);
				//计算采样坐标
				o.uv.xy = TRANSFORM_TEX(v.texcoord, _MainTex);
				o.uv.zw = TRANSFORM_TEX(v.texcoord, _WaveMap);
				
				float3 worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;  
				//因为在片元着色器中需要把法线方向从切线空间变换到世界空间，以便对Cubemap采样
				//所以这里要求得切线空间到世界空间的变换矩阵
				fixed3 worldNormal = UnityObjectToWorldNormal(v.normal);  
				fixed3 worldTangent = UnityObjectToWorldDir(v.tangent.xyz);  
				fixed3 worldBinormal = cross(worldNormal, worldTangent) * v.tangent.w; 
				
				o.TtoW0 = float4(worldTangent.x, worldBinormal.x, worldNormal.x, worldPos.x);  
				o.TtoW1 = float4(worldTangent.y, worldBinormal.y, worldNormal.y, worldPos.y);  
				o.TtoW2 = float4(worldTangent.z, worldBinormal.z, worldNormal.z, worldPos.z);  
				
				return o;
			}
			
			fixed4 frag(v2f i) : SV_Target 
			{
				float3 worldPos = float3(i.TtoW0.w, i.TtoW1.w, i.TtoW2.w);
				//得到视角方向
				fixed3 viewDir = normalize(UnityWorldSpaceViewDir(worldPos));
				//计算法线纹理当前偏移量
				float2 speed = _Time.y * float2(_WaveXSpeed, _WaveYSpeed);
				
				// 两次采样，模拟两层交叉水面波动效果
				fixed3 bump1 = UnpackNormal(tex2D(_WaveMap, i.uv.zw + speed)).rgb;
				fixed3 bump2 = UnpackNormal(tex2D(_WaveMap, i.uv.zw - speed)).rgb;
				//得到切线空间下的法线方向
				fixed3 bump = normalize(bump1 + bump2);
				
				//计算偏移量
				float2 offset = bump.xy * _Distortion * _RefractionTex_TexelSize.xy;
				//与z相乘是为了模拟深度越大，折射程度越大的效果
				i.scrPos.xy = offset * i.scrPos.z + i.scrPos.xy;
				//scrPos透视除法
				fixed3 refrCol = tex2D( _RefractionTex, i.scrPos.xy/i.scrPos.w).rgb;
				
				//把法线方向从切线空间变换到世界空间下
				bump = normalize(half3(dot(i.TtoW0.xyz, bump), dot(i.TtoW1.xyz, bump), dot(i.TtoW2.xyz, bump)));
				
				fixed4 texColor = tex2D(_MainTex, i.uv.xy + speed);
				//得到视角方向相对于法线方向的反射方向
				fixed3 reflDir = reflect(-viewDir, bump);
				//使用反射方向对Cubemap进行采样
				fixed3 reflCol = texCUBE(_Cubemap, reflDir).rgb * texColor.rgb * _Color.rgb;
				//就三菲涅尔系数
				fixed fresnel = pow(1 - saturate(dot(viewDir, bump)), 4);
				//混合折射和反射颜色
				fixed3 finalColor = reflCol * fresnel + refrCol * (1 - fresnel);
				
				return fixed4(finalColor, 1);
			}
			
			ENDCG
		}
	}
	// Do not cast shadow
	FallBack Off
}

```

[![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/11/10-syl.gif)](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/11/10-syl.gif)
# 再谈全局雾效
```csharp
using UnityEngine;
using System.Collections;

public class FogWithNoise : PostEffectsBase
{
    public Shader fogShader;
    private Material fogMaterial = null;

    public Material material
    {
        get
        {
            fogMaterial = CheckShaderAndCreateMaterial(fogShader, fogMaterial);
            return fogMaterial;
        }
    }

    private Camera myCamera;

    public Camera camera
    {
        get
        {
            if (myCamera == null)
            {
                myCamera = GetComponent<Camera>();
            }

            return myCamera;
        }
    }

    private Transform myCameraTransform;

    public Transform cameraTransform
    {
        get
        {
            if (myCameraTransform == null)
            {
                myCameraTransform = camera.transform;
            }

            return myCameraTransform;
        }
    }

    /// <summary>
    /// 雾浓度
    /// </summary>
    [Range(0.1f, 3.0f)] public float fogDensity = 1.0f;

    /// <summary>
    /// 雾颜色
    /// </summary>
    public Color fogColor = Color.white;

    public float fogStart = 0.0f;
    public float fogEnd = 2.0f;

    /// <summary>
    /// 噪声纹理
    /// </summary>
    public Texture noiseTexture;
    
    /// <summary>
    /// X方向的移动速度
    /// </summary>
    [Range(-0.5f, 0.5f)] public float fogXSpeed = 0.1f;

    /// <summary>
    /// Y方向上的移动速度
    /// </summary>
    [Range(-0.5f, 0.5f)] public float fogYSpeed = 0.1f;

    /// <summary>
    /// 噪声程度
    /// </summary>
    [Range(0.0f, 3.0f)] public float noiseAmount = 1.0f;

    void OnEnable()
    {
        GetComponent<Camera>().depthTextureMode |= DepthTextureMode.Depth;
    }

    void OnRenderImage(RenderTexture src, RenderTexture dest)
    {
        if (material != null)
        {
            Matrix4x4 frustumCorners = Matrix4x4.identity;

            float fov = camera.fieldOfView;
            float near = camera.nearClipPlane;
            float aspect = camera.aspect;

            float halfHeight = near * Mathf.Tan(fov * 0.5f * Mathf.Deg2Rad);
            Vector3 toRight = cameraTransform.right * halfHeight * aspect;
            Vector3 toTop = cameraTransform.up * halfHeight;

            Vector3 topLeft = cameraTransform.forward * near + toTop - toRight;
            float scale = topLeft.magnitude / near;

            topLeft.Normalize();
            topLeft *= scale;

            Vector3 topRight = cameraTransform.forward * near + toRight + toTop;
            topRight.Normalize();
            topRight *= scale;

            Vector3 bottomLeft = cameraTransform.forward * near - toTop - toRight;
            bottomLeft.Normalize();
            bottomLeft *= scale;

            Vector3 bottomRight = cameraTransform.forward * near + toRight - toTop;
            bottomRight.Normalize();
            bottomRight *= scale;

            frustumCorners.SetRow(0, bottomLeft);
            frustumCorners.SetRow(1, bottomRight);
            frustumCorners.SetRow(2, topRight);
            frustumCorners.SetRow(3, topLeft);

            material.SetMatrix("_FrustumCornersRay", frustumCorners);

            material.SetFloat("_FogDensity", fogDensity);
            material.SetColor("_FogColor", fogColor);
            material.SetFloat("_FogStart", fogStart);
            material.SetFloat("_FogEnd", fogEnd);

            material.SetTexture("_NoiseTex", noiseTexture);
            material.SetFloat("_FogXSpeed", fogXSpeed);
            material.SetFloat("_FogYSpeed", fogYSpeed);
            material.SetFloat("_NoiseAmount", noiseAmount);

            Graphics.Blit(src, dest, material);
        }
        else
        {
            Graphics.Blit(src, dest);
        }
    }
}
```
```GLSL
// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'

Shader "Unity Shaders Book/Chapter 15/Fog With Noise" 
{
	Properties 
	{
		_MainTex ("Base (RGB)", 2D) = "white" {}
		_FogDensity ("Fog Density", Float) = 1.0
		_FogColor ("Fog Color", Color) = (1, 1, 1, 1)
		_FogStart ("Fog Start", Float) = 0.0
		_FogEnd ("Fog End", Float) = 1.0
		_NoiseTex ("Noise Texture", 2D) = "white" {}
		_FogXSpeed ("Fog Horizontal Speed", Float) = 0.1
		_FogYSpeed ("Fog Vertical Speed", Float) = 0.1
		_NoiseAmount ("Noise Amount", Float) = 1
	}
	SubShader 
	{
		CGINCLUDE
		
		#include "UnityCG.cginc"
		
		float4x4 _FrustumCornersRay;
		
		sampler2D _MainTex;
		half4 _MainTex_TexelSize;
		sampler2D _CameraDepthTexture;
		half _FogDensity;
		fixed4 _FogColor;
		float _FogStart;
		float _FogEnd;
		sampler2D _NoiseTex;
		half _FogXSpeed;
		half _FogYSpeed;
		half _NoiseAmount;
		
		struct v2f 
		{
			float4 pos : SV_POSITION;
			float2 uv : TEXCOORD0;
			float2 uv_depth : TEXCOORD1;
			float4 interpolatedRay : TEXCOORD2;
		};
		
		v2f vert(appdata_img v) 
		{
			v2f o;
			o.pos = UnityObjectToClipPos(v.vertex);
			
			o.uv = v.texcoord;
			o.uv_depth = v.texcoord;
			
			#if UNITY_UV_STARTS_AT_TOP
			if (_MainTex_TexelSize.y < 0)
				o.uv_depth.y = 1 - o.uv_depth.y;
			#endif
			
			int index = 0;
			if (v.texcoord.x < 0.5 && v.texcoord.y < 0.5) 
			{
				index = 0;
			} 
			else if (v.texcoord.x > 0.5 && v.texcoord.y < 0.5) 
			{
				index = 1;
			} 
			else if (v.texcoord.x > 0.5 && v.texcoord.y > 0.5) 
			{
				index = 2;
			} 
			else 
			{
				index = 3;
			}
			#if UNITY_UV_STARTS_AT_TOP
			if (_MainTex_TexelSize.y < 0)
				index = 3 - index;
			#endif
			
			o.interpolatedRay = _FrustumCornersRay[index];
				 	 
			return o;
		}
		
		fixed4 frag(v2f i) : SV_Target 
		{
		    //先根据深度纹理重建该像素在世界空间中的位置
			float linearDepth = LinearEyeDepth(SAMPLE_DEPTH_TEXTURE(_CameraDepthTexture, i.uv_depth));
			float3 worldPos = _WorldSpaceCameraPos + linearDepth * i.interpolatedRay.xyz;
			//计算当前噪声纹理偏移量
			float2 speed = _Time.y * float2(_FogXSpeed, _FogYSpeed);
			//最终噪声值
			float noise = (tex2D(_NoiseTex, i.uv + speed).r - 0.5) * _NoiseAmount;
					
			float fogDensity = (_FogEnd - worldPos.y) / (_FogEnd - _FogStart); 
			fogDensity = saturate(fogDensity * _FogDensity * (1 + noise));
			
			fixed4 finalColor = tex2D(_MainTex, i.uv);
			//颜色混合
			finalColor.rgb = lerp(finalColor.rgb, _FogColor.rgb, fogDensity);
			
			return finalColor;
		}
		
		ENDCG
		
		Pass 
		{          	
			CGPROGRAM  
			
			#pragma vertex vert  
			#pragma fragment frag  
			  
			ENDCG
		}
	} 
	FallBack Off
}

```
[![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/11/10-f4v.gif)](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/11/10-f4v.gif)
# 扩展阅读
噪声纹理可以被认为是一种程序纹理，都是计算机利用某些算法生成的。
最常用的有Perlin噪声和Worley噪声。
Perlin噪声可以用于生成更加自然的噪声纹理，Worley噪声通常用于模拟诸如石头，水，纸张等多孔噪声。
