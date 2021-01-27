---
title: Box2D篇：整合Box2D到项目，并支持导出数据到服务端
date:
updated:
tags: [Unity技术, 可视化编辑器, 服务器, Box2D]
categories:
  - - GamePlay
    - 实用工具
  - - 游戏引擎
    - Unity
keywords:
top_img:
cover: https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/07/QQ截图20190704203259.png
aplayer:
---
<meta name="referrer" content="no-referrer" />

### Box2D介绍
**为游戏打造的2D物理引擎，就像Unity自带的2D物理引擎一样的功能**
### Box2D官网
**[http://box2d.org](http://box2d.org "http://box2d.org")**
### 白纸无字大佬整合的Box2D地址
**[https://github.com/Zonciu/Box2DSharp](https://github.com/Zonciu/Box2DSharp "https://github.com/Zonciu/Box2DSharp")**
### 绪论
**因为服务端对某些技能进行击中判定，所以需要一个不依赖于Unity的物理引擎库，对于我做的Moba游戏，Box2D无疑是最佳选择。
如果是FPS游戏想在服务端做碰撞检测，需要3D的物理引擎库，因为你用2D物理引擎无法进行爆头，腰射这种判定。**
## 具体步骤
### 下载白纸无字的Box2D
### 客户端配置
运行link.bat批处理命令
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/07/QQ截图20190704193635.png)
然后只需要将这几个文件放到项目的Plugins文件夹下面即可
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/07/QQ截图20190704194039.png)
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/07/QQ截图20190704194243.png)
### 服务端配置
同样的，我们在服务端创建Box2D项目
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/07/QQ截图20190704203259.png)
`注意要开启允许不安全代码才不会报错`

### Box2D碰撞体数据可视化编辑器
[https://www.lfzxb.top/box2d_unityvistualeditor/](https://www.lfzxb.top/box2d_unityvistualeditor/)
### Box2D碰撞体碰撞关系可视化编辑器
[https://www.lfzxb.top/box2d_collisionrelationvisualeditor/](https://www.lfzxb.top/box2d_collisionrelationvisualeditor/)
### 服务端与Box2D相关的架构设计
由于代码较多，并且我也不确定自己的做法是否足够好，所以在这里只说一下大致思想，仅供参考。
- 一个技能产生的碰撞体即作为一个Entity
- 一个Entity包含一个Body（刚体），一个Body挂载一个或多个Fixture（形状）
- 使用ContactListener作为碰撞事件管理者，负责一个world所有碰撞事件的分发
如果想要深入了解更多细节，请前往
<script src='https://gitee.com/NKG_admin/MKGMobaBasedOnET/widget_preview' async defer></script>
<div id='osc-gitee-widget-tag'></div>
<style>
.pro_name a{color: #4183c4;}
.osc_git_title{background-color: #fff;}
.osc_git_box{background-color: #fff;}
.osc_git_box{border-color: #E3E9ED;}
.osc_git_info{color: #666;}
.osc_git_main a{color: #9B9B9B;}
</style>
目前已完成功能包括不限于
- Box2D碰撞体可视化编辑器
- Box2D碰撞关系可视化编辑器
- 客户端接收服务端传来的相关信息，重绘碰撞体
