---
title: 解决血条在透视相机下的偏移
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

## 起因

我在为Moba项目开发人物头顶血条同步逻辑时，出现了一些问题。
Moba项目链接：<a href='https://gitee.com/NKG_admin/NKGMobaBasedOnET/stargazers'><img src='https://gitee.com/NKG_admin/NKGMobaBasedOnET/badge/star.svg?theme=dark' alt='star'></img></a>
### 前提条件
UI工具：FGUI
Unity版本：2018.4

自己计算同步血条的位置，这样相比于直接在人物身上挂载Canvas做血条显示的优点是不必考虑远大近小的情况（参考LOL中戏命师烬开大，咖喱奥开大跃起时的屏幕缩放而血条大小不变），并且更便于管理和维护。
### 问题描述
当人物与当前屏幕中心点不重合时，头顶血条会出现偏移的情况。
当人物基本上都在中央时
![当人物基本上都在中央时](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2020/01/QQ截图20200123130006.png)
当偏离屏幕中央过远时（右边人物为屏幕中央）
![当偏离屏幕中央过远时（右边人物为屏幕中央）](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2020/01/QQ截图20200123130038.png)

### 参照目标
要解决问题很多时候需要一个参照模型，这里就拿LOL来举例。可以看到即使圣枪和最右边那一坨辨识度极低的英雄他们的血条都没有发生偏移。这里不理他是具体怎么实现的，只需要朝这个结果靠拢就行了。
![参照模型](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2020/01/QQ截图20200123131543.png)

### 问题定位
发现了问题就应该去找问题的根源。
遥想当年，自己写一个用手拖动2D小飞机的功能都觉得自己要起飞了（老飞行员了），回看今天，怎么2D变成3D的就拉了胯呢。
有一个很明显的不同点，2D的使用的是正交相机，3D的使用的是透视相机。
然后就有以下几个重要的信息点：

- Moba游戏的相机一般而言都是有一定倾斜度的（具体倾斜多少看个人），这才会让我们有斜俯视的感觉，而就是这个倾斜度导致了偏移
- 正常来说在正交相机下2D UI大小不会根据距离摄像机远近而变化，然而我们透视相机下的3D模型却会远大近小。
- 那么，是不是以上两点导致了血条偏移问题呢？

### 实验排除
首先分析第一种可能：是不是因为倾斜度而导致了UI偏差
我们知道，在渲染流水线中，一个顶点要从模型空间到屏幕空间，需要进行许多转换
$$\mathrm{模型空间}\xrightarrow{\mathrm{模型变换}}\mathrm{世界空间}\xrightarrow{\mathrm{观察变换}}\\\\\\\mathrm{观察空间}\xrightarrow{\mathrm{投影变换}}\mathrm{齐次裁剪空间}\xrightarrow{\mathrm{齐次除法}+\mathrm{屏幕映射}}\mathrm{屏幕空间}$$
经过分析，可以得知正交相机和透视相机在这个过程中只有一个不一样的地方，即投影变换。
透视相机的投影矩阵:
$$\begin{bmatrix}x\frac{\cot\left({\displaystyle\frac{FOV}2}\right)}{Aspect}\\y\cot\left(\frac{FOV}2\right)\\-z\frac{Far+Near}{Far-Near}-2\cdot\frac{Near\cdot Far}{Far-Near}\\-z\end{bmatrix}$$
正交相机的投影矩阵:
$$\begin{bmatrix}\frac x{Aspect\cdot Size}\\\frac y{Size}\\-\frac{2z}{Far-Near}-\frac{Far+Near}{Far-Near}\\1\end{bmatrix}$$

其中FOV为相机视角范围(Field Of View)，Aspect为横纵比，Far和Near分别为摄像机远裁剪面和近裁剪面距离摄像机的距离，Size则为正交相机中竖直方向视角大小一半。
发现并没有由`角度`所影响的参数，所以这种可能性`Pass`。

然后分析第二种可能：难道自己求出的坐标是正确的，但是由于透视相机的视觉效果（远大近小）而导致我们的UI看起来偏移了？
下面是一个测试：
左上角是测试人物血条，旁边的是在FGUI工程中的血条原型（FGUI的组件默认以左上角为原点），可以看出坐标基本上是吻合的
![测试图片](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2020/01/QQ截图20200123162830.png)

### 解决方案
得知了误差原因，就好解决了，这里是因为视觉原因，所以我们只需要在上层更改一下位置同步逻辑就行了。
思路就是与屏幕中心偏差越大，因数就越大。
其中的100为血条UI宽度的一半左右，这样就可以居中显示。180为一个高度设置，这个基本上不用动。
X偏移
```csharp
/// <summary>
/// 得到偏移的x
/// </summary>
/// <param name="barPos">血条的屏幕坐标</param>
/// <returns></returns>
private float GetOffsetX(Vector2 barPos)
{
	float final = 100 + (Screen.width / 2.0f - barPos.x) * 0.05f;
	return final;
}
```
Y偏移
```csharp
this.m_HeadBar.GObject.y -= 180;
```
### 结果
正确的结果应该是（右边人物为屏幕中央）
![正确结果](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2020/01/QQ截图20200123130210.png)

### 总结
用一个试了很多次的经验模型解决了这个问题，虽然没有想象中高端，但是好歹大致实现了效果。
