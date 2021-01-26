---
title: Unity Shader入门精要学习笔记：基于物理的渲染
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

# 引言
随着计算机的处理能力越来越强，人们开始考虑使用更加复杂的算法来渲染更加真实的画面。 在二十世纪八十年代左右， `基于物理的渲染技术（Physically Based Shading, PBS）` 首次被引入图形学的正统研究中，学者们提出了使用光线追踪的方法来渲染全局光照，由此打开了精确渲染光线传播的大门。
在实时渲染领域， 人们也发现了这种基于物理的光照模型的巨大优势。 在这之前， Lambert光照模型、 Phong 光照模型和 Blinn-Phong 光照模型等经验模型占据了主流。 然而，这种不满足能量守恒的光照模型使得美术人员需要花费大量的时间在参数调节上。 尤其是， 美术人员往往好不容易为一个物体调节好了所有参数，使得它在当前的光照条件下看起来是满意的。然而， 一旦光照环境发生了变化， 这一切都得从头再来。 因此， 近年来游戏从业者开始着手把基于物理的光照模型应用于实时渲染中。
# PBS的理论和数学基础
## 光是什么
在物理学中，光是一种电磁波。首先，光由太阳或其他光源中被发射出来，然后与场景中的对象相交，一些光线被吸收（absorption），而另一些则被散射（scattering），最后光线被一个感应器（例如我们的眼睛）吸收成像。
光线会被吸收是由于光被转化成了其他能量，但吸收并不会改变光的传播方向。相反的，散射则不会改变光的能量，但会改变它的传播方向。在光的传播过程中，影响光的一个重要的特性是材质的折射率（refractive index）。我们知道，在均匀的介质中，光是沿直线传播的。但如果光在传播时介质的折射率发生了变化，光的传播方向就会发生变化。特别是，如果折射率是突变的，就会发生光的散射现象。
为了在渲染中对光照进行建模，我们往往只考虑一种特殊情况，即只考虑两个介质的边界是无限大并且是光学平滑（optically flat）的。尽管真实物体的表面并不是无限延伸的，也不是绝对光滑的，但和光的波长相比，它们的大小可以被近似认为是无限大以及光学平滑的。在这样的前ᨀ下，光在不同介质的边界会被分割成两个方向：反射方向和折射方向。而有多少百分比的光会被反射（另一部分就是被折射了）则是由菲涅耳等式（Fresnel equations） 来描述的。
[![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/12/QQ截图20191220111434.png)](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/12/QQ截图20191220111434.png)
尽管相对于光的波长来说，它们的确可以被认为是光学平坦的。但是，如果想象我们有一个高倍放大镜，去放大这些被照亮的物体表面，就会发现有很多之前肉眼不可见的凹凸不平的平面。在这种情况下，物体的表面和光照发生的各种行为，更像是一系列微小的光学平滑平面和光交互的结果，其中每个小平面会把光分割成不同的方向。这种建立在微表面的模型更容易解释为什么有些物体看起来粗糙，而有些看起来就平滑。
金属材质具有很高的吸收系数，因此， 所有被折射的光往往会被立刻吸收，被金属内部的自由电子转化成其他形式的能量。而非金属材质则会同时表现出吸收和散射两种现象，这些被散射出去的光又被称为次表面散射光`（subsurface-scattered light）`。
微表面反射的光可以被认为是该点上一些方向变化不大的反射光，如图 18.4 中的黄线所示（可参考随书资源中的彩图）。而折射光线（蓝线）则需要更多的考虑。那些次表面散射光会从不同于入射点的位置从物体内部再次射出，如图 18.4 左图所示。而这些离入射点的距离值和像素大小之间的关系会产生两种建模结果。如果像素要大于这些散射距离的话，意味着这些次表面散射产生的距离可以被忽略，那我们的渲染就可以在局部进行，如图 18.4 右图所示。如果像素要小于这些散射距离，我们就不可以选择忽略它们了，要实现更真实的次表面散射效果，我们需要使用特殊的渲染模型，也就是所谓的`次表面散射渲染技术`。
[![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/12/QQ截图20191220111948.png)](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/12/QQ截图20191220111948.png)
## 渲染方程
我们可以用`辐射率`（radiance） 来量化光。辐射率是单位面积、单位方向上光源的辐射通量，通常用$L$来表示，被认为是对单一光线的亮度和颜色评估。在渲染中，我们通常会使用入射光线的入射辐射率$L_i$来计算出射辐射率$L_o$，这个过程也往往被称为是着色（shading） 过程。
$$L_o(\nu)\;=L_e(\nu)+\int_\Omega f(\omega_{\mathcal i},\nu)L_{\mathcal i}(\omega_{\mathcal i})(\mathcal n\cdot\omega_{\mathcal i})d\omega_{\mathcal i}$$
[![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/12/QQ截图20191221090146.png)](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/12/QQ截图20191221090146.png)
渲染方程是计算机图形学的核心公式，当去掉自发光项$L_e(\nu)$后，剩余的部分就是著名的`反射等式`：我们现在要计算表面上某
点的出射辐射率，我们已知到该点的观察方向，该点的出射辐射率是由从许多不同方向的入射辐射率叠加后的结果。其中， $f(ω_i, v)$表示了不同方向的入射光在该观察方向上的权重分布。我们把这些不同方向的光辐射率（$L_i(ω_i)$部分）乘以观察方向上所占的权重（$f(ω_i, v)$部分），再乘以它们在该表面的投影结果（$(n·ω_i)$部分），最后再把这些值加起来（即做积分）就是最后的出射辐射率。
在实时渲染中，自发光项通常就是直接加上某个自发光值。除此之外，积分累加部分在实时渲染中也基本无法实现， 因此积分部分通常会被若干精确光源的叠加所代替， 而不需要计算所有入射光线在半球面上的积分。
## 精确光源
在真实的物理世界中，所有的光源都是有面积概念的，即所谓的`面光源`。由于面光源的光照计算通常要耗费大量的时间，因此在实时渲染中， 我们通常会使用`精确光源`（punctual light sources）来近似模拟这些面光源。 图形学中常见的精确光源类型有点光源、平行光和聚光灯等， 这些精确光源被认为是大小为无限小且方向确定的， 尽管这并不符合真实的物理定义，但它们在大多数情况下都能得到令人满意的渲染效果。我们使用 $l_c$ 来表示它的方向，使用 $c_{light}$ 表示它的颜色。使用精确光源的最大的好处在于，我们可以大大简化上面的反射等式。 即对于一个精确光源，我们可以使用下面的等式来计算它在某个观察方向 v 上的出射辐射率.
$$L_o(\nu)\;=\;\pi f(l_c,v)c_{light}(n\cdot l_c)$$
使用一个特定方向的$f(l_c,v)$值来代替积分操作，简化了运算。如果场景包含多个精确光源，可以把他们分别代入上面式子计算，然后结果相加。即
$$L_o(\nu)\;=\;\sum_{i=0}^nL_O^i(v)\;=\;\sum_{i=0}^n\mathrm{πf}(\mathrm l_{\mathrm c}^{\mathrm i},\mathrm v){\mathrm c}_{\mathrm{light}}(\mathrm n\cdot\mathrm l_{\mathrm c}^{\mathrm i})\\$$
$f(l_c,v)$实际上描述了当前点是如何与入射光线进行交互的，当给定某个入射方向的入射光后，有多少百分比的光照被反射到了观察方向上。即双向反射分布函数（BRDF）。
## 双向反射分布函数（BRDF）
BRDF定量描述了物体表面一点是如何和光进行交互的，大多数情况下，BRDF可以用$f(l,v)$来表示，l为入射方向，v为观察方向。绕着表面法线旋转入射方向或观察方向并不会影响 BRDF 的结果，这种 BRDF 被称为是`各项同性`（isotropic） 的 BRDF。与之对应的则是`各向异性`（anisotropic） 的 BRDF。
BRDF有两种理解方式
1. 当给定入射角度后，BRDF可以给出所有出射方向上的反射和散射光线的相对分布情况
2. 当给定观察方向（即出射方向）后，BRDF可以给出从所有入射方向到该出射方向的光线分布
更加直白的理解是，当一束光线沿着入射方向$l$到达表面某点时，$f(l,v)$表示有多少部分的能量被反射到了观察方向v上。
### BRDF是如何计算的
可以看出，BRDF决定了着色过程是否是基于物理的，这可以由BRDF是否满足两个特性来判断：`交换律`和`能量守恒`。
#### 交换律
交换律要求交换$l$和$v$的值后，BRDF的值不变
$f(l,v) = f(v,l)$
#### 能量守恒
能量守恒要求表面反射的能量不能超过入射的光能
$$\forall l,\int_\Omega f(l,v)(n\cdot l)d\omega_o\leq1\\$$
基于这些理论，BRDF可以用于描述两种不同的物理现象：表面反射和次表面反射。
针对每种现象， BRDF通常会包含一个单独的部分来描述它们—用于描述表
面反射的部分被称为`高光反射项`（specular term），以及用于描述次表面散射的`漫反射项`（diffuse term）
[![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/12/QQ截图20191221095408.png)](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/12/QQ截图20191221095408.png)
## 漫反射项
之前的Lambert模型就是最简单，也是应用最广泛的满发射BRDF，准确的Lambertian BRDF的表示为：
$$f_{Lambert}(l,v)\;=\;\frac{c_{diff}}{\mathrm\pi}\\$$
其中，$c_{diff}$表示漫反射光线所占的比例，它也通常被称为是漫反射颜色，上面的式子实际上是一个定值，我们常见到的余弦因子部分（即(n·l)）实际是反射等式的一部分，而不是 BRDF 的部分。上面的式子之所以要除以π，是因为我们假设漫反射在所有方向上的强度都是相同的，而 BRDF 要求在半球内的积分值为 1。因此，给定入射方向 l 的光源在表面某点的出射漫反射辐射率为
$$L_O(v)\;=\;{\mathrm{πf}}_{\mathrm{Lambert}}(\mathrm l,\mathrm v){\mathrm c}_{\mathrm{light}}(\mathrm n,\mathrm l)\;=\;{\mathrm c}_{\mathrm{diff}\;}\times{\mathrm c}_{\mathrm{light}}(\mathrm n,\mathrm l)\\$$
通过对真实材质的 BRDF 数据进行分析，研究人员发现许多材质在掠射角度表现出了明显的高光反射峰值，而且还与表面的粗糙度有着强烈的联系。粗糙表面在掠射角容易形成一条亮边，而相反地光滑表面则容易在掠射角形成一条阴影边。这些都是 Lambert 模型所无法描述的。下图显示了这样的例子，注意图中在掠射角的光照效果。
[![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/12/QQ截图20191221102238.png)](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/12/QQ截图20191221102238.png)
因此，许多基于物理的渲染选择使用更加复杂的漫反射项来模拟更加真实次表面散射的结果。例如，在 Disney BRDF中，它的漫反射项为
$$f_{diff}(l,v)\;=\;\frac{baseColor}{\mathrm\pi}(1+(F_{D90}-1){(1-n\cdot l)}^5)(1+(F_{D90}-1){(1-n\cdot v)}^5)\;$$
$$F_{D90}\;=\;0.5\;+\;2roughness{(h\cdot l)}^2\;$$
其中baseColor是表面颜色，通常由纹理采样得到，roughness是表面的粗糙度。
## 高光反射项
在基于物理的渲染中， BDRF 中的高光反射项大多数都是建立在微面元理论（microfacet theory） 的假设上的。微面元理论认为，物体表面实际是由许多人眼看不到的微面元组成的，虽然物体表面并不是光学平滑的，但这些微面元可以被认为是光学平滑的，也就是说它们具有完美的高光反射。当光线和物体表面一点相交时，实际上是和一系列微面元交互的结果。当光和这些微面元相交时，光线会被分割成两个方向—反射方向和折射方向。这里我们只需要考虑被反射的光线，而折射光线已经在之前的漫反射项中考虑过了。
假设表面法线为 n，这些微面元的法线 m 并不都等于 n，因此，不同的微面元会把同一入射方向的光线反射到不同的方向上。而当我们计算 BRDF 时，入射方向 l 和观察方向 v 都会被给定，这意味着只有一部分微面元反射的光线才会进入到我们的眼睛中，这部分微面元会恰好把光线反射到方向 v 上，即它们的法线 m 等于 l 和 v 的一半，也就是我们一直看到的半角度矢量 h（halfangle vector，也被称为 half vector） 如图 18.6（a） 所示。
然而，这些 m = h 的微面元反射也并不会全部添加到 BRDF 的计算中。这是因为，它们其中一部分会在入射方向 l 上被其他微面元挡住（shadowing），如图 18.6（b） 所示，或是在它们的反射方向 v 上被其他微面元挡住了（masking），如图 18.6（c） 所示。微面元理论认为，所有这些被遮挡住的微面元不会添加到高光反射项的计算中（实际上它们中的一些由于多次反射仍然会被我们看到，但这不在微面元理论的考虑范围内）。
[![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/12/QQ截图20191221103528.png)](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/12/QQ截图20191221103528.png)
基于微面片理论的这些假设，BRDF的高光反射项可以用下面的通用形式表示
$$f_{spec}(l,v)\;=\;\frac{F(l,h)G(l,v,h)D(h)}{4(n,l)(n,v)}\\$$
这就是著名的 `Torrance-Sparrow 微面元模型`（Torrance 和 Sparrow 来源于两个作者的姓名）。上面的式子看起来难以理解，实际上其中的各个项对应了我们之前讲到的不同现象。 D(h)是微面元的`法线分布函数`（normal distribution function， NDF），它用于计算有多少比例的微面元的法线满足 m=h，只有这部分微面元才会把光线从 l 方向反射到 v 上。 G(l, v, h)是`阴影— 遮掩函数`（shadowing-masking function），它用于计算那些满足 m=h 的微面元中有多少会由于遮挡而不会被人眼看到，因此它给出了活跃的微面元（active microfacets）所占的浓度，只有活跃的微面元才会成功地把光线反射到观察方向上。 F(l, h)则是这些活跃微面元的`菲涅尔反射`（Fresnel reflectance）函数，它可以告诉我们每个活跃的微面元会把多少入射光线反射到观察方向上，即表示了反射光线占入射光线的比率。事实上，现实生活中几乎所有的物体都会表现出菲涅耳现象。最后，分母4(n ⋅ l)(n ⋅ v)是用于`校正从微面元的局部空间到整体宏观表面数量差异的校正因子`。这些不同的部分又可以衍生出很多不同的 BRDF 模型。 首先是菲涅耳反射函数部分 F(l, h)
### 菲涅尔反射函数
菲涅耳反射函数计算了光学表面反射光线所占的部分，它表明了当光照方向和观察方向夹角逐渐增大时高光反射强度增大的现象。 完整的菲涅耳等式非常复杂，包含了诸如复杂的折射率等与材质相关的参数。为了给美术人员ᨀ供更加直观且方便调节的参数，大多数 PBS 实现选择使用Schlick 菲涅耳近似等式来得到近似的菲涅尔反射效果：
$$F_{Schlick}(l,h)\;=\;c_{spec}\;+(1-c_{spec})(1-{(l\cdot h))}^5\;$$
其中， $c_{spec}$ 是材质的高光反射颜色。 通过对真实世界材质的观察， 人们现金属材质的高光反射颜色值往往比较大，而非金属材质的反射颜色值则往往较小
### 法线分布函数
法线分布函数 D(h)表示了对于当前表面来说有多少比例的微面元的法线满足 m=h， 这意味着只有这些微面元才会把光线从 l 方向反射到 v 上。 对于大多数表面来说，微面元的法线朝向并不是均匀分布的， 更多的微面元会具有和表面法线 n 相同的面法线。 `法线分布函数的值必须是非负的标量值`，它决定了高光区域的大小、亮度和形状，因此是高光反射项中非常重要的一项。 一个直观的感受的， 当表面的粗糙度下降时， 应该有更多的微面元的面法线满足 m=n。，因此法线分布函数应该考虑到表面粗糙度的影响。
我们之前学习的 Blinn-Phong 模型就是一种非常简单的模型。 Blinn 他的论文中改进了Phong 模型并提出了 Blinn-Phong 模型，使它更贴合微面元 BRDF 模型的理论。 Blinn-Phong 模型使用的法线分布函数 D(h)为：
$$D_{blinn}(h)=\frac{gloss+2}{2\mathrm\pi}{(n\cdot h)}^{gloss}\;$$
其中， gloss 是与表面粗糙度相关的参数，它的值可以是任意非负数。上面式子和我们之前所见的 Blinn-Phong 模型有所不同，这是因为我们在里面加入了归一化因子， 这是因为`法线分布函数必须满足一个条件，即所有微面元的投影面积必须等于该区域宏观表面的投影面积。` 因此，上述公式也被称为是`归一化的 Phong 法线分布函数`
### 阴影-遮挡函数
阴影-遮挡函数 G(l, v, h)也被称为几何函数（geometry function），它表明了具有给定面法线 m的微面元在沿着入射方向 l 和观察方向 v 上不会被其他微面元挡住的概率。在微面元理论的 BRDF中， m 可以使用半向量 h 来代替，因为只有这部分微面元才会把光线从 l 方向反射到 v 上。 由于G(l, v, h)表示的是一个概率值，因此它的值是一个范围在 0 到 1 之间的标量。 学术界发表了许多对于 G(l, v, h)的分析模型， 这些公式大多建立在一些简化的表面模型基础下。许多已发表的微面元 BRDF 模型习惯把 G(l, v, h)和高光反射项的分母(n ⋅ l)(n ⋅ v)部分结合起来，即把 G(l, v, h)除以(n⋅ l)(n ⋅ v)的部分合在一起讨论，这是因为这两个部分都和微面元的可见性有关， 因此 Naty Hoffman在他的演讲中称这个合项为`可见性项`（visibility term）。一些 BRDF 模型选择完全省略可见性项，即把该项的值设为 1。这意味着， 这些 BRDF 中的G(l, v, h)表达式等同于：
$$G_{implicit}(l,v,h)\;=\;(n\cdot l_c)(n\cdot v)\\$$
目前在图形学中广受推崇的是` Smith 阴影-遮掩函数`。 Smith 函数考虑进了表面粗糙度和法线分布的影响。
$$G(l,v,h)\;=\;\frac2{1+\sqrt{1+\sqrt{1+\alpha_g^2\tan\left(\theta_v\right)^2}}}\\$$
$$\alpha_g={(0.5+\frac{roughness}2)}^2\;$$
上述公式中的 θv 表示观察方向 v 和表面法线 n 之间的夹角。 根据艺术家的反馈以及对测量得到的 BRDF 图像的观察， Disney 在上述式子中重新映射了 αg 和 roughness 之间的关系，由此得到了一个在视觉上让艺术家更加满意的效果。
## PBS中的光照
尽管基于物理渲染的理论比较复杂，但在实际应用中绝大部分情况下我们其实只需要按照上面提到的各种公式来实现相应的 BRDF 模型即可。 然而， 要想得到画面出色的渲染效果，仅仅应用这些公式是远远不够的， 我们还需要为这些 PBS 材质搭配以出色的光照。
基于图像的光照通常指的是把场景中远处的光照存储在类似环境贴图的图像中。 这些环境贴图可以表示光滑物体表面反射的环境光， 从而允许我们可以快速得到拥有很高细节的真实光照效果。 在 Unity 中，这种光照通常是由反射探针（Reflection Probes）机制来实现的， 我们可以在 Shader中获取当前物体所在的反射探针并在需要时对它们的采样结果进行混合。
## Unity中的PBS实现
由于 Unity版本的不同，内置 PBS 的实现也可能会发生变化。 除此之外，在学术界和工业界仍然不断有新的或改良后的 BRDF 模型的出现， 读者也可以根据项目需要选择与 Unity 实现不同的 BRDF 模型。尤其是如果需要在移动端应用基于物理的渲染，除了效果外性能是我们最应当关心的问题之一，此时我们可能需要针对移动平台对采用的 BRDF 模型进行一些修改。
# 动手:PBS实践
((`这个Shader代码没给全，我按照自己的理解结合报错拼凑起来的，不保证正确，仅供参考`))
```GLSL
Shader "Unity Shaders Book v2/Chapter 18/Custom PBR"
{
    Properties 
    {
        //控制漫反射材质颜色
        _Color ("Color", Color) = (1, 1, 1, 1)
        //控制漫反射材质纹理
        _MainTex ("Albedo", 2D) = "white" {}
        _Glossiness ("Smoothness", Range(0.0, 1.0)) = 0.5
        //RGB通道控制材质的高光反射颜色
        _SpecColor ("Specular", Color) = (0.2, 0.2, 0.2)
        _SpecGlossMap ("Specular (RGB) Smoothness (A)", 2D) = "white" {}
        _BumpScale ("Bump Scale", Float) = 1.0
        //材质的法线纹理，控制凹凸程度
        _BumpMap ("Normal Map", 2D) = "bump" {}
        //控制材质自发光颜色
        _EmissionColor ("Color", Color) = (0, 0, 0)
        _EmissionMap ("Emission", 2D) = "white" {}
    }
    
    SubShader 
    {
        Tags { "RenderType"="Opaque" }
        LOD 300
        Pass 
        {
            Tags { "LightMode" = "ForwardBase" }
            CGPROGRAM
            #pragma target 3.0
            #pragma multi_compile_fwdbase
            #pragma multi_compile_fog
            #pragma vertex vert
			#pragma fragment frag
			#include "AutoLight.cginc"
			#include "UnityCG.cginc"
			#include "Lighting.cginc"
			#include "HLSLSupport.cginc"
			
			fixed4 _Color;
		    sampler2D _MainTex;
		    half _Glossiness;
		    sampler2D _SpecGlossMap;
		    sampler2D _BumpMap;
			float4 _BumpMap_ST;
			float _BumpScale;
			fixed4 _EmissionColor;
			sampler2D _EmissionMap;
			float4 _MainTex_ST;
			
			//告诉编译器尽可能使用内联调用方式来调用该函数
            inline half3 CustomDisneyDiffuseTerm(half NdotV, half NdotL, half LdotH, half roughness, half3 baseColor) 
            {
                half fd90 = 0.5 + 2 * LdotH * LdotH * roughness;
                // Two schlick fresnel term
                half lightScatter = (1 + (fd90 - 1) * pow(1 - NdotL, 5));
                half viewScatter = (1 + (fd90 - 1) * pow(1 - NdotV, 5));
                return baseColor * UNITY_INV_PI * lightScatter * viewScatter;
            }
            inline half CustomSmithJointGGXVisibilityTerm(half NdotL, half NdotV, half roughness)
            {
                // Original formulation:
                // lambda_v = (-1 + sqrt(a2 * (1 - NdotL2) / NdotL2 + 1)) * 0.5f;
                // lambda_l = (-1 + sqrt(a2 * (1 - NdotV2) / NdotV2 + 1)) * 0.5f;
                // G = 1 / (1 + lambda_v + lambda_l);
                // Approximation of the above formulation (simplify the sqrt, not mathematically correct but close enough)
                half a2 = roughness * roughness;
                half lambdaV = NdotL * (NdotV * (1 - a2) + a2);
                half lambdaL = NdotV * (NdotL * (1 - a2) + a2);
                return 0.5f / (lambdaV + lambdaL + 1e-5f);
            }
            inline half CustomGGXTerm(half NdotH, half roughness) 
            {
                half a2 = roughness * roughness;
                half d = (NdotH * a2 - NdotH) * NdotH + 1.0f;
                return UNITY_INV_PI * a2 / (d * d + 1e-7f);
            }
            inline half3 CustomFresnelTerm(half3 c, half cosA) 
            {
                half t = pow(1 - cosA, 5);
                return c + (1 - c) * t;
            }
            //对高光反射和掠射颜色进行菲涅尔插值，可以得到更加真实的菲涅尔反射效果
            inline half3 CustomFresnelLerp(half3 c0, half3 c1, half cosA) 
            {
                half t = pow(1 - cosA, 5);
                return lerp (c0, c1, t);
            }
            
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
                float2 uv : TEXCOORD0;
                float4 TtoW0 : TEXCOORD1;
                float4 TtoW1 : TEXCOORD2;
                float4 TtoW2 : TEXCOORD3;
                //阴影三剑客之一
                SHADOW_COORDS(4) // Defined in AutoLight.cginc
                UNITY_FOG_COORDS(5) // Defined in UnityCG.cginc
            };
            
            v2f vert(a2v v) 
            {
               v2f o;
               UNITY_INITIALIZE_OUTPUT(v2f, o); // Defined in HLSLSupport.cginc
               o.pos = UnityObjectToClipPos(v.vertex); // Defined in UnityCG.cginc
               o.uv = TRANSFORM_TEX(v.texcoord, _MainTex); // Defined in UnityCG.cginc
               //为了在片元着色器中把采样得到的切线空间下的法线方向转换到世界空间下
               //把变换矩阵的相关数据存储
               float3 worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;
               fixed3 worldNormal = UnityObjectToWorldNormal(v.normal);
               fixed3 worldTangent = UnityObjectToWorldDir(v.tangent.xyz);
               fixed3 worldBinormal = cross(worldNormal, worldTangent) * v.tangent.w;
               o.TtoW0 = float4(worldTangent.x, worldBinormal.x, worldNormal.x, worldPos.x);
               o.TtoW1 = float4(worldTangent.y, worldBinormal.y, worldNormal.y, worldPos.y);
               o.TtoW2 = float4(worldTangent.z, worldBinormal.z, worldNormal.z, worldPos.z);
               //阴影三剑客之一
               TRANSFER_SHADOW(o); // Defined in AutoLight.cginc
               //We need this for fog rendering
               UNITY_TRANSFER_FOG(o, o.pos); // Defined in UnityCG.cginc
               return o;
            }
             
            half4 frag(v2f i) : SV_Target 
            {
                //准备所有输入数据
                half4 specGloss = tex2D(_SpecGlossMap, i.uv);
                specGloss.a *= _Glossiness;
                half3 specColor = specGloss.rgb * _SpecColor.rgb;
                half roughness = 1 - specGloss.a;
                //计算掠射角的反射颜色，从而得到更好的菲涅尔反射效果
                half oneMinusReflectivity = 1 - max(max(specColor.r, specColor.g), specColor.b);
                half3 diffColor = _Color.rgb * tex2D(_MainTex, i.uv).rgb * oneMinusReflectivity;
                half3 normalTangent = UnpackNormal(tex2D(_BumpMap, i.uv));
                normalTangent.xy *= _BumpScale;
                normalTangent.z = sqrt(1.0 - saturate(dot(normalTangent.xy, normalTangent.xy)));
                half3 normalWorld = normalize(half3(dot(i.TtoW0.xyz, normalTangent),dot(i.TtoW1.xyz, normalTangent), dot(i.TtoW2.xyz, normalTangent)));
                float3 worldPos = float3(i.TtoW0.w, i.TtoW1.w, i.TtoW2.w);
                half3 lightDir = normalize(UnityWorldSpaceLightDir(worldPos)); 
                half3 viewDir = normalize(UnityWorldSpaceViewDir(worldPos)); 
                half3 reflDir = reflect(-viewDir, normalWorld);
                //计算阴影和光照衰减值atten
                UNITY_LIGHT_ATTENUATION(atten, i, worldPos);
                //开始计算BRDF光照模型
                half3 halfDir = normalize(lightDir + viewDir);
                //截取范围到[0,1]
                half nv = saturate(dot(normalWorld, viewDir));
                half nl = saturate(dot(normalWorld, lightDir));
                half nh = saturate(dot(normalWorld, halfDir));
                half lv = saturate(dot(lightDir, viewDir));
                half lh = saturate(dot(lightDir, halfDir));

                //计算BRDF中的漫反射
                half3 diffuseTerm = CustomDisneyDiffuseTerm(nv, nl, lh, roughness, diffColor);
                //计算BRDF中的高光反射
                //可见性V
                half V = CustomSmithJointGGXVisibilityTerm(nl, nv, roughness);
                //法线分布项
                half D = CustomGGXTerm(nh, roughness * roughness);
                //菲涅尔反射项
                half3 F = CustomFresnelTerm(specColor, lh);
                half3 specularTerm = F * V * D;
                //计算自发光项
                half3 emisstionTerm = tex2D(_EmissionMap, i.uv).rgb * _EmissionColor.rgb;
                //计算IBL项（基于图像的光照部分）
                half perceptualRoughness = roughness * (1.7 - 0.7 * roughness);
                half mip = perceptualRoughness * 6;
                half4 envMap = UNITY_SAMPLE_TEXCUBE_LOD(unity_SpecCube0, reflDir, mip);
                half grazingTerm = saturate((1 - roughness) + (1 - oneMinusReflectivity));
                //考虑粗糙度影响
                half surfaceReduction = 1.0 / (roughness * roughness + 1.0);
                half3 indirectSpecular = surfaceReduction * envMap.rgb * CustomFresnelLerp(specColor, grazingTerm, nv);
                half3 col = emisstionTerm + UNITY_PI * (diffuseTerm + specularTerm) * _LightColor0.rgb* nl * atten + indirectSpecular;
                UNITY_APPLY_FOG(i.fogCoord, c.rgb);
                return half4(col, 1);
            }
            
            ENDCG
        }
    }
}

```
[![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/12/QQ截图20191221164516.png)](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/12/QQ截图20191221164516.png)
## 反射探针
在实时渲染中，我们经常会使用 Cubemap 来模拟物体的反射效果。例如，在赛车游戏中，我们需要对车身或车窗使用反射映射的技术来模拟它们的反光材质。然而，如果我们永远使用同一个 Cubemap，那么， 当赛车周围的场景发生较大变化时，就很容易出现“穿帮镜头”，因为车身或车窗的环境反射并没有随着环境变化而发生变化。一种解决办法是可以在脚本中控制何时生成从当前位置观察到的 Cubemap，而 Unity 5 为我们ᨀ供了一种更加方便的途径，即使用反射探针（Reflection Probes）。
反射探针的工作原理和光照探针（Light Probes）类似，它允许我们在场景中的特定位置上对整个场景的环境反射进行采样，并把采样结果存储在每个探针上。当游戏中包含反射效果的物体从这些探针附近经过时， Unity 会把从这些邻近探针存储的反射结果传递给物体使用的反射纹理。如果物体周围存在多个反射探针， Unity 还会在这些反射结果之间进行插值，来得到平滑渐变的反射效果。实际上， Unity 会在场景中放置一个默认的反射探针，这个反射探针存储了对场景使用的 Skybox 的反射结果，来作为场景的环境光照。

反射探针同样有 3 种类型：

- Baked，这种类型的反射探针是过提前烘焙来得到该位置使用的Cubemap 的，在游戏运行时反射探针中存储的 Cubemap 并不会发生变化。需要注意的是，这种类型的反射探针在烘焙时同样只会处理那些静态物体（即那些被标识为 Reflection Probe Static 的物体）；
- Realtime，这种类型则会实时更新当前的 Cubemap，并且不受静态物体还是动态物体的影响。当然，这种类型的反射探针需要花费更多的处理时间，因此， 在使用时应当非常小心它们的性能。幸运的是， Unity 允许我们从脚本中通过触发来精确控制反射探针的更新；
- Custom，这种类型的探针既可以让我们从编辑器中烘焙它，也可以让我们使用一个自定义的 Cubemap 来作为反射映射，但自定义的 Cubemap 不会被实时更新。
## 调整材质
在 Unity 中，要想和全局光照、反射探针等内置功能良好地配合来得到出色的渲染结果，就需要使用 Unity 内置的 Standard Shader。
## 线性空间
在使用基于物理的渲染方法渲染整个场景时，我们应该使用线性空间（Linear Space） 来得到最好的渲染效果。默认情况下， Unity 会使用伽马空间（Gamma Space），如果要使用线性空间的话，我们需要在 Edit→Project Settings→Player→OtherSettings→Color Space 中选择 Linear 选项。图 18.17显示了分别在线性空间和伽马空间下场景的渲染结果。
[![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/12/QQ截图20191221165830.png)](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/12/QQ截图20191221165830.png)
从图 18.17 中可以看出，`使用线性空间可以得到更加真实的效果。`但它的缺点在于， 需要一些硬件支持来实现线性计算，但一些移动平台对它的支持并不好。这种情况下，我们往往只能退而求其次，选择伽马空间进行渲染和计算。那么，线性空间、伽马空间到底是什么意思？为什么线性空间可以得到更加真实的效果呢？这就需要介绍伽马校正（Gamma Correction） 的相关内容了。实际上，当我们在默认的伽马空间下进行渲染计算时，由于使用了非线性的输入数据，导致很多计算都是在非线性空间下进行的，这意味着我们得到的结果并不符合真实的物理期望。除此之外，由于输出时没有考虑显示器的显示伽马的影响，会导致渲染出来的画面整体偏暗，总是和真实世界不像。
# 答疑解惑
## 什么是全局光照
`全局光照，指的就是模拟光线是如何在场景中传播的，它不仅会考虑那些直接光照的结果，还会计算光线被不同的物体表面反射而产生的间接光照。`
在使用基于物理的着色技术时，当渲染表面上一点时，我们需要计算该点的半球范围内所有会反射到观察方向的入射光线的光照结果，这些入射光线中就包含了直接光照和间接光照。
通常来讲，这些间接光照的计算是非常耗时间的，通常不会用在实时渲染中。一个传统的方法是使用光线追踪，来追踪场景中每一条重要的光线的传播路径。使用光线追踪能得到非常出色的画面效果，因此， 被大量应用在电影制作中。但是，这种方法往往需要大量时间才能得到一帧，并不能满足实时的要求。
总体来讲， Unity 使用了实时+预计算的方法来模拟场景中的光照。其中，实时光照用于计算那些直接光源对场景的影响，当物体移动时，光照也会随之发生变化。但正如我们之前所说，实时光照无法模拟光线被多次反射的效果。为了得到更加真实的渲染效果，Unity 又引入了预计算光照的方法，使得全局光照甚至在一些高端的移动设备上也可以达到实时的要求。
### 预计算光照
预计算光照包含了我们常见的光照烘焙，也就是指我们把光源对场景中静态物体的光照效果提前烘焙到一张光照纹理中，然后把这张光照纹理直接贴在这些物体的表面，来得到光照效果。
这些光照纹理不仅存储了直接光照的结果，还包含了那些由物体反射得到的间接光照。
`但是，这些光照纹理无法在游戏运行时不断更新，也就是说，它们是静态的。`
由于静态的光照烘焙无法在光照条件改变时更新物体的光照效果，因此， Unity 使用了预计算实时全局光照（Precomputed Realtime GI） 为我们提供了一个解决途径，来动态地为场景实时更新复杂的光照结果。
正如我们之前看到的，使用这种技术我们可以让场景中的物体包含丰富的全局光照效果，例如多次反射等，并且这些计算都是实时的，可以随着光源和物体的移动而发生变化。这是使用之前的实时光照或烘焙光照所无法实现的。
Unity 全新的全局光照解决方案可以大大ᨀ高一些基于 PC/游戏机等平台的大型游戏的画面质量，`但如果要在移动平台上使用仍需要非常小心它的性能。一些低端手机是不适合使用这种比较复杂的基于物理的渲染。`
### 什么是伽马校正
`这部分内容非常重要，而且非常多，这里只选出了后半部分内容。前半部分对伽马的介绍也很重要，建议大家直接去乐乐的Github查看`
[2019年新增：改版后的第十八章](https://github.com/candycat1992/Unity_Shaders_Book "2019年新增：改版后的第十八章")
要想渲染出更符合真实光照环境的场景就需要使用线性空间。而Unity 默认的空间是伽马空间，在伽马空间下进行渲染会导致很多非线性空间下的计算，从而引入了一些误差。而要把伽马空间转换到线性空间，就需要进行`伽马校正（Gamma Correction）。`
伽马的存在使得我们很容易得到非线性空间下的渲染结果。在游戏渲染中，我们应该保证所有的输入都被转换到了线性空间下，并在线性空间下进行各种光照计算，最后在输出前通过一个编码伽马进行伽马校正后再输出到颜色缓冲中。 Untiy 的颜色空间设置就可以满足我们的需求。当我们选择伽马空间时，实际上就是“放任模式”，不会对 Shader 的输入进行任何处理，即使输入可能是非线性的；也不会对输出像素进行任何处理，这意味着输出的像素会经过显示器的显示伽马转换后得到非预期的亮度，通常表现为整个场景会比较昏暗。当选择线性空间时，Unity 会把输入纹理设置为 sRGB 模式，在这种模式下，硬件在对纹理进行采样时会自动将其转换到线性空间中；并且， GPU 会在 Shader 写入颜色缓冲前自动进行伽马校正或是保持线性在后面进行伽马校正，这取决于当前的渲染配置。如果我们开启了 HDR的话，渲染就会使用一个浮点精度的缓冲。这些缓冲有足够的精度不需要我们进行任何伽马校正，此时所有的混合和屏幕后处理都是在线性空间下进行的。当渲染完成要写入显示设备的后备缓冲区（back buffer）时，再进行一次最后的伽马校正。如果我们没有使用 HDR，那么 Unity 就会把缓冲设置成 sRGB格式，这种格式的缓冲就像一个普通的纹理一样，在写入缓冲前需要进行伽马校正，在读取缓冲时需要再进行一次解码操作。如果此时开启了混合（像我们之前的那样），在每次混合时，硬件会首先把之前颜色缓冲中存储的颜色值转换回线性空间中，然后再与当前的颜色进行混合，完成后再进行伽马校正，最后把校正后的混合结果写入颜色缓冲中。这里需要注意，透明通道是不会参与伽马校正的。
Unity 的线性空间并不是所有平台都支持的，例如， `移动平台就无法使用线性空间`。此时，我们就需要自己在 Shader 中进行伽马校正。对非线性输入纹理的校正代码通常如下：
```GLSL
float3 diffuseCol = pow(tex2D( diffTex, texCoord ), 2.2 );
```
在最后输出前，对输出像素值的校正代码通常如下面这样
```GLSL
fragColor.rgb = pow(fragColor.rgb, 1.0/2.2);
return fragColor;
```
但是，手工对输出像素进行伽马校正会在使用混合时出现问题。这是因为，校正会导致写入颜色缓冲内的颜色是非线性的，这样混合就发生在非线性空间中。一种解决方法是，在中间计算时不要对输出颜色值进行伽马校正，但在最后需要进行一个屏幕后处理操作来对最后的输出进行伽马校正，也就是说我们需要保证伽马校正发生在渲染的最后一步中，但这可能会造成一定的性能损耗。
你会说，伽马这么麻烦，什么时候可以舍弃它呢？如果有一天我们对图像的存储空间能够大大提升，通用的格式不再是 8 位时，例如是 32 位时，伽马也许就会消失。因为，我们有足够多的颜色空间可以利用，不需要为了充分利用存储空间进行伽马编码的工作了。这就是我们下面要讲的 HDR。
### 什么是HDR
`这部分同伽马部分，一样很重要`
[2019年新增：改版后的第十八章](https://github.com/candycat1992/Unity_Shaders_Book "2019年新增：改版后的第十八章")
HDR 是 High Dynamic Range的缩写，即高动态范围，与之相对的是低动态范围（Low Dynamic Range， LDR）。那么这个动态范围是指什么呢？通俗来讲，动态范围指的就是最高的和最低的亮度值之间的比值。
