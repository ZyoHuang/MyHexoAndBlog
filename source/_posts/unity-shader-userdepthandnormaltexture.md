---
title: Unity Shader入门精要学习笔记：使用深度和法线纹理
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
cover: https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/11/QQ截图20191102213006.png
katex: true
aplayer:
---
<meta name="referrer" content="no-referrer" />

# 前言
在进行边缘检测时，直接利用颜色信息会使检测到的边缘信息受物体纹理和光照等外部因素影响，得到很多我们不需要的边缘点。
我们可以在`深度纹理和法线纹理上进行边缘检测`，这些图像不会受光照和纹理的影响，仅仅保存了当前渲染物体的模型信息，通过这样的方式检测出来的边缘更加可靠。
# 获取深度和法线纹理
## 背后的原理
深度纹理实际就是一张渲染纹理，里面存储的像素值是高精度的深度值，由于被存储在一张纹理中，`深度纹理中深度值范围为[0,1]，通常是非线性分布的。`
这些深度值来自于顶点变换后得到的`归一化的设备坐标`（NDC）。
看下面一组透视相机投影变换的过程图（`使用的变换矩阵是非线性的`），最左边是投影变换前（观察空间下视锥体的结构及相应的顶点位置），中间是应用透视裁剪矩阵后的变换结果（顶点着色器阶段输出的顶点变换结果），最右边是底层硬件进行了透视除法后得到的归一化的设备坐标。
[![测试](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/11/QQ截图20191102213006.png)](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/11/QQ截图20191102213006.png)
看下面一组正交相机投影变换的过程图（`使用的变换矩阵是线性的`）
[![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/11/QQ截图20191102213159.png)](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/11/QQ截图20191102213159.png)
在得到NDC后，深度纹理中的像素值就可以很方便的计算得到了，这些深度值就对应了NDC中顶点坐标的z分量的值，由于NDC中z分量的范围在[-1,1]，为了让这些值能存储在一张图像中，需要用下面公式映射
$$d\;=\;0.5\cdot z_{NDC}+0.5$$
`d对应深度纹理中的像素值，$z_{NDC}$对应NDC坐标中的z分量的值。`
在Unity中，`深度纹理可以直接来自于真正的深度缓存，也可以由一个单独的Pass渲染而得`，这取决于使用的渲染路径和硬件。
当使用延迟渲染路径（包括遗留的延迟渲染路径）时，深度纹理理所当然可以访问到，因为延迟渲染会把这些信息渲染到G-buffer，而当无法直接获取深度缓存时，深度和法线纹理是通过一个单独的Pass渲染而得的。
`Unity会使用着色器替换（Shader Replacement）技术选择那些渲染类型（SubShader的RenderType标签）为Opaque的物体，判断他们使用的渲染队列是否小于等于2500（内置的Background，Geometry和AlphaTest渲染队列均在此范围内），如果满足条件，就把它渲染到深度和法线纹理中。`

## 如何获取
在Unity中获取深度纹理是非常简单的
`camera.depthTextureMode = DepthTextureMode.Depth;`
然后我们可以在Shader中通过声明_CameraDepthTexture变量访问它。
如果想获取深度+法线纹理
`camera.depthTextureMode = DepthTextureMode.DepthNormals;`
然后我们可以在Shader中通过声明_CameraDepthNormalsTexture变量来访问他。
还可以使用组合模式，让相机同时产生一张深度和深度+法线纹理
`camera.depthTextureMode = DepthTextureMode.DepthNormals | DepthTextureMode.Depth;`
我们只需要在Shader中使用`float d = SAMPLE_DEPTH_TEXTURE(_CameraDepthTexture,i.uv);`就可以对深度纹理进行采样。
当通过纹理采样得到深度值后，这些深度值往往是非线性的，这种非线性来自透视投影使用的裁剪矩阵。然鹅我们计算过程中通常是需要线性的深度值，我们需要把投影后的深度值变换到线性空间下，例如视角空间下的深度值。
Unity提供了两个辅助函数为我们进行变换，`LinearEyeDepth负责把深度纹理的采样结果转换到视角空间下的深度值，Linear01Depth则会返回一个范围在[0,1]的线性深度值`。这两个函数内部使用了内置的_ZBufferParams变量来得到远近裁剪平面的距离。
如果我们需要获取深度+法线纹理，可以直接使用tex2D函数对_CameraDepthNormalsTexture进行采样，得到里面存储的深度和法线信息。Unity提供了辅助函数为这个采样结果进行解码，这个函数是DecodeDepthNormal，第一个参数是对深度+法线纹理的采样结果，这个采样结果是Unity对深度和法线信息编码后的结果，它的xy分量存储的是视角空间下的法线信息，zw分量存储的是编码后的深度信息。`解码后得到的是范围[0,1]的线性深度值与视角空间下的法线方向。`
也可以使用`DecodeFloatRG`和`DecodeBiewNormalStereo`来解码深度+法线纹理中的深度和法线信息。
# 再谈运动模糊
一种应用更加广泛的技术是使用速度映射图，速度映射图存储了每个像素的速度，然后使用这个速度来决定模糊的方向和大小。
## 如何生成速度映射图
利用深度纹理在片元着色器中为每个像素计算其在世界空间下的位置，这是通过使用`当前视角*投影矩阵的逆矩阵`对NDC下的顶点坐标进行变换得到的。得到世界空间中的顶点坐标后，我们使用`前一帧的视角*投影矩阵`对其进行变换，得到该位置在前一帧的NDC坐标，然后计算前一帧和当前帧的位置差，生成该像素的速度，
`这种方法有点是可以在一个屏幕后处理步骤中完成整个效果的模拟，但缺点就是需要在片元着色器进行两次矩阵乘法，对性能不友好`。
```csharp
using UnityEngine;
using System.Collections;

public class MotionBlurWithDepthTexture : PostEffectsBase
{
    public Shader motionBlurShader;
    private Material motionBlurMaterial = null;

    public Material material
    {
        get
        {
            motionBlurMaterial = CheckShaderAndCreateMaterial(motionBlurShader, motionBlurMaterial);
            return motionBlurMaterial;
        }
    }

    /// <summary>
    /// 缓存相机
    /// </summary>
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

    [Range(0.0f, 1.0f)] public float blurSize = 0.5f;

    /// <summary>
    /// 保存上一帧摄像机视角*投影矩阵
    /// </summary>
    private Matrix4x4 previousViewProjectionMatrix;

    void OnEnable()
    {
        //设置摄像机状态，用以获取深度纹理
        camera.depthTextureMode |= DepthTextureMode.Depth;

        previousViewProjectionMatrix = camera.projectionMatrix * camera.worldToCameraMatrix;
    }

    void OnRenderImage(RenderTexture src, RenderTexture dest)
    {
        if (material != null)
        {
            material.SetFloat("_BlurSize", blurSize);

            material.SetMatrix("_PreviousViewProjectionMatrix", previousViewProjectionMatrix);
            Matrix4x4 currentViewProjectionMatrix = camera.projectionMatrix * camera.worldToCameraMatrix;
            Matrix4x4 currentViewProjectionInverseMatrix = currentViewProjectionMatrix.inverse;
            material.SetMatrix("_CurrentViewProjectionInverseMatrix", currentViewProjectionInverseMatrix);
            previousViewProjectionMatrix = currentViewProjectionMatrix;

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

Shader "Unity Shaders Book/Chapter 13/Motion Blur With Depth Texture" 
{
	Properties 
	{
		_MainTex ("Base (RGB)", 2D) = "white" {}
		_BlurSize ("Blur Size", Float) = 1.0
	}
	SubShader 
	{
		CGINCLUDE
		
		#include "UnityCG.cginc"
		
		sampler2D _MainTex;
		//主纹素大小，需要使用该变量对深度文里的采样坐标进行平台差异化处理
		half4 _MainTex_TexelSize;
		//Unity传递的深度纹理
		sampler2D _CameraDepthTexture;
		//当前帧的视角*投影逆矩阵
		float4x4 _CurrentViewProjectionInverseMatrix;
		//前一帧帧的视角*投影矩阵
		float4x4 _PreviousViewProjectionMatrix;
		half _BlurSize;
		
		struct v2f 
		{
			float4 pos : SV_POSITION;
			half2 uv : TEXCOORD0;
			half2 uv_depth : TEXCOORD1;
		};
		
		v2f vert(appdata_img v) 
		{
			v2f o;
			o.pos = UnityObjectToClipPos(v.vertex);
			
			o.uv = v.texcoord;
			o.uv_depth = v.texcoord;
			//平台差异化处理
			#if UNITY_UV_STARTS_AT_TOP
			if (_MainTex_TexelSize.y < 0)
				o.uv_depth.y = 1 - o.uv_depth.y;
			#endif
					 
			return o;
		}
		
		fixed4 frag(v2f i) : SV_Target 
		{
			// 对深度纹理进行采样，得到深度值d
			float d = SAMPLE_DEPTH_TEXTURE(_CameraDepthTexture, i.uv_depth);
			// d是由NDC下的坐标映射来的，想要构建像素NDC坐标H，就需要重新映射回NDC
			float4 H = float4(i.uv.x * 2 - 1, i.uv.y * 2 - 1, d * 2 - 1, 1);
			// 使用当前帧的视角*投影矩阵的逆矩阵进行变换
			float4 D = mul(_CurrentViewProjectionInverseMatrix, H);
			// 得到世界空间下的坐标表示worldPos
			float4 worldPos = D / D.w;
			
			// 当前NDC下的位置
			float4 currentPos = H;
			// 使用前一帧的视角*投影矩阵进行变换，得到前一帧NDC空间下的位置 
			float4 previousPos = mul(_PreviousViewProjectionMatrix, worldPos);
			// 通过除以w转换成非齐次点[-1,1]
			previousPos /= previousPos.w;
			
			// 计算速度
			float2 velocity = (currentPos.xy - previousPos.xy)/2.0f;
			
			float2 uv = i.uv;
			float4 c = tex2D(_MainTex, uv);
			uv += velocity * _BlurSize;
			for (int it = 1; it < 3; it++, uv += velocity * _BlurSize)
			{
				float4 currentColor = tex2D(_MainTex, uv);
				c += currentColor;
			}
			//平均模糊
			c /= 3;
			
			return fixed4(c.rgb, 1.0);
		}
		
		ENDCG
		
		Pass 
		{      
			ZTest Always Cull Off ZWrite Off
			    	
			CGPROGRAM  
			
			#pragma vertex vert  
			#pragma fragment frag  
			  
			ENDCG  
		}
	} 
	FallBack Off
}

```
# 全局雾效
基于屏幕后处理的全局雾效的关键是，根据深度纹理来重建每个像素在世界空间下的位置，在上面的再谈运动模糊哪里，我们已经实现了，即构建出当前像素的NDC坐标，再通过`当前摄像机的视角*投影矩阵的逆矩阵`来得到世界空间下的像素坐标，但是这样实现需要在片元着色器中进行矩阵乘法的操作，这通常会印象游戏性能。
现在有一个更好的办法：首先对图像空间下的视锥体射线（从摄像机出发指向图像上的某点的射线）进行插值，这条射线存储的该像素在世界空间下到摄像机的方向信息。然后把该射线和线性化后的视角空间深度值相乘，再加上摄像机的世界为之，就可以得到该像素在世界空间下的位置。
## 重建世界坐标
我们知道，坐标系中的一个顶点坐标可以通过它相对于另一个顶点坐标的偏移量来求得，重建像素的世界坐标也是如此，我们`只需要知道摄像机在世界空间下的位置，以及世界空间下该像素相对于摄像机的偏移量，把他们相加就可以得到该像素的世界坐标`。
`float4 worldPos = _WorldSpaceCameraPos + linearDepth * interpolatedRay;`
`_WorldSpaceCameraPos`是摄像机在世界空间下的位置，这可以由Unity的内置变量直接访问得到，`linearDepth * interpolatedRay`则可以计算得到该像素相对于摄像机的偏移量，linearDepth是由深度纹理的到的线性深度值，`interpolatedRay是由顶点着色器输出并插值后得到的射线，不仅包含了该像素到摄像机的方向，也包含了距离信息。`
interpolatedRay来源于对近裁剪平面的4个角的某个特定向量的插值，这4个向量包含了他们到摄像机方向和距离，可以利用摄像机的近裁剪平面距离，FOV，横纵比计算而得。
[![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/11/QQ截图20191103173348.png)](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/11/QQ截图20191103173348.png)
$$halfHeight\;=\;Near\times\tan\left(\frac{FOV}2\right)$$
$$toTop\;=\;camera.up\;\times\;halfHeight$$
$$toRight\;=\;camera.right\;\times\;half\;\times\;halfHeight.aspect$$
Near为相机离近面的距离，FOV为竖直方向的视角范围。
[![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/11/QQ截图20191103175726.png)](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/11/QQ截图20191103175726.png)
然后计算4个角对于摄像机的方向。
$$TL\;=\;camera.forward\cdot Near\;+\;toTop\;-\;toRight$$
$$TR\;=\;camera.forward\cdot Near\;+\;toTop\;+\;toRight$$
$$BL\;=\;camera.forward\cdot Near\;-\;toTop\;-\;toRight$$
$$BR\;=\;camera.forward\cdot Near\;-\;toTop\;+\;toRight$$
他们的模对应4个点到摄像机的空间距离。
我们还需要计算深度值距离摄像机的欧氏距离。以TL所在射线上的一点A为例
[![灵魂画手，勿怪](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/11/QQ截图20191103182524.png)](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/11/QQ截图20191103182524.png)
根据相似三角形有
$$\because\frac{depth}{dist}\;=\;\frac{Near}{\left|\overrightarrow{TL}\right|}\;\therefore dist\;=\;\frac{\left|\overrightarrow{TL}\right|}{Near}\times depth$$
所以可以得到一个因子，与单位向量相乘，就能得到他们对应的向量值
$$scale\;=\;\frac{\left|\overrightarrow{TL}\right|}{Near}$$
即有
$$Ray_{TL}\;=\;\frac{\overrightarrow{TL}}{\left|\overrightarrow{TL}\right|}\times scale,\;\;\;Ray_{TR}\;=\;\frac{\overrightarrow{TR}}{\left|\overrightarrow{TR}\right|}\times scale\\Ray_{BL}\;=\;\frac{\overrightarrow{BL}}{\left|\overrightarrow{BL}\right|}\times scale,\;\;\;Ray_{BR}\;=\;\frac{\overrightarrow{BR}}{\left|\overrightarrow{BR}\right|}\times scale$$
`屏幕后处理的原理是使用特定材质去渲染一个刚好填充整个屏幕的四边形面片。
这个四边形面片的4个顶点就对应了近裁剪平面的4个角，因此我们可以把上面的计算结果传递给顶点着色器，顶点着色器根据当前位置选择它对应的向量，然后输出，经过插值后传递给片元着色器得到interpolateRay。`
## 雾的计算
在简单的雾效实现中，我们需要计算一个`雾效系数f`，作为混合原始颜色和雾颜色的混合系数。
`float3 afterFog = f * fogColor + (1 - f) * origColor;`
这个系数在Unity内置雾效实现中，支持三种雾的计算方式——`线性（Linear）`，`指数（Exponential）`，`指数的平方（Exponential Squared）`。当给定距离z后，f的计算公式如下
Linear：
$f\;=\;\frac{d_{max}\;-\;\left|z\right|}{d_{max}\;-\;d_{ming}}$，$d_{max}$和$d_{min}$分别表示受雾影响的最小距离和最大距离。
Exponential：
$f\;=\;e^{-d\cdot\left|z\right|}$，d是控制雾的浓度的参数。
Exponential Squared：
$f\;=\;e^{-\left(d\;\cdot\;\left|z\right|\right)^2}$，d是控制雾的浓度的参数。
下面我们使用类似线性雾的计算方式，计算基于高度的雾效，
当给定一点在世界空间下的高度y后，f的计算公式为：
$f\;=\;\frac{H_{end}\;-\;y}{H_{end}\;-\;H_{start}}$，$H_{start}$和$H_{end}$分别表示受雾影响的起始高度和终止高度。
## 实现
```csharp
using UnityEngine;
using System.Collections;

public class FogWithDepthTexture : PostEffectsBase
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
    [Range(0.0f, 3.0f)] public float fogDensity = 1.0f;

    /// <summary>
    /// 雾颜色
    /// </summary>
    public Color fogColor = Color.white;

    /// <summary>
    /// 起始高度
    /// </summary>
    public float fogStart = 0.0f;

    /// <summary>
    /// 终止高度
    /// </summary>
    public float fogEnd = 2.0f;

    void OnEnable()
    {
        camera.depthTextureMode |= DepthTextureMode.Depth;
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

            //计算近裁剪平面四个角对应向量，并存储在一个矩阵类型的变量中
            frustumCorners.SetRow(0, bottomLeft);
            frustumCorners.SetRow(1, bottomRight);
            frustumCorners.SetRow(2, topRight);
            frustumCorners.SetRow(3, topLeft);

            material.SetMatrix("_FrustumCornersRay", frustumCorners);

            material.SetFloat("_FogDensity", fogDensity);
            material.SetColor("_FogColor", fogColor);
            material.SetFloat("_FogStart", fogStart);
            material.SetFloat("_FogEnd", fogEnd);

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

Shader "Unity Shaders Book/Chapter 13/Fog With Depth Texture" 
{
	Properties 
	{
		_MainTex ("Base (RGB)", 2D) = "white" {}
		_FogDensity ("Fog Density", Float) = 1.0
		_FogColor ("Fog Color", Color) = (1, 1, 1, 1)
		_FogStart ("Fog Start", Float) = 0.0
		_FogEnd ("Fog End", Float) = 1.0
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
		
		struct v2f 
		{
			float4 pos : SV_POSITION;
			half2 uv : TEXCOORD0;
			half2 uv_depth : TEXCOORD1;
			float4 interpolatedRay : TEXCOORD2;
		};
		
		v2f vert(appdata_img v) 
		{
			v2f o;
			o.pos = UnityObjectToClipPos(v.vertex);
			
			o.uv = v.texcoord;
			o.uv_depth = v.texcoord;
			//平台差异化处理
			#if UNITY_UV_STARTS_AT_TOP
			if (_MainTex_TexelSize.y < 0)
				o.uv_depth.y = 1 - o.uv_depth.y;
			#endif
			
			//根据纹理坐标特性判定属于哪个角
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
			//使用索引值来获取_FrustumCornersRay中对应行为作为该顶点interpolatedRay值
			o.interpolatedRay = _FrustumCornersRay[index];
				 	 
			return o;
		}
		
		fixed4 frag(v2f i) : SV_Target 
		{
		    //先对深度纹理进行采样，再得到视角空间下的线性深度值
			float linearDepth = LinearEyeDepth(SAMPLE_DEPTH_TEXTURE(_CameraDepthTexture, i.uv_depth));
			//得到世界空间下的位置
			float3 worldPos = _WorldSpaceCameraPos + linearDepth * i.interpolatedRay.xyz;
			float viewDistance = length(worldPos - _WorldSpaceCameraPos);
		    //计算得雾浓度
		    //线性
			float fogDensity = (_FogEnd - worldPos.y) / (_FogEnd - _FogStart); 
			//指数
			//float fogDensity = exp(-_FogDensity * viewDistance);
			//指数平方
			//float fogDensity = exp(-pow(viewDistance * _FogDensity, 2));
			
			//截取[0,1]
			fogDensity = saturate(fogDensity);
			
			fixed4 finalColor = tex2D(_MainTex, i.uv);
			//雾颜色与原始颜色混合
			//线性混合
			finalColor.rgb = lerp(finalColor.rgb, _FogColor.rgb,  fogDensity);
			//指数混合(远浓近稀)
			//finalColor.rgb = lerp(_FogColor.rgb, finalColor.rgb,  fogDensity);
			return finalColor;
		}
		
		ENDCG
		
		Pass 
		{
			ZTest Always Cull Off ZWrite Off
			     	
			CGPROGRAM  
			
			#pragma vertex vert  
			#pragma fragment frag  
			  
			ENDCG  
		}
	} 
	FallBack Off
}

```
[![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/11/QQ截图20191106145933.png)](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/11/QQ截图20191106145933.png)
[![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/11/QQ截图20191105214417.png)](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/11/QQ截图20191105214417.png)
# 再谈边缘检测
之前我们使用Sobel算子对屏幕图像进行边缘检测实现描边，但是这种`直接利用颜色信息进行边缘检测的方法会产生很多我们不希望得到的边缘线`。
下面我们会基于深度和法线纹理进行边缘检测，这些图像不会受到纹理和光照的影响，仅仅保存了当前渲染物体的模型信息。
[![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/11/QQ截图20191106220023.png)](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/11/QQ截图20191106220023.png)
使用Roberts算子，它的卷积核如下
[![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/11/QQ截图20191106220210-t02.png)](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/11/QQ截图20191106220210-t02.png)
Roberts算子本质就是计算左上角和右下角的差值，乘以右上角和左下角的差值，作为评估边缘的依据，具体一下就是`取对角方向的深度或法线值，比较他们之间的差值，如果超过某个阈值（可由参数控制），就认为他们之间存在一条边。`
```csharp
using UnityEngine;
using System.Collections;

public class EdgeDetectNormalsAndDepth : PostEffectsBase
{
    public Shader edgeDetectShader;
    private Material edgeDetectMaterial = null;

    public Material material
    {
        get
        {
            edgeDetectMaterial = CheckShaderAndCreateMaterial(edgeDetectShader, edgeDetectMaterial);
            return edgeDetectMaterial;
        }
    }

    /// <summary>
    /// 描边程度，为一时只显示描边
    /// </summary>
    [Range(0.0f, 1.0f)] public float edgesOnly = 0.0f;

    /// <summary>
    /// 描边颜色
    /// </summary>
    public Color edgeColor = Color.black;

    /// <summary>
    /// 背景色
    /// </summary>
    public Color backgroundColor = Color.white;

    /// <summary>
    /// 采样距离
    /// </summary>
    public float sampleDistance = 1.0f;

    /// <summary>
    /// 深度纹理灵敏度（相差多少时会认为存在一条边界）
    /// </summary>
    public float sensitivityDepth = 1.0f;

    /// <summary>
    /// 法线纹理灵敏度（相差多少时会认为存在一条边界）
    /// </summary>
    public float sensitivityNormals = 1.0f;

    void OnEnable()
    {
        GetComponent<Camera>().depthTextureMode |= DepthTextureMode.DepthNormals;
    }

    /// <summary>
    /// ImageEffectOpaque特性可以在不透明Pass执行完毕后就执行，可以不对透明物体产生影响
    /// </summary>
    /// <param name="src"></param>
    /// <param name="dest"></param>
    [ImageEffectOpaque]
    void OnRenderImage(RenderTexture src, RenderTexture dest)
    {
        if (material != null)
        {
            material.SetFloat("_EdgeOnly", edgesOnly);
            material.SetColor("_EdgeColor", edgeColor);
            material.SetColor("_BackgroundColor", backgroundColor);
            material.SetFloat("_SampleDistance", sampleDistance);
            material.SetVector("_Sensitivity", new Vector4(sensitivityNormals, sensitivityDepth, 0.0f, 0.0f));

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

Shader "Unity Shaders Book/Chapter 13/Edge Detection Normals And Depth" 
{
	Properties 
	{
		_MainTex ("Base (RGB)", 2D) = "white" {}
		_EdgeOnly ("Edge Only", Float) = 1.0
		_EdgeColor ("Edge Color", Color) = (0, 0, 0, 1)
		_BackgroundColor ("Background Color", Color) = (1, 1, 1, 1)
		_SampleDistance ("Sample Distance", Float) = 1.0
		_Sensitivity ("Sensitivity", Vector) = (1, 1, 1, 1)
	}
	SubShader 
	{
		CGINCLUDE
		
		#include "UnityCG.cginc"
		
		sampler2D _MainTex;
		//需要对邻域像素采样，所以声明存储纹素大小变量
		half4 _MainTex_TexelSize;
		fixed _EdgeOnly;
		fixed4 _EdgeColor;
		fixed4 _BackgroundColor;
		float _SampleDistance;
		half4 _Sensitivity;
		
		sampler2D _CameraDepthNormalsTexture;
		
		struct v2f 
		{
			float4 pos : SV_POSITION;
			half2 uv[5]: TEXCOORD0;
		};
		  
		v2f vert(appdata_img v) 
		{
			v2f o;
			o.pos = UnityObjectToClipPos(v.vertex);
			
			half2 uv = v.texcoord;
			o.uv[0] = uv;
			
			#if UNITY_UV_STARTS_AT_TOP
			if (_MainTex_TexelSize.y < 0)
				uv.y = 1 - uv.y;
			#endif
			
			o.uv[1] = uv + _MainTex_TexelSize.xy * half2(1,1) * _SampleDistance;
			o.uv[2] = uv + _MainTex_TexelSize.xy * half2(-1,-1) * _SampleDistance;
			o.uv[3] = uv + _MainTex_TexelSize.xy * half2(-1,1) * _SampleDistance;
			o.uv[4] = uv + _MainTex_TexelSize.xy * half2(1,-1) * _SampleDistance;
					 
			return o;
		}
		
		half CheckSame(half4 center, half4 sample) 
		{
		    //得到两个采样点的深度和法线值
			half2 centerNormal = center.xy;
			float centerDepth = DecodeFloatRG(center.zw);
			half2 sampleNormal = sample.xy;
			float sampleDepth = DecodeFloatRG(sample.zw);
			
			//只需要对比采样值差异，不需要解码求得真正法线值
			half2 diffNormal = abs(centerNormal - sampleNormal) * _Sensitivity.x;
			int isSameNormal = (diffNormal.x + diffNormal.y) < 0.1;
			
			float diffDepth = abs(centerDepth - sampleDepth) * _Sensitivity.y;
			int isSameDepth = diffDepth < 0.1 * centerDepth;
			
			//1说明差异不明显，0说明存在边界
			return isSameNormal * isSameDepth ? 1.0 : 0.0;
		}
		
		fixed4 fragRobertsCrossDepthAndNormal(v2f i) : SV_Target
		{
			half4 sample1 = tex2D(_CameraDepthNormalsTexture, i.uv[1]);
			half4 sample2 = tex2D(_CameraDepthNormalsTexture, i.uv[2]);
			half4 sample3 = tex2D(_CameraDepthNormalsTexture, i.uv[3]);
			half4 sample4 = tex2D(_CameraDepthNormalsTexture, i.uv[4]);
			
			half edge = 1.0;
			//计算两个对角线纹理差值
			edge *= CheckSame(sample1, sample2);
			edge *= CheckSame(sample3, sample4);
			
			fixed4 withEdgeColor = lerp(_EdgeColor, tex2D(_MainTex, i.uv[0]), edge);
			fixed4 onlyEdgeColor = lerp(_EdgeColor, _BackgroundColor, edge);
			
			return lerp(withEdgeColor, onlyEdgeColor, _EdgeOnly);
		}
		
		ENDCG
		
		Pass 
		{ 
			ZTest Always Cull Off ZWrite Off
			
			CGPROGRAM      
			
			#pragma vertex vert  
			#pragma fragment fragRobertsCrossDepthAndNormal
			
			ENDCG  
		}
	} 
	FallBack Off
}

```
这个描边效果是基于整个屏幕空间的，场景内所有物体都会被添加描边效果，`但有时我们只希望对特定物体进行描边。我们可以用Unity提供的Graphic.DrawMesh或Graphic.DrawMeshNow把需要描边的物体再渲染一遍（在所有不透明物体渲染完毕后），然后再使用这个边缘检测算法计算深度或法线纹理中每个像素梯度值，判断是否小于某个阈值，如果是，就是用clip()函数裁剪像素，显示原来的物体颜色。`
