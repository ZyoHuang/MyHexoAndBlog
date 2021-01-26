---
title: 个人对于游戏UI架构的思考与总结
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

## 前言
这阵子我的开源Moba项目要开始着手准备客户端的表现工作了，后端的逻辑基本上没有太大的问题。
如果对这个项目感兴趣的可以去看一下<a href='https://gitee.com/NKG_admin/MKGMobaBasedOnET/stargazers'><img src='https://gitee.com/NKG_admin/MKGMobaBasedOnET/badge/star.svg?theme=dark' alt='star'></img></a>
谈及客户端表现，UI是必不可少的一环，那么选定一个好的UI解决方案和框架就更加重要了。
对于解决方案，我们耳熟能详的有UGUI,FGUI,NGUI等。
对于UI框架，基本架构基本上就是MVC，MVVM这种`MV*`系列的框架。
在此之前，我并没有对这些UI框架的使用经验，所以就趁此机会好好学习一下。

## UI解决方案的选择
### UGUI
原生的UGUI似乎是一个好的选择，因为官方在一直维护，各个方面都有保障，但是许多功能需要自己重新造轮子，对于没有模块积累的人来说可能有些麻烦。
### NGUI

与UGUI还是有比较大的差别的，有一些轮子，拓展模块，底层也做了一些调整和优化。不过近几年势头越来越弱了，本身也不打算选择，如果UGUI和NGUI选一个，我会选UGUI。
### FGUI
FGUI全名FairyGUI，是一个开源的，跨平台UI解决方案，它包含几乎所有游戏UI常用功能，易于拓展，性能优秀，可以做到一次导出，各处使用，开发效率也很高。无疑更加适合独立游戏团队或者个人开发者。
所以我最终也是选定了FGUI作为我的项目UI解决方案。这是FGUI的官网：[https://www.fairygui.com](https://www.fairygui.com "https://www.fairygui.com")
## UI框架的选择
UI框架个人其实早已经接触了一些，比如GameFramework的UI框架，C#原生的MVC框架，个人觉得都已经很优秀了，但还是想找找看有没有更好的选择。于是就有了下面的学习历程。
### 一个完整的UI框架所必需要包含的基本功能
完整的UI生命周期管理，完整的数据更新方案。当然如果游戏的UI系统比较复杂，可能还需要UI组，层级管理这些功能。生命周期没什么好说的，直接来看数据更新部分。
### MVC
虽然大家可能都已经对这个MVC听到耳朵生茧子了，但我这里还是要说一下（皮一下~）

- M：Model，UI的数据以及对于UI数据的操作函数。`数据部分`可以理解为UI组件的具体渲染数据。以UGUI为例，一个UI上挂载了一个Text组件，一个Button组件，那么Model里可能就会有一个string属性代表Text显示的内容，一个Action属性表示Button被点击时会调用的回调函数。`逻辑部分`就是我们在游戏逻辑中会对UI数据进行的操作，可能还会有一些复杂的变换操作，比如一个人物最大生命值是100，并且血条本身只显示血量百分比，那么这个Model里的SetHPBarValue()函数就会执行下面的操作，把传进来的当前人物生命值（这里假设是50）换算到一个百分比的值，即：当前人物生命值/人物最大生命值 = 0.5。然后会被Controller调用把这个值给View，让它显示50%。
- V：View，UI的渲染组件，以UGUI为例，一个UI上挂载了一个Text，一个Button，那么这两个UI组件都将写到View里。也会包含一些操作，例如获取一个UI组件，对一个UI组件进行操作。
- C：Controller，本身Model和View他们是互不耦合的，所以需要一个中介者把他们联系起来，从而达到数据更新和渲染状态更新同步。它本身是没什么代码量的，代码几乎都在Model和View中。

然后就是很多人说MVC会有很多冗余代码，写起来不方便，应该是他们的使用方法有问题，可能并没有搞清楚M,V,C的分工，可能把M的工作写到了C里，或者把C里的工作写到了V里，总之概念就搞的一团糟，更不用说写出的代码如何了。
### MVVM
上面我已经提到了MVVM，下面来详细介绍一下MVVM到底是个什么东西。

- M：Model，还是和MVC的M差不多。
- V：View，还是和MVC的V差不多。
- VM：双向耦合（可能这里叫双向绑定更加贴切一点）M和V，什么是绑定呢，这里用观察者模式来解释可能更好懂一些。以M绑定V为例，M的更新会执行一个委托，而我们此时已经将V加入到委托链里了，所以M的更新会执行我们指定的一个委托函数，从而达到V也更新的目的。V绑定M也是同理。最后就可以M更新会导致V更新，V更新也会导致M相关数据更新的效果（避免循环应该加一个判重机制）。

我感觉MVVM和MVC并没有本质上的区别，他只是更加强制制定了更加明显的规范，所以写起来感觉MVVM更加厉害一些。
其实MVVM并没有大家想象中的那么神秘，可能是因为它的绑定机制让我们摸不到头脑？感觉yiyi大佬说的更加透彻一些，大家可以去看看：[https://zhuanlan.zhihu.com/p/99443196](https://zhuanlan.zhihu.com/p/99443196 "https://zhuanlan.zhihu.com/p/99443196")
### Flux和Redux
本来到这里就已经结束了，MVVM是我的不二之选，但是一次偶然的机会，我认识了Flux架构，这是奶泡大佬的知乎：[https://www.zhihu.com/people/xiao-fan-fan-zhu/posts](https://www.zhihu.com/people/xiao-fan-fan-zhu/posts "https://www.zhihu.com/people/xiao-fan-fan-zhu/posts")，大家可以去看看UFlux篇，基本把重要的点都讲到了。下面我来继续介绍Flux和Redux。
其实Flux和Redux基本上都是用来做Web应用的，只是他们架构中有一些优秀的点，可以拿到游戏来迁移应用。
Redux中文文档中有这样一段，我觉得非常经典：
`管理不断变化的 state 非常困难。如果一个 model 的变化会引起另一个 model 变化，那么当 view 变化时，就可能引起对应 model 以及另一个 model 的变化，依次地，可能会引起另一个 view 的变化。直至你搞不清楚到底发生了什么。state 在什么时候，由于什么原因，如何变化已然不受控制。当系统变得错综复杂的时候，想重现问题或者添加新功能就会变得举步维艰。`
然后我并不准备过于详细的讲解Flux和Redux这两个架构，但是会提供资料，有兴趣的可以去康康。
#### Flux
架构模型
$$Action->Dispatcher->Store->View->Action$$
核心是就是如上所示的`单向数据流`。
什么是单向数据流呢？
其实我们在`MV*`系列中的数据流向是紊乱的，在MV多对多的情况下尤为明显。这样有什么坏处呢？查Bug不好查，你有时候甚至不知道这个数据从哪来的，Bug自然不好找。
单向数据流就是始终保持数据只有一个流向，也就是上面那个示例，这样出了Bug基本上很快就可以确定是哪里出了问题。
#### Redux
其实我更倾向于Redux，虽然他脱胎于Flux，但是比Flux更加简洁，并且在单向数据流的基础上增加了状态管理。
我通读了一遍Redux的中文文档：[http://cn.redux.js.org](http://cn.redux.js.org "http://cn.redux.js.org")，受益良多。
它的架构是：
$$Action->Store->Reducer->View->Action$$
其中Store->Reducer->View这几步操作的就是State，也就是状态管理的关键。
说了那么多，什么是状态管理？
状态管理就是记录每一次UI数据改变，这样可以更加方便的回溯状态，找Bug，重现Bug，在开发调试的过程中有着非常重要的作用。
其实我个人更倾向于把状态管理做成一个热插拔的功能，只在开发时使用，正式上线不需要。
### UI框架总结
从`MV*`系列架构我学到了`逻辑与数据的分离与组合`，从Flux和Redux我学到了`单向数据流`和`状态管理`这两个重要的概念。但是我想各自取长补短，也就是全都要怎么办呢，去网上找了一大圈也没找到，但是还是有意外收获的，找到了城主大佬的博客：[https://www.cnblogs.com/OceanEyes/](https://www.cnblogs.com/OceanEyes/ "https://www.cnblogs.com/OceanEyes/")，他从零开始搭建了一个简易的MVVM框架，并且十分适合融合单向数据流和状态管理。
于是我自己修修改改缝缝补补，就写了一个简易的UI框架，他有MVVM的绑定特性，有Flux的单向数据流和Redux的状态管理，不依赖于Unity，用.net core 3.0写的，只是为了更好得理解逻辑。
<a href='https://gitee.com/NKG_admin/NKG_MVVM_UI/stargazers'><img src='https://gitee.com/NKG_admin/NKG_MVVM_UI/badge/star.svg?theme=dark' alt='star'></img></a>
## 大总结
最终选定FGUI和自己改的UI框架作为最终的解决方案，然而我又发现了一个问题，FGUI本身的自动生成代码在开发中已经相当于做了数据绑定和单向数据流的角色。。。那我岂不是白折腾一趟。
其实也没有啦，这个过程中也学到了很多从来没有听过的名词，知识点。受益匪浅。
可能以后再回来用UGUI的时候我会重新拾起来这个小框架也说不定。。。
