---
title: GameFramework篇：自定义事件并订阅的流程
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

## 自定义事件并订阅

**正如木头大佬所说**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190128114151449.png)

 **在GF里面创建和使用自定义事件还是比较容易的**

 **这里举个例子**

 **我们进行场景切换,要加载完下一场景所有资源才能切换吧,不然黑屏是什么鬼**

 **这里我是切换场景后,开始生成多个实体,所有实体生成完毕,关闭过渡UI,显示游戏界面**

 **创建**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190128114443281.png)

 **这个事件类里面的变量可以随意定义(至少要包括Clear()方法和Id变量)**

 **其中Fill()函数是用来填充事件类里面的数据的(可有可无,视情况而定),将LoadNextResourcesSuccess设为true**

 **定义好了事件,我们要怎么使用呢**

 **先订阅**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190128115611121.png)

 **HideUI为回调函数的名称,参数必须为(委托的原因)**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190128115823523.png)

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190128120246624.png)

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190128115107770.png)

 **在最后一行执行Fire,意为抛出事件,所有订阅这个事件的回调函数都将被执行**

 **最后,用完了,要放回去,取消订阅(在UI关闭的时候)**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190128121756479.png)

### **总结**

 **在GF里自定义事件主要分为以下步骤**

1. **创建自定义事件** 
2. **订阅自定义事件并指定回调函数** 
3. **达成条件,抛出事件** 
4. **取消订阅** **其实我们不难发现,GF的内置事件执行流程也和这大致一样,不过更复杂点就是了,大家可以自行查看内置事件,循着源码过一遍,就理解了**

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190128120106898.png)
