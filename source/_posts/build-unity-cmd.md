---
title: 为Unity搭建完善的的Cmd工作环境
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

## 前言

我们在使用Unity开发游戏的时候，往往会需要GM工具来测试游戏稳定性，在游戏中呼出对话框输入命令行执行对应GM代码是非常高效的（下图来自B站Up：[https://www.bilibili.com/video/BV1v741187To](https://www.bilibili.com/video/BV1v741187To)）

![image-20201018150334389](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20201018150335.png)

此外，对于项目中不断增加的MenuItem编辑器拓展，每次使用工具都要去翻找对应的条目也是很麻烦的一件事，尤其是对于嵌套层次稍深的条目。。。

![image-20201018144035662](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20201018144035.png)

如果有个插件可以在Editor中输入Cmd执行对应MenuItem代码就好了，幸运的是上面提到的两个问题，我都找到了解决方案

## 正文

### Runtime Cmd-UnityIngameDebugConsole

首先是我们Runtime的Cmd，推荐：[https://github.com/yasirkula/UnityIngameDebugConsole](https://github.com/yasirkula/UnityIngameDebugConsole)插件

下载之后我们只需要把

![image-20201018145753901](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20201018145753.png)

放到场景中即可

如果我们想要在游戏中根据命令执行某个方法(这是最简单直接的用法，同样还支持参数传递，返回值，参数重定向等多种功能，可以去插件Readme查看更多)

```cs
        [ConsoleMethod("测试await/async的Log","这是一条测试await/async的Log")]
        public static async void TestAsyncLog()
        {
            Log.Info("这是一个测试常规的Log--一秒后会Log加密通话");
            await Game.Scene.GetComponent<TimerComponent>().WaitAsync(1000);
            Log.Info("这是一个测试常规的Log--别比别比，别比巴伯");
        }
```

然后打开Console，在对话框输入对应Cmd命令(自带智能提示和补全功能)，选中Cmd，点击回车即可运行

![image-20201018143510499](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20201018145922.png)

### Editor Cmd-MonKey Commander

然后是Editor下的Cmd（如果只是在Unity编辑器中，他可以同时担任Editor和Runtime的Cmd），推荐：[https://assetstore.unity.com/packages/tools/utilities/monkey-productivity-commands-119938?locale=zh-CN](https://assetstore.unity.com/packages/tools/utilities/monkey-productivity-commands-119938?locale=zh-CN)

对于它的自定义Cmd方法，官网文档有很详细的介绍：[https://sites.google.com/view/monkey-user-guide/command-creation](https://sites.google.com/view/monkey-user-guide/command-creation)

比如我们想让一个原本MenuItem方法变成我们Cmd启动的，可以这样

```cs
		[Command("ETEditor_RsyncEditor","Rsync同步",Category = "ETEditor")]
		private static void ShowWindow()
		{
			GetWindow(typeof (RsyncEditor));
		}
```

然后我们开启Monkey Command Console（默认热键是**`**,可以在Setting中进行修改![image-20201018150640229](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20201018150640.png)）

输入对应Cmd，然后选中，点击回车即可运行

![image-20201018150756913](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20201018150756.png)

我在使用Monkey Commander中遇到了一个奇怪的问题（Unity 2019.4.8 ltf）：每次修改代码进行编译的时候都会报这个错误

![image-20201018151126293](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20201018151126.png)

猜测应该是Unity 2019 Editor底层的缓存机制更改，导致插件运行到这部分代码时会出错，所以我们可以使用dnSpy对其dll进行更改并且重编译即可

![image-20201018151609202](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20201018151609.png)

## 总结

这两个插件的Cmd实现核心原理都很简单，都利用了C#强大的反射机制，但是他们提供的周边拓展如果自己实现的话，需要颇费一些功夫
