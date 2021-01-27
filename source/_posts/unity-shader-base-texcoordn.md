---
title: Unity Shader基础篇：浅谈TEXCOORDn
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
cover: https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/10/QQ截图20191031135634.png
aplayer:
---
<meta name="referrer" content="no-referrer" />

## 事情的起因
今早起床，发现自己在知乎被邀请回答一个问题：（应该是这阵子逛的图形学区域比较多）
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/10/QQ截图20191031131639.png)
## TEXCOORD到底是个什么东西
我们来看看官方文档怎么说

- TEXCOORD0 is the first UV coordinate, typically float2, float3 or float4.
- TEXCOORD1, TEXCOORD2 and TEXCOORD3 are the 2nd, 3rd and 4th UV coordinates, respectively.

看上去很简单，`TEXCOORD是指纹理坐标，float2, float3, float4类型。n是指第几组纹理坐标。`
## 第几组？？？能有几组uv？？？
身为对美术一无所知的逻辑仔，我是不太明白的，在网上也没有找到好的答案，好在咨询了很多大佬，在此整理一下。

- 模型中每个顶点保存有uv，可能有一套或者几套，这些uv是指三维模型在2D平面的展开，跟纹理对应上进行插值采样就看到三维里的纹理颜色了
- [https://kumokyaku.github.io/2019/07/14/UNITY%E7%94%9F%E6%88%90LightmapUVs/](https://kumokyaku.github.io/2019/07/14/UNITY%E7%94%9F%E6%88%90LightmapUVs/)
- 这张Blender的图很好展示了多套纹理坐标到底是个什么东西（来自上面的链接）![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/10/QQ截图20191031135634.png)
- 简单来说texcoord就是存在顶点里的一组数据，我们可以通过这组数据在渲染的时候进行贴图采样，比如我们常用的`第一套uv作为基础纹理，通常基础纹理我们可以根据需求进行一些区域的uv重用（比如左右脸贴图一样，可以映射到统一贴图区域），第二套uv经常用于光照贴图，光照贴图要求是uv不可以重复，所以通常不能用第一套uv，第三套uv用于更加奇特的需求，以此类推。。。`
- texcoord应该是更加标准的名称，不过因为这个坐标系里面用uvw作为三个轴名称，所以美术那边普遍称作uv
