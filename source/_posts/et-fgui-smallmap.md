---
title: ET篇：基于FGUI的小地图制作
date:
updated:
tags: [Unity技术, 游戏框架, GameFramework]
categories:
  - - 游戏引擎
    - Unity
  - - GamePlay
    - 游戏框架
keywords:
top_img:
cover: https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/05/QQ截图20190523125800.png
katex: true
aplayer:
---
<meta name="referrer" content="no-referrer" />

### 前言

**小地图开发对于我们来说是再常见不过的需求了，单用Unity中的UGUI来说倒也方便，但是用FGUI又如何呢？**
## 正式开始

### FGUI资源的搭建
**注意小地图图片要用装载器来包装，不然他将会是不可触摸的，`小地图宽高我们设置为200，大地图我们设置大小为100`
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/05/QQ截图20190523125800.png)
然后导出到Unity**

### FGUI坐标系统
**[http://www.fairygui.com/guide/unity/transform.html](http://www.fairygui.com/guide/unity/transform.html "http://www.fairygui.com/guide/unity/transform.html")**

### 小地图到大地图的坐标映射
**![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/05/QQ截图20190523130240.png)
落实到代码就是**
```csharp
        public override void Start(FUI5V5Map self)
        {
			//为小地图图片添加点击事件
            self.SmallMapSprite.onRightClick.Add(this.AnyEventHandler);
        }

        void AnyEventHandler(EventContext context)
        {
            //Log.Info("点击了小地图");
            //Log.Info($"在小地图上的坐标为{((GObject) context.sender).GlobalToLocal(context.inputEvent.position)}");
            Vector2 global2Local = ((GObject) context.sender).GlobalToLocal(context.inputEvent.position);
            Vector2 fgui2Unity = new Vector2(global2Local.x, 200 - global2Local.y);
			//这里的'-'号是我地图所在场景位置问题，大家可以根据自己项目来灵活调整，但公式不变
            Vector3 targetPos = new Vector3(-fgui2Unity.x / (200.0f / 100.0f), 0, -fgui2Unity.y / (200.0f / 100.0f));
            Game.EventSystem.Run(EventIdType.ClickSmallMap, targetPos);
        }
```

### 大地图到小地图的坐标映射
**将上述步骤反过来做一遍即可**

### 解决小地图和大地图点击寻路冲突问题
**由于我们小地图UI处于大地图上，当我们点击小地图时会发射射线到大地图，这并不是我们所想看到的，FGUI提供了一个接口判断是否点击到了FGUI**
**[http://www.fairygui.com/guide/unity/input.html](http://www.fairygui.com/guide/unity/input.html)**
**这样也算解决了这个问题**

### 运行Gif图
[点此查看运行图](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/05/bandicam-2019-05-23-13-18-08-677.mp4 "点此查看运行图")

### 感悟
**由点击冲突问题，我想到另一个问题，比如我们按下A键，会出现攻击范围并改变鼠标图标，我第一反应是两个组件里都写上**
```csharp
if (Input.GetKeyDown(KeyCode.A)
{
	//TODO
}
```
**但是这样做，以后组件多起来是否有点不妥？
我想到了做一个接收用户输入的组件，把指令做成事件分发出去
但是ks大哥给了我更好的解决方案，也是做一个组件，里面有大量bool来标记是否按下（长按）了（抬起了）某个按键，然后其他需要的组件自己来读取
有一说一，这种方法是典型的ECS思想，效率确实比我所想的事件机制高很多，tql，wsl**
