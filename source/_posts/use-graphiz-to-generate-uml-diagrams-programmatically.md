---
title: 使用Graphiz程序化生成UML的完整工作流
tags: []
id: '2835'
categories:
  - - 技术博客
date: 2020-05-17 01:00:25
---

<meta name="referrer" content="no-referrer" />



## 前言

想必大家在整理学习一个项目架构时会用到类似UML这样的类图来表述各个模块之间的关系，但是当一个系统很庞大时，我们的UML也会很大，如果项目架构变动或者自己哪里搞错了，基本上就是一场连连看灾难，今天就给大家安利一下Graphiz，用它来程序化生成我们想要的UML图。 我第一次接触这个插件是在一个AI开源库里，使用了Graphiz输出AI决策图，就像这样 !{}(https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2020/05/QQ截图20200516230456.png) 甚至这样 !{}(https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2020/05/QQ截图20200516230517.png) 很难想象如果人为的去画这些图是怎样的一副光景。 !{}(https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2020/05/QQ截图20200516231020.png) 当然了，上面那两个例子可能并不是很中肯，因为我们平时剖析的架构往往没有这么井井有条，他可能会是这样 !{}(https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2020/05/QQ截图20200516231358.png) 不过这都没有关系，在Graphiz中，我们只需要写好代码，这些图都会自动生成啦。

## 正文

### Graphiz简介

图形化是一种将结构信息表示为抽象图形和网络图的方式。自动图形绘制在软件工程，数据库和Web设计，网络以及许多其他领域的可视界面中具有许多重要的应用。 Graphviz是开源的图形可视化软件，用户只需要编写dot语言然后让Graphviz读取，即可生成自己想要的UML图。官网：[http://www.graphviz.org/about/](http://www.graphviz.org/about/ "http://www.graphviz.org/about/")

### Graphiz下载与环境配置

[https://graphviz.gitlab.io/\_pages/Download/Download\_windows.html](https://graphviz.gitlab.io/_pages/Download/Download_windows.html "https://graphviz.gitlab.io/_pages/Download/Download_windows.html") 选择msi和zip都可以，因为后面我们要用到批处理命令来利用Graphiz软件生成png或pdf，所以为了方便我们要去配置一下系统环境，在`用户变量的Path里加入我们Graphiz安装目录`，就像这样 !{}(https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2020/05/QQ截图20200516233430.png) 打开命令行输入`dot -help`，出现以下界面即为环境配置成功 !{}(https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2020/05/QQ截图20200516233652.png)

### 正式使用

创建dot文件

```d
digraph {
    rankdir = LR
    A -> B -> c[color=green]
}
```

创建批处理文件，编写以下代码

```bash
rem 这是dot文件所在目录
SET "rawPath=DotProjects"
rem 这是导出的pdf文件所在目录
SET "resultsPath=ExportProjects"
rem 这是导出的格式
SET "outputType=pdf"
rem 后缀过滤
SET "rawSearch=%rawPath%/*.dot"

SET "currentDir=%cd%"

MKDIR %resultsPath%
FOR %%I IN (%rawSearch%) DO dot -T%outputType% %rawPath%/%%I -o %resultsPath%/%%I.pdf

explorer %cd%\%resultsPath%

CD %currentDir%
```

按需求更改并双击执行就可以生成pdf文件了（当然可以自定义想要生成的类型） !{}(https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2020/05/QQ截图20200517005551.png)

### JetBrains系列编辑器插件支持

JB家的编辑器可以下载一个叫dotplugin的插件，他可以实时预览我们编写的dot程序生成的图片（不过这波显示结果和编辑器配合的不是很好） !{}(https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2020/05/QQ截图20200516234955.png)

## 推荐阅读

[画图总结（常用语法）](https://www.cnblogs.com/shuqin/p/11897207.html "画图总结（常用语法）")