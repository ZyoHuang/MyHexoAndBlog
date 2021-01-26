---
title: GameFramework篇：StarForce学习笔记
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


# 流程模块
## StarForce流程讲解
  **环境：StarForce matser 3.1.7**

 **之前我们就提到过流程贯穿游戏的始终，那么今天就来详细的说一下流程模块**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/Procedure.png)

 **ProcedureLauch为游戏入口流程，我们就来看他是什么，他做了什么**

 **一眼看见他继承自ProcedureBase**

 **我们一层层跟进，发现他和状态机有着不可分割的关系，每个流程都是一个状态**

 **他们的关系大体上可以用这个UML图解释，当然这只是非常非常简略的版本，如果要把所有特性都列出来，那将会很庞大的。推荐大家仔细研读这部分代码，体会编程的乐趣与开发者的智慧。**
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/QQ截图20190429123328.png)

 **我们需要知道，由基层的Fsm更新来驱动状态机状态的更新，状态机状态的切换也会传递到底层的Fsm上，即一切状态都被Fsm持有与维护**

 **我们就以ProcedureChangeScene这个流程类来熟悉上面的流程图**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/QQ截图20190429123403.png)

 **进入流程时，订阅了一些事件，并且为切换场景做好了准备**

 **流程状态更新，如果场景加载已经完成，就根据需要切换场景。其中的procedureOwner为流程持有者，为GameFramework.Fsm.IFsm<GameFramework.Procedure.IProcedureManager>别名**
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/QQ截图20190429123419.png)

 **离开流程，你要搬家了，自然要把订的报纸杂志之类的给取消，如果不取消订阅，会造成回调函数的重复执行(按自己需求来)**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/QQ截图20190429123427.png)

 **一般流程这三个生命周期函数都是必不可少的，也可以增加别的生命周期函数，按自己需求来**

 **基本的操作都知道了，下面就是StarForce的流程图了**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/QQ截图20190429124945.png)

 **如果哪里说的不对或者不好，恳请大家指出，共同进步！**

# 资源模块
## StarForce 资源辅助器
 **前前后后看了一星期，才有了这篇博文，再次感叹，心急吃不了热豆腐。**

 **在看这篇博文之前，建议先去了解一下Assetbundle和StreamingAsset和WWW和WebRequest这几个东西以及他们的用法。这是必须的，不然你会不知道我在说些什么。。。**

 **之前博文已经介绍了一些基本的模块，流程，热更新，AB包，在执行这些功能的时候，总是不可避免的谈到资源的加载，于是我就在这里讲一下**

 **先来到ProcedureSplash流程**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/1.png)

 **然后我们知道ProcedureCheckVersion流程里有资源初始化，初始化好之后再进入ProcedurePreload流程，那就说明如果当前资源模式为编辑器模式，一定是做了特殊处理，不然不可能逃脱资源初始化这一步骤。**

 **我们先不管资源初始化，先全局搜索EditorResourceMode，看它都影响着什么**

 **然后我们就在ResoureComponent里面发现了这句，这句的意思是，如果当前是编辑器模式加载资源，那么加载资源就用EditorResourceHelper这个辅助器，否则就用默认的资源加载辅助器（可更改）**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/2.png)

 **那么EditorResourceHelper是个啥呢**
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/3.png)
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/4.png)
 **我们看到，注意这个东西很重要，记住，它包含了将要被加载资源所有内容，文件名（完整路径），加载方式，优先级等**
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/5.png)
 **他将在Unity的Update生命周期函数被轮询，作用就是将加载好的资源通过LoadAssetSuccessCallback这类回调函数传回（Update函数只截取了部分，请大家自行查找并阅读）**
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/6.png)
 **好，我们再回到这里，看看他在非编辑器模式下设置的资源管理者是啥，加一句Debug**
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/7.png)
 **这个就是非编辑器模式下的辅助器了**
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/8.png)
 **总结一下这篇博客：**

 **在预加载流程，判断是否为编辑器模式，从而使用不同的资源加载器，不同的资源加载器就有自己不同的资源加载方式**

## ResourceComponent详解

**上篇博客已经将编辑器和非编辑器资源加载区分开了，那我们这篇就来具体看看非编辑器模式下资源加载**

 **进入InitResources函数，此时的ResourceManager已经是我们非编辑器模式下的资源器了，所以这篇博客中的m_ResourceManager不再过多赘述**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/1.2.png)

 **这 里的m_ReadOnlyPath就是**![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/3-1.png)**也就是我们AB包所在位置，他读取的是StreamAssets文件夹下的version.dat**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/4-1.png)

 **至于这里面到底加载的啥，我也弄不清楚，应该是资源映射表吧，我们先跟进去看看，我们发现他考虑很多种状况，并加载到了数据流，并通过回调函数，将加载的数据传递出来了**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/5-1.png)

 **我们再看看这个回调函数，可以看到他做了非常多非常多事情**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/6-1.png)

 **为加载到的资源打好标签**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/7-1.png)

 **处理依赖关系**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/8-1.png)

 **把资源全部加入资源组中**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/9.png)

 **处理各个资源组**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/10.png)

 **此时，已经将所有资源信息都加载到资源组里了， 需要使用的时候直接加载即可。**

 **总结一下过程：**

 **从version.dat读取二进制文件流，利用回调函数解析资源信息并添加到资源组，供项目使用，建议大家多看看**![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/11.png)**这部分代码，稳赚不亏！**

 **这时候已经把资源映射路径做好了，也就是说某种意义上完成了加载路径从Assets/StreamingAssets/到Assets/GameMain/的转变**

## 获取使用资源与总结

 **经过我的观察，我发现，不论是**

 **打开UI**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/1-1.png)

 **生成游戏物体**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/2-1.png)

 **加载配置文件**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/3-2.png)

 **都离不开这个**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/4-2.png)

 **我们在这个LoadAsset函数里看到了几个熟悉的身影**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/5-2.png)

 **没错了，这就是从我们初始化好的资源组拿取资源了**

### **GF资源部分总结：**

 **经过这几篇博客的胡扯，我们可以比较清晰的感受到资源从创建，到加载，再到使用这几个流程，先是初始化资源，直接一个InitResource函数就好，这个函数会处理资源映射表，完成加载路径的转变，所有一切GF底层都将帮你做好，因为AB包是它打出来的，所以他也知道要怎么处理，不用我们操心，之后就是拿取资源了，注意，使用LoadAsset函数时，要填写资源的全路径，例如Assets/GameMain/DataTables/UISound.txt，这样才能正确的拿到我们想要的资源，而内部怎么实现的，有兴趣的可以再深入了解。**

### **心路历程：**

 **在学习Resource这个模块的时候，真的很痛苦，因为刚开始什么都不懂，也没有指路人，就自己按自己的想法看，所以走了很多弯路，比如，我一直想着他是怎么用Assets/GameMain/路径加载到Assets/StreamingAssets/的文件的，如果不是加载的StreamingAssets下的文件，那他打AB包的意义又何在呢?这个回调函数是什么意思？这个回调函数是怎么实现的？WWW加载不是联网加载吗？怎么找不到加载StreamingAssets所需要的Application.stramingAssetsPath关键字呢?。。。总之问题多到爆炸，不过当把一切捋通顺，却又发现一切又是那么的理所应当。总的来说收获很大，也明白了一些学习方法，明天继续加油！

#  实体模块

## 实体加载

  **本篇博客来说一下StarForce里面的实体，StarForce里面实体的数据和逻辑是分离的，通过组件的形式组装到实体身上。**

 **我们来看SurvivalGame类，这个是游戏逻辑类**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190114180222219.png)

 **就做了一件事，每隔一秒生成一个实体，细化起来就是，拿到陨石表，然后调用ShowAsteroid函数来实例化陨石**

 **那我们就来看ShowAsteroid函数**

 **他是EntityComponent的拓展方法**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190114181234846.png)

 **参数为陨石数据类AsteroidData，在执行构造函数的时候对其相关属性进行了赋值**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190114181614217.png)

 **那么它所封装的ShowEntity类呢**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190114181829288.png)

 **第一个参数，陨石的实体逻辑类（可以看成AI）**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190114182435746.png)

 **他最终父类为EntityLogic，中间做了两层封装，主要是伤害计算和初始化之类的，大家可以自己看看，所以大家可以把实体当成我们的GameObject，因为继承自Mono，所以Unity自带的生命周期方法也都可以用，不过学都学了，我们不妨学习人家的思想，逻辑和数据相分离**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190114181913229.png)

 **第二个参数，EntityGroup，他是存储实体的组，需要在Entity模块设置，不然会报错，这一点上，和UI的显示有些类似**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190114182231730.png)

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190114182306641.png)

 **第三个参数，资源优先级**

 **第四个参数，就是刚刚我们所构造的实体数据了，里面存储了实体信息，方便内部封装的使用**

 **在往里走就是底层的加载资源文件，显示GameObject了，大家可以自己去看**

### **总结**

 **我们可以感觉到，StarForce在显示实体这里做了很多封装，可能写的时候很麻烦，但用起来真的很舒服，可以说一劳永逸**

 **在大家进行开发的时候，有这两种实例化实体方法供调用，也可以参考StarForce里的封装方法策略，找出最适合自己项目的策略**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190114182833355.png)

# UI模块

## UI的加载讲解
  **准备工作：**

1. **了解C#委托和事件** **这篇博客就以初学者角度来讲讲GF加载UI的方式，因为卸载UI的方式相对简单，就留给大家自己看了。**

 **来到ProcedureMenu流程，因为这里有UI**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/2019010514215932.png)

 **OnEnter函数做了两件事，订阅打开UI成功事件，并设置回调函数，意思是当UI实例化成功时，调用OnOpenUIFormSuccess函数 。**

 **好了，这时候我们先不管别的，先把这句的原理搞清楚**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190105142620848.png)

 **id传入的是OpenUIFormSuccessEventArgs哈希值（所以是唯一的）**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190105143039931.png)

 **我们继续往里看**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190105150401246.png)

 **当我们看到这里的时候，已经有些眉目了，GF内部将一类事件函数按规定的事件ID分类，这里就是很好的例子，以OpenUIFormSuccessEventArgs（打开UI成功事件）为一类，添加OnOpenUIFormSuccess这一事件参数。**

 **还没完，我们继续，添加了事件，我们怎么确保加载成功后才执行呢？**

 **回到****ProcedureMenu流程，我们把注意力放在这句上（因为除了这句找不到其他和这一块有关系的了）**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190105151545678.png)

 **如果 不存在UI实例，则通过AB加载方式加载，然后调用回调函数**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190105151830292.png)

 **到这里先暂停一下 ，先理解m_loadAssetCallbacks是个啥**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190105164355598.png)

 **是封装了加载资源回调函数的一个函数集，他封装的这些都来自UIManager里面的**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190105164619635.png)

 ** 继续回到LoadAsset函数**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190105163359896.png)

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190105163424974.png)

 **到最后，加载好还是回到了InternalOpenUIForm**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190105164956298.png)

 **所以我们再看InternalOpenUIForm**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190105164824748.png)

 **这又是个啥？ **

 **在UIManager里**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190105170546135.png)

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190105170603287.png)

 **查找引用，在UIComponent里，添加订阅**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190105170707373.png)

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190105170734807.png)

 **好了，看到这，已经快到和最开始的订阅结合的地方了，马上就要到重点了，下篇继续，先用一张UML图总结这篇博客的内容，其实也没啥，理解了委托和事件很容易**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190105170949931.png)

 **我们从这里开始看，可以看到这里已经给事件参数赋好值了**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190105171540948.png)

 **追踪OpenUIFormSuccessEventArgs**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190105171656597.png)

 **回到上篇博文的结尾，发现最终还是用到了我们刚开始时依据哈希值订阅的事件OpenUIFormSuccessEvent，二GameFramework.UI.OpenUIFormSuccessEventArgs是我们这篇文章开头说的打开界面成功事件，大家注意区分，**

 **然后我们发现这里用GameFramework.UI.OpenUIFormSuccessEventArgs填充了OpenUIFormSuccessEvent，而他如何用一个Fire函数实现分发事件我们马上就讲**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190105171918832.png)

 **看看它的Fill函数**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190105172248592.png)

 **这条路已经通了，从资源或者已经加载好的实例取得想要加载的UI，层层订阅与回调，最终将原本的OpenUIFormSuccessEvent填充，然后根据需要取其值**

 **然后我们要把整个流程走通，回到Fire函数，追踪**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190105173647533.png)

 **回到了梦想最开始的地方，事件池**

 **而我们的m_EventHandlers也在这定义的哦**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190105173507257.png)

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190105174018104.png)**，这里的m_Event是个事件队列，所以整个抛出事件函数最后是把事件放进事件队列**

 **放进这个事件队列是干嘛的？是进行轮询进行事件分发的**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190105174256823.png)

 **这一句就是处理事件了**

 ![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190105174336497.png)

 **至此，终于执行到了我们最开始订阅的事件函数**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/2019010514215932.png)

 **而我们也可以根据传进来的事件参数来判定是不是我们需要的，来继续进行操作**

### **总结UML图**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190105175224517.png)

 **至于退订，整体上要比订阅简单，大家可以自己去看看**

### 总结：
其实这也基本上是GF所有这种类型订阅函数的流程，大家看别的订阅函数，比如显示实体，大体也差不多，一法通万法通
