---
title: Unity Shader入门精要学习笔记：屏幕后处理效果
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

# 建立一个基本的屏幕后处理脚本系统
屏幕后处理通常是在渲染完整个场景得到屏幕图像后，再对这个图像进行一系列操作，实现各种屏幕特效。例如景深（Depth of Field），运动模糊（Motion Blur）等。
要实现屏幕后处理的基础在于得到渲染后的屏幕图像。
```csharp
/// <summary>
/// 抓取屏幕图像
/// </summary>
/// <param name="src">Unity会把当前渲染的得到的图像存储在第一个参数对应的源渲染纹理中</param>
/// <param name="dest">目标渲染纹理</param>
MonoBehaviour.OnRenderImage (RenderTexture src, RenderTexture dest)
```
```csharp
/// <summary>
/// 完成对渲染纹理的处理
/// </summary>
/// <param name="source">源纹理，通常是当前屏幕的渲染纹理或者上一步处理得到的渲染纹理</param>
/// <param name="dest">目标渲染纹理，如果值为null会直接把它显示在屏幕上</param>
/// <param name="mat">使用的材质，材质的shader可以进行一些后处理特效</param>
/// <param name="pass">如果是-1，会调用mat里shader的所有pass，不然只会调用指定的pass</param>
/// <param name="offset">源纹理的偏移</param>
/// <param name="scale">源纹理的缩放</param>
public static void Blit(Texture source, RenderTexture dest, Material mat, [DefaultValue("-1")] int pass)
{
	...
}
```
默认情况下，OnRenderImage函数会在所有的不透明和透明的Pass执行完毕后被调用，以便对场景所有所有游戏对象都产生影响。
有时我们希望在不透明的Pass（渲染队列小于等于2500的Pass，内置的Background，Geometry和AlphaTest渲染队列都在此范围内）执行完毕后立即调用OnRenderImage函数，从而不对透明物体产生影响。
我们可以通过在`OnRenderImage函数前添加ImageEffectOpaque`特性来实现。
```csharp
[ImageEffectOpaque]
void OnRenderImage(RenderTexture src, RenderTexture dest)
{
	...
}
```
想在Unity实现屏幕后处理效果，过程如下：

1. 检查一列条件是否满足，例如平台是否支持渲染纹理和屏幕特效
2. 在摄像机添加一个用于屏幕后处理的脚本
3. 在这个脚本中实现OnRenderImage函数来获取当前屏幕的渲染纹理
4. 调用Graphics.Blit函数使用特定的Unity Shader来对当前图像进行处理
5. 把返回的渲染纹理显示到屏幕上
6. 对于复杂的屏幕特效，可以多次调用Graphics.Blit函数来对上一步的输出结果进行下一步处理

```csharp
using UnityEngine;


[ExecuteInEditMode]//这个特性意为在编辑器模式下也可以运行
[RequireComponent(typeof(Camera))]
public class PostEffectsBase : MonoBehaviour
{
    protected void CheckResources()
    {
        bool isSupported = CheckSupport();

        if (isSupported == false)
        {
            NotSupported();
        }
    }
    
    /// <summary>
    /// 判断是否支持图片特效和渲染纹理
    /// </summary>
    /// <returns></returns>
    protected bool CheckSupport()
    {
        if (SystemInfo.supportsImageEffects == false || SystemInfo.supportsRenderTextures == false)
        {
            Debug.LogWarning("This platform does not support image effects or render textures.");
            return false;
        }

        return true;
    }

    /// <summary>
    /// 如果检查没有通过，就不执行了
    /// </summary>
    protected void NotSupported()
    {
        enabled = false;
    }

    protected void Start()
    {
        CheckResources();
    }

    /// <summary>
    /// 检查shader并创建材质
    /// </summary>
    /// <param name="shader"></param>
    /// <param name="material"></param>
    /// <returns></returns>
    protected Material CheckShaderAndCreateMaterial(Shader shader, Material material)
    {
        if (shader == null)
        {
            return null;
        }

        if (shader.isSupported && material && material.shader == shader)
            return material;

        if (!shader.isSupported)
        {
            return null;
        }

        material = new Material(shader);
        material.hideFlags = HideFlags.DontSave;
        return material ? material : null;
    }
}
```
# 调整屏幕的亮度，饱和度和对比度
```csharp
using UnityEngine;
using System.Collections;

public class BrightnessSaturationAndContrast : PostEffectsBase
{
    public Shader briSatConShader;
    private Material briSatConMaterial;

    public Material material
    {
        get
        {
            briSatConMaterial = CheckShaderAndCreateMaterial(briSatConShader, briSatConMaterial);
            return briSatConMaterial;
        }
    }

    [Range(0.0f, 3.0f)] public float brightness = 1.0f;

    [Range(0.0f, 3.0f)] public float saturation = 1.0f;

    [Range(0.0f, 3.0f)] public float contrast = 1.0f;

    void OnRenderImage(RenderTexture src, RenderTexture dest)
    {
        if (material != null)
        {
            material.SetFloat("_Brightness", brightness);
            material.SetFloat("_Saturation", saturation);
            material.SetFloat("_Contrast", contrast);

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

Shader "Unity Shaders Book/Chapter 12/Brightness Saturation And Contrast" 
{
	Properties 
	{
	    //Graphics.Blit(src, dest, material)会把第一个参数传递给shader中名为__MainTex的属性
		_MainTex ("Base (RGB)", 2D) = "white" {}
		//亮度
		_Brightness ("Brightness", Float) = 1
		//饱和度
		_Saturation("Saturation", Float) = 1
		//对比度
		_Contrast("Contrast", Float) = 1
	}
	SubShader 
	{
		Pass 
		{  
		    //屏幕后处理实际上是在场景中绘制了一个与屏幕同宽同高的四边形面片
		    //为了防止它“挡住”其后面被渲染物体，需要深度测试总是通过和关闭深度写入
			ZTest Always Cull Off ZWrite Off
			
			CGPROGRAM  
			#pragma vertex vert  
			#pragma fragment frag  
			  
			#include "UnityCG.cginc"  
			  
			sampler2D _MainTex;  
			half _Brightness;
			half _Saturation;
			half _Contrast;
			  
			struct v2f 
			{
				float4 pos : SV_POSITION;
				half2 uv: TEXCOORD0;
			};
			  
			//appdata_img是unity内置的结构体
			//只包含了图像处理时必需的顶点坐标和纹理坐标等变量
			v2f vert(appdata_img v) 
			{
				v2f o;
				
				o.pos = UnityObjectToClipPos(v.vertex);
				
				o.uv = v.texcoord;
						 
				return o;
			}
		
			fixed4 frag(v2f i) : SV_Target 
			{
			    //采样
				fixed4 renderTex = tex2D(_MainTex, i.uv);  
				  
				//计算亮度
				fixed3 finalColor = renderTex.rgb * _Brightness;
				
				//计算饱和度
				fixed luminance = 0.2125 * renderTex.r + 0.7154 * renderTex.g + 0.0721 * renderTex.b;
				fixed3 luminanceColor = fixed3(luminance, luminance, luminance);
				finalColor = lerp(luminanceColor, finalColor, _Saturation);
				
				//计算对比度
				fixed3 avgColor = fixed3(0.5, 0.5, 0.5);
				finalColor = lerp(avgColor, finalColor, _Contrast);
				
				return fixed4(finalColor, renderTex.a);  
			}  
			  
			ENDCG
		}  
	}
	
	Fallback Off
}

```
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/10/QQ截图20191023171026.png)
# 边缘检测
边缘检测的原理是利用一些`边缘检测算子对图像进行卷积操作`。
## 什么是卷积
在图像处理中，卷积操作指的就是使用一个`卷积核`对一张图像中的每个像素进行一系列操作。
卷积核通常是一个四方形网格结构（例如2x2,3x3的方形区域），该区域内每个方格都有一个权重值。
当对图像中的某个像素进行卷积时，会把卷积核的中心放置到该像素上。
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/10/QQ截图20191026153757.png)
使用卷积计算可以实现很多常见图像处理效果，例如图像模糊，边缘检测等。
## 常见的边缘检测算子
梯度概念：如果相邻像素之间存在差别明显的颜色，亮度，纹理等属性，我们就会认为他们之间应该有一条边界。这种相邻像素之间的差值可以用`梯度`表示。
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/10/QQ截图20191026154413.png)
他们都包含了两个方向的卷积核，分别用于检测水平方向和竖直方向上的边缘信息。
在进行边缘检测时，我们需要对每个像素分别进行一次卷积计算，得到两个方向上的梯度值$G_x$和$G_y$，整体的梯度可以由以下公式求得：
$$G\;=\;\sqrt{G_x^2\;+G_y^2}$$
由于包含了开根号操作，为了节省性能，有时会用绝对值操作来代替开根号操作：
$$G\;=\;\left|G_x\right|\;+\;\left|G_y\right|$$
得到梯度G之后，就可以据此来判断哪些像素对应了边缘（梯度值越大，越有可能是边缘点）
## 实现
使用Sobel算子进行边缘检测，实现描边效果。
```csharp
using UnityEngine;
using System.Collections;

public class EdgeDetection : PostEffectsBase
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
    /// 调整边缘线强度，为1时，只显示边缘，不显示原渲染图像
    /// </summary>
    [Range(0.0f, 1.0f)] public float edgesOnly = 0.0f;

    /// <summary>
    /// 描边颜色
    /// </summary>
    public Color edgeColor = Color.black;

    /// <summary>
    /// 背景颜色
    /// </summary>
    public Color backgroundColor = Color.white;

    void OnRenderImage(RenderTexture src, RenderTexture dest)
    {
        if (material != null)
        {
            material.SetFloat("_EdgeOnly", edgesOnly);
            material.SetColor("_EdgeColor", edgeColor);
            material.SetColor("_BackgroundColor", backgroundColor);

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

Shader "Unity Shaders Book/Chapter 12/Edge Detection" 
{
	Properties 
	{
		_MainTex ("Base (RGB)", 2D) = "white" {}
		_EdgeOnly ("Edge Only", Float) = 1.0
		_EdgeColor ("Edge Color", Color) = (0, 0, 0, 1)
		_BackgroundColor ("Background Color", Color) = (1, 1, 1, 1)
	}
	SubShader 
	{
		Pass 
		{  
		    //深度测试总是通过
		    //关闭剔除
		    //关闭深度写入
			ZTest Always    Cull Off    ZWrite Off
			
			CGPROGRAM
			
			#include "UnityCG.cginc"
			
			#pragma vertex vert  
			#pragma fragment fragSobel
			
			sampler2D _MainTex;  
			//Unity提供的访问xxx纹理对应的每个纹素的大小
			uniform half4 _MainTex_TexelSize;
			fixed _EdgeOnly;
			fixed4 _EdgeColor;
			fixed4 _BackgroundColor;
			
			struct v2f 
			{
				float4 pos : SV_POSITION;
				//对应使用Sobel算子采样时需要的9个临域纹理坐标
				half2 uv[9] : TEXCOORD0;
			};
			  
			v2f vert(appdata_img v) 
			{
				v2f o;
				o.pos = UnityObjectToClipPos(v.vertex);
				
				half2 uv = v.texcoord;
				
				o.uv[0] = uv + _MainTex_TexelSize.xy * half2(-1, -1);
				o.uv[1] = uv + _MainTex_TexelSize.xy * half2(0, -1);
				o.uv[2] = uv + _MainTex_TexelSize.xy * half2(1, -1);
				o.uv[3] = uv + _MainTex_TexelSize.xy * half2(-1, 0);
				o.uv[4] = uv + _MainTex_TexelSize.xy * half2(0, 0);
				o.uv[5] = uv + _MainTex_TexelSize.xy * half2(1, 0);
				o.uv[6] = uv + _MainTex_TexelSize.xy * half2(-1, 1);
				o.uv[7] = uv + _MainTex_TexelSize.xy * half2(0, 1);
				o.uv[8] = uv + _MainTex_TexelSize.xy * half2(1, 1);
						 
				return o;
			}
			
			fixed luminance(fixed4 color) 
			{
				return  0.2125 * color.r + 0.7154 * color.g + 0.0721 * color.b; 
			}
			
			half Sobel(v2f i) 
			{
			    //定义水平方向卷积核Gx
				const half Gx[9] = {-1,  0,  1,
									-2,  0,  2,
									-1,  0,  1};
			    //定义水平方向卷积核Gy
				const half Gy[9] = {-1, -2, -1,
									0,  0,  0,
									1,  2,  1};		
				
				half texColor;
				half edgeX = 0;
				half edgeY = 0;
				for (int it = 0; it < 9; it++) {
				    //像素采样，计算亮度值
					texColor = luminance(tex2D(_MainTex, i.uv[it]));
					//卷积核计算叠加
					edgeX += texColor * Gx[it];
					edgeY += texColor * Gy[it];
				}
				//值越小，表明该位置越有可能是一个边缘点
				half edge = 1 - abs(edgeX) - abs(edgeY);
				
				return edge;
			}
			
			fixed4 fragSobel(v2f i) : SV_Target 
			{
			    //计算梯度值
				half edge = Sobel(i);
				//原图插值
				fixed4 withEdgeColor = lerp(_EdgeColor, tex2D(_MainTex, i.uv[4]), edge);
				//纯色插值
				fixed4 onlyEdgeColor = lerp(_EdgeColor, _BackgroundColor, edge);
				//最终插值
				return lerp(withEdgeColor, onlyEdgeColor, _EdgeOnly);
 			}
			
			ENDCG
		} 
	}
	FallBack Off
}

```
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/10/QQ截图20191026172829.png)
# 高斯模糊
模糊的实现有很多种方法，例如`均值模糊`和`中值模糊`（都使用了卷积核）。
`均值模糊同样使用了卷积操作`，它使用的卷积核中的各个元素值都相等，且相加等于1。也就是说，卷积后得到的像素值是其邻域内各个像素值得平均值。
中值模糊则是选择邻域内所有像素排序后的中值替换掉原颜色。
更高级的就是高斯模糊了。
## 高斯滤波
高斯模糊同样使用了卷积计算，它使用的卷积核名为`高斯核`。高斯核是一个正方形大小的滤波核，其中每个元素的计算都是基于下面的高斯方程：
$$G_{\left(x,y\right)}=\frac1{2\pi\sigma^2}e^{-\frac{x^2+y^2}{2\sigma^2}}$$
$\sigma$是标准方差（一般取值为1），x和y分别对应当前位置到卷积核中心的整数距离。
为了保证滤波后的图像不会变暗，我们需要对高斯核中的权重进行归一化——让每个权重除以所有权重的和，可以保证所有权重和为1。
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/10/QQ截图20191026175143.png)
高斯方程很好的模拟了邻域每个像素对当前处理像素的影响程度——距离越近，影响越大。
高斯核的维数越高，模糊程度越大。对应的采样次数越多。可以把二维的高斯函数拆分成两个一维函数。
对于一个大小为5的一维高斯核，我们实际只需要记录三个权重值即可。
## 实现
```csharp
using UnityEngine;
using System.Collections;

public class GaussianBlur : PostEffectsBase
{
    public Shader gaussianBlurShader;
    private Material gaussianBlurMaterial = null;

    public Material material
    {
        get
        {
            gaussianBlurMaterial = CheckShaderAndCreateMaterial(gaussianBlurShader, gaussianBlurMaterial);
            return gaussianBlurMaterial;
        }
    }

    /// <summary>
    /// 高斯模糊迭代次数
    /// </summary>
    [Range(0, 4)] public int iterations = 3;

    /// <summary>
    /// 模糊范围，在高斯核维数不变的情况下，blurSpread越大，模糊程度越高，但采样数不会被影响
    /// 但过大的blurSpread会造成虚影
    /// </summary>
    [Range(0.2f, 3.0f)] public float blurSpread = 0.6f;

    /// <summary>
    /// 缩放系数，downSample越大，需要处理的像素数越少，同时也能进一步提高模糊程度
    /// 但过大的downSample可能会使图像像素化
    /// </summary>
    [Range(1, 8)] public int downSample = 2;
    
    void OnRenderImage(RenderTexture src, RenderTexture dest)
    {
        if (material != null)
        {
            int rtW = src.width / downSample;
            int rtH = src.height / downSample;
            
            //存储src中图像缩放后的纹理
            RenderTexture buffer0 = RenderTexture.GetTemporary(rtW, rtH, 0);
            //设置双线性滤波
            buffer0.filterMode = FilterMode.Bilinear;

            Graphics.Blit(src, buffer0);

            for (int i = 0; i < iterations; i++)
            {
                material.SetFloat("_BlurSize", 1.0f + i * blurSpread);
                
                RenderTexture buffer1 = RenderTexture.GetTemporary(rtW, rtH, 0);

                //使用垂直Pass处理
                Graphics.Blit(buffer0, buffer1, material, 0);

                RenderTexture.ReleaseTemporary(buffer0);
                buffer0 = buffer1;
                buffer1 = RenderTexture.GetTemporary(rtW, rtH, 0);

                //使用水平Pass处理
                Graphics.Blit(buffer0, buffer1, material, 1);

                RenderTexture.ReleaseTemporary(buffer0);
                buffer0 = buffer1;
            }
            //直接显示到屏幕上
            Graphics.Blit(buffer0, dest);
            RenderTexture.ReleaseTemporary(buffer0);
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

Shader "Unity Shaders Book/Chapter 12/Gaussian Blur" 
{
	Properties 
	{
		_MainTex ("Base (RGB)", 2D) = "white" {}
		_BlurSize ("Blur Size", Float) = 1.0
	}
	SubShader 
	{
	    //相当于C++中的头文件，可用于复用函数
		CGINCLUDE
		
		#include "UnityCG.cginc"
		
		sampler2D _MainTex;  
		half4 _MainTex_TexelSize;
		float _BlurSize;
		  
		struct v2f 
		{
			float4 pos : SV_POSITION;
			half2 uv[5]: TEXCOORD0;
		};
		
		//通过计算采样纹理坐标的代码从片元着色器中转移到顶点着色器中，可以减少运算，提高性能
		//由于从顶点着色器到片元着色器的插值是线性的，因此这样的转移不会影响纹理坐标的计算结果
		v2f vertBlurVertical(appdata_img v) 
		{
			v2f o;
			o.pos = UnityObjectToClipPos(v.vertex);
			
			half2 uv = v.texcoord;
			
			o.uv[0] = uv;
			//与_BlurSize相乘控制采样距离
			o.uv[1] = uv + float2(0.0, _MainTex_TexelSize.y * 1.0) * _BlurSize;
			o.uv[2] = uv - float2(0.0, _MainTex_TexelSize.y * 1.0) * _BlurSize;
			o.uv[3] = uv + float2(0.0, _MainTex_TexelSize.y * 2.0) * _BlurSize;
			o.uv[4] = uv - float2(0.0, _MainTex_TexelSize.y * 2.0) * _BlurSize;
					 
			return o;
		}
		
		v2f vertBlurHorizontal(appdata_img v) 
		{
			v2f o;
			o.pos = UnityObjectToClipPos(v.vertex);
			
			half2 uv = v.texcoord;
			
			o.uv[0] = uv;
			o.uv[1] = uv + float2(_MainTex_TexelSize.x * 1.0, 0.0) * _BlurSize;
			o.uv[2] = uv - float2(_MainTex_TexelSize.x * 1.0, 0.0) * _BlurSize;
			o.uv[3] = uv + float2(_MainTex_TexelSize.x * 2.0, 0.0) * _BlurSize;
			o.uv[4] = uv - float2(_MainTex_TexelSize.x * 2.0, 0.0) * _BlurSize;
					 
			return o;
		}
		
		fixed4 fragBlur(v2f i) : SV_Target 
		{
			float weight[3] = {0.4026, 0.2442, 0.0545};
			
			fixed3 sum = tex2D(_MainTex, i.uv[0]).rgb * weight[0];
			
			//根据对称性，进行两次迭代
			for (int it = 1; it < 3; it++) 
			{
				sum += tex2D(_MainTex, i.uv[it*2-1]).rgb * weight[it];
				sum += tex2D(_MainTex, i.uv[it*2]).rgb * weight[it];
			}
			 
			return fixed4(sum, 1.0);
		}
		    
		ENDCG
		
		ZTest Always Cull Off ZWrite Off
		
		Pass 
		{
			NAME "GAUSSIAN_BLUR_VERTICAL"
			
			CGPROGRAM
			  
			#pragma vertex vertBlurVertical  
			#pragma fragment fragBlur
			  
			ENDCG  
		}
		
		Pass 
		{  
			NAME "GAUSSIAN_BLUR_HORIZONTAL"
			
			CGPROGRAM  
			
			#pragma vertex vertBlurHorizontal  
			#pragma fragment fragBlur
			
			ENDCG
		}
	} 
	FallBack "Diffuse"
}

```
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/10/QQ截图20191026204645.png)
# Bloom效果
`Bloom让画面中较亮的区域扩散到周围的区域中，造成一种朦胧的效果`。
Bloom实现原理：根据一个阈值提取出图像中的较亮区域，把他们存储在一张渲染纹理中，再利用高斯模糊对这张渲染纹理进行模糊处理，模拟光线扩散的效果，最后再将其和原图像进行混合，得到最终的效果。
```C#
using UnityEngine;
using System.Collections;

public class Bloom : PostEffectsBase
{
    public Shader bloomShader;
    private Material bloomMaterial = null;

    public Material material
    {
        get
        {
            bloomMaterial = CheckShaderAndCreateMaterial(bloomShader, bloomMaterial);
            return bloomMaterial;
        }
    }
    
    [Range(0, 4)] public int iterations = 3;
    
    [Range(0.2f, 3.0f)] public float blurSpread = 0.6f;

    [Range(1, 8)] public int downSample = 2;

    /// <summary>
    /// 亮度阈值，一般情况下亮度不会超过1
    /// 如果开启HDR，硬件会允许我们把颜色值存储在一个更高精度范围的缓冲中
    /// 此时像素亮度值可能会超过1
    /// </summary>
    [Range(0.0f, 4.0f)] public float luminanceThreshold = 0.6f;

    void OnRenderImage(RenderTexture src, RenderTexture dest)
    {
        if (material != null)
        {
            material.SetFloat("_LuminanceThreshold", luminanceThreshold);

            int rtW = src.width / downSample;
            int rtH = src.height / downSample;

            RenderTexture buffer0 = RenderTexture.GetTemporary(rtW, rtH, 0);
            buffer0.filterMode = FilterMode.Bilinear;

            //使用第一个Pass提取图像中较亮部分，存储在buffer0中
            Graphics.Blit(src, buffer0, material, 0);

            for (int i = 0; i < iterations; i++)
            {
                material.SetFloat("_BlurSize", 1.0f + i * blurSpread);

                RenderTexture buffer1 = RenderTexture.GetTemporary(rtW, rtH, 0);
                
                Graphics.Blit(buffer0, buffer1, material, 1);

                RenderTexture.ReleaseTemporary(buffer0);
                buffer0 = buffer1;
                buffer1 = RenderTexture.GetTemporary(rtW, rtH, 0);

                Graphics.Blit(buffer0, buffer1, material, 2);

                RenderTexture.ReleaseTemporary(buffer0);
                buffer0 = buffer1;
            }

            material.SetTexture("_Bloom", buffer0);
            Graphics.Blit(src, dest, material, 3);

            RenderTexture.ReleaseTemporary(buffer0);
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

Shader "Unity Shaders Book/Chapter 12/Bloom" 
{
	Properties 
	{
		_MainTex ("Base (RGB)", 2D) = "white" {}
		_Bloom ("Bloom (RGB)", 2D) = "black" {}
		_LuminanceThreshold ("Luminance Threshold", Float) = 0.5
		_BlurSize ("Blur Size", Float) = 1.0
	}
	SubShader 
	{
		CGINCLUDE
		
		#include "UnityCG.cginc"
		
		sampler2D _MainTex;
		half4 _MainTex_TexelSize;
		sampler2D _Bloom;
		float _LuminanceThreshold;
		float _BlurSize;
		
		struct v2f 
		{
			float4 pos : SV_POSITION; 
			half2 uv : TEXCOORD0;
		};	
		
		v2f vertExtractBright(appdata_img v) 
		{
			v2f o;
			
			o.pos = UnityObjectToClipPos(v.vertex);
			
			o.uv = v.texcoord;
					 
			return o;
		}
		
		fixed luminance(fixed4 color) 
		{
			return  0.2125 * color.r + 0.7154 * color.g + 0.0721 * color.b; 
		}
		
		fixed4 fragExtractBright(v2f i) : SV_Target 
		{
			fixed4 c = tex2D(_MainTex, i.uv);
			//亮度减去阈值，控制在0,1之间
			fixed val = clamp(luminance(c) - _LuminanceThreshold, 0.0, 1.0);
			//与原像素相乘，得到提取后的亮部区域
			return c * val;
		}
		
		struct v2fBloom 
		{
			float4 pos : SV_POSITION; 
			half4 uv : TEXCOORD0;
		};
		
		v2fBloom vertBloom(appdata_img v) 
		{
			v2fBloom o;
			
			o.pos = UnityObjectToClipPos (v.vertex);
			//初始化纹理坐标
			o.uv.xy = v.texcoord;		
			o.uv.zw = v.texcoord;
			
			#if UNITY_UV_STARTS_AT_TOP			
			if (_MainTex_TexelSize.y < 0.0)
				o.uv.w = 1.0 - o.uv.w;
			#endif
				        	
			return o; 
		}
		
		fixed4 fragBloom(v2fBloom i) : SV_Target 
		{
			return tex2D(_MainTex, i.uv.xy) + tex2D(_Bloom, i.uv.zw);
		} 
		
		ENDCG
		
		ZTest Always Cull Off ZWrite Off
		
		Pass 
		{  
			CGPROGRAM  
			#pragma vertex vertExtractBright  
			#pragma fragment fragExtractBright  
			
			ENDCG  
		}
		
		//引用之前的高斯模糊Pass
		UsePass "Unity Shaders Book/Chapter 12/Gaussian Blur/GAUSSIAN_BLUR_VERTICAL"
		
		UsePass "Unity Shaders Book/Chapter 12/Gaussian Blur/GAUSSIAN_BLUR_HORIZONTAL"
		
		Pass 
		{  
			CGPROGRAM  
			#pragma vertex vertBloom  
			#pragma fragment fragBloom  
			
			ENDCG  
		}
	}
	FallBack Off
}

```
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/10/QQ截图20191026211908.png)
# 运动模糊
不用多嗦，直接上图。（狂野飙车9——竞速传奇）
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/10/QQ图片20191026212609.jpg)
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/10/QQ图片20191026212601.jpg)
运动模糊的一种实现方式是利用一块累积缓存（accumulation buffer）来混合多张连续图像，当物体快速移动产生多张图像后，我们取他们之间的平均值作为最后的运动模糊图像，但是这种方式消耗很大，因为想要获取多张帧图像往往意味着我们需要在同一帧里渲染多次场景。
另一种实现方式是创建和使用速度缓存（velocity buffer），这个缓存中存储了各个像素当前的运动速度，然后利用该值来决定模糊的方向和大小。
我们这里使用第一种实现方式
```csharp
using UnityEngine;
using System.Collections;

public class MotionBlur : PostEffectsBase
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
    /// 值越大，运动拖尾的效果越明显，为了防止拖尾效果完全替代当前帧渲染结果，把他的值截取在0.0~0.9
    /// </summary>
    [Range(0.0f, 0.9f)] public float blurAmount = 0.5f;

    /// <summary>
    /// 保存当前渲染叠加结果
    /// </summary>
    private RenderTexture accumulationTexture;

    void OnDisable()
    {
        DestroyImmediate(accumulationTexture);
    }

    /// <summary>
    /// 抓取屏幕图像
    /// </summary>
    /// <param name="src">Unity会把当前渲染的得到的图像存储在第一个参数对应的源渲染纹理中</param>
    /// <param name="dest">目标渲染纹理</param>
    [ImageEffectOpaque]
    void OnRenderImage(RenderTexture src, RenderTexture dest)
    {
        if (material != null)
        {
            if (accumulationTexture == null || accumulationTexture.width != src.width ||
                accumulationTexture.height != src.height)
            {
                DestroyImmediate(accumulationTexture);
                accumulationTexture = new RenderTexture(src.width, src.height, 0);
                accumulationTexture.hideFlags = HideFlags.HideAndDontSave;
                //使用当前帧渲染结果初始化accumulationTexture
                Graphics.Blit(src, accumulationTexture);
            }

            //表明我们需要进行一个渲染纹理的恢复操作
            //使用这个函数明确告诉Unity我知道我在做什么，不用担心
            //恢复操作发生在渲染到纹理而该纹理又没有被提前清空或销毁的情况下
            //作用是保存上一帧渲染结果
            accumulationTexture.MarkRestoreExpected();

            material.SetFloat("_BlurAmount", 1.0f - blurAmount);
            //把当前屏幕图像src叠加到accumulationTexture中
            Graphics.Blit(src, accumulationTexture, material);
            //把结果显示到屏幕上
            Graphics.Blit(accumulationTexture, dest);
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

Shader "Unity Shaders Book/Chapter 12/Motion Blur" 
{
	Properties 
	{
		_MainTex ("Base (RGB)", 2D) = "white" {}
		_BlurAmount ("Blur Amount", Float) = 1.0
	}
	SubShader 
	{
		CGINCLUDE
		
		#include "UnityCG.cginc"
		
		sampler2D _MainTex;
		fixed _BlurAmount;
		
		struct v2f 
		{
			float4 pos : SV_POSITION;
			half2 uv : TEXCOORD0;
		};
		
		v2f vert(appdata_img v) 
		{
			v2f o;
			
			o.pos = UnityObjectToClipPos(v.vertex);
			
			o.uv = v.texcoord;
					 
			return o;
		}
		
		fixed4 fragRGB (v2f i) : SV_Target 
		{
			return fixed4(tex2D(_MainTex, i.uv).rgb, _BlurAmount);
		}
		
		half4 fragA (v2f i) : SV_Target 
		{
			return tex2D(_MainTex, i.uv);
		}
		
		ENDCG
		
		ZTest Always Cull Off ZWrite Off
		
		Pass 
		{
			Blend SrcAlpha OneMinusSrcAlpha
			ColorMask RGB
			
			CGPROGRAM
			
			#pragma vertex vert  
			#pragma fragment fragRGB  
			
			ENDCG
		}
		
		Pass 
		{   
			Blend One Zero
			ColorMask A
			   	
			CGPROGRAM  
			
			#pragma vertex vert  
			#pragma fragment fragA
			  
			ENDCG
		}
	}
 	FallBack Off
}

```
