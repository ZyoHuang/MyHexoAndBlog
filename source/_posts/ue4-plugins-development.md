---
title: UE4插件开发总结
date:
updated:
tags: [UE4技术, 编辑器拓展, 工具开发]
categories:
  - - 游戏引擎
    - UE4
  - 工具流
keywords:
top_img:
cover: https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20201218233814.png
aplayer:
---
<meta name="referrer" content="no-referrer" />

## 前言

UE4的插件开发包含很多零散的知识点，包括Moudles，Plugins，Slate，UI_COMMAND等各类知识，为了方便自己和他人查阅，在此记录一些零碎的知识点以及优秀的文章。

## 正文

### 显示UI拓展点

通过开启Editor Preferences-General-Miscellaneous-Display UI Extension Points即可以在UI面板看到用绿色字符标识出的可拓展点。

### 利用UI反射器查看Slate绘制方式

我们在进行复杂的编辑器拓展开发时往往会需要参考UE4引擎已有的UI样式，或者说对自己绘制的Slate进行Debug，这时就需要点击Window-Developer Tools-Widget Reflector来进行操作。

### 利用UMG设计Slate基础样式

因为UMG就是基于Slate的封装，但他是可视化的，正好弥补了Slate不是所见即所得的短板，所以我们可以先用UMG拼出想要的Slate样式，然后再去根据这个样式去写Slate代码。

### UI_COMMAND基础认知

其实和UI_COMMAND相关模块有三个，分别是FUICommandList，FUICommandInfo以及UI_COMMAND宏，他们的关系是FUICommandList包含FUICommandInfo，并且映射FUICommandInfo到委托，UI_COMMAND宏负责正式注册FUICommandInfo。

### 拓展菜单栏

这部分大体来说有两种方案

- 通过FExtender实现
- 通过UToolMenus实现

#### 拓展已有菜单栏

先来看通过FExtender实现的

首先新建一个FExtender，此时就可以把它当成一个菜单项了，可以对他进行AddMenuExtension，然后通过FModuleManager::LoadModuleChecked加载要拓展的目标模块，随后通过GetMenuExtensibilityManager()->AddExtender进行添加拓展

```c++
TSharedPtr<FExtender> MenuExtender = MakeShareable(new FExtender());
//拓展LevelEditor的Help菜单的WindowLayout分段
MenuExtender->AddMenuExtension("WindowLayout",EExtensionHook::After,PluginCommands,FNewMenuDelegate::CreateRaw(
                                   this, &FOfficial_EditorToolbarButtonModule::FillSubmenu));
FLevelEditorModule& LevelEditorModule = FModuleManager::LoadModuleChecked<FLevelEditorModule>("LevelEditor");
LevelEditorModule.GetMenuExtensibilityManager()->AddExtender(MenuExtender);
```

然后看通过UToolMenus实现的

首先通过UToolMenus::Get()->ExtendMenu定位菜单类型，例如LevelEditor就是LevelEditor.MainMenu.Window，蓝图编辑器就是AssetEditor.BlueprintEditor.MainMenu.Window。。。

然后通过FindOrAddSection获取或添加段落，最后即可对段落添加Menu来拓展

```c++
UToolMenu* Menu = UToolMenus::Get()->ExtendMenu("LevelEditor.MainMenu.Window");
{
    //获取LevelEditor的Window菜单的WindowLayout分段
    FToolMenuSection& Section = Menu->FindOrAddSection("WindowLayout");
    //添加一个Menu
    Section.AddMenuEntryWithCommandList(FOfficial_EditorToolbarButtonCommands::Get().PluginAction, PluginCommands);
    //添加一个子Menu
    Section.AddSubMenu("New Sub Menu", FText::FromString("???"), FText::FromString("???"),
                       FNewToolMenuChoice(
                           FNewMenuDelegate::CreateRaw(
                               this, &FOfficial_EditorToolbarButtonModule::FillSubmenu)));
}
```

![image-20201218233814620](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20201218233814.png)

#### 拓展新菜单栏

拓展新菜单栏就只能通过传统的FExtender来进行操作，如果有大佬知道可以有别的做法，请告知

这部分其实和拓展已有菜单栏差不多，主要是把AddMenuExtension换成了AddMenuBarExtension

```c++
TSharedPtr<FExtender> MenuExtender = MakeShareable(new FExtender());
//将新的菜单栏放到LevelEditor的Help后面
MenuExtender->AddMenuBarExtension("Help", EExtensionHook::After, PluginCommands,
                                  FMenuBarExtensionDelegate::CreateRaw(
                                      this, &FOfficial_EditorToolbarButtonModule::AddPullDownMenu));
FLevelEditorModule& LevelEditorModule = FModuleManager::LoadModuleChecked<FLevelEditorModule>("LevelEditor");
LevelEditorModule.GetMenuExtensibilityManager()->AddExtender(MenuExtender);
```

![image-20201218233611343](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20201218233617.png)

### 拓展工具栏

这部分同样有两种方案

- 通过FExtender实现
- 通过UToolMenus实现

通过FExtender

```c++
TSharedPtr<FExtender> ToolExtender = MakeShareable(new FExtender());
ToolExtender->AddToolBarExtension("Content", EExtensionHook::After, PluginCommands,
                                  FToolBarExtensionDelegate::CreateRaw(
                                      this, &FOfficial_EditorToolbarButtonModule::FillToolbar));
FLevelEditorModule& LevelEditorModule = FModuleManager::LoadModuleChecked<FLevelEditorModule>("LevelEditor");
LevelEditorModule.GetToolBarExtensibilityManager()->AddExtender(ToolExtender);
```

通过UToolMenus

```c++
UToolMenu* ToolbarMenu = UToolMenus::Get()->ExtendMenu("LevelEditor.LevelEditorToolBar");
{
    FToolMenuSection& Section = ToolbarMenu->FindOrAddSection("Settings");
    {
        FToolMenuEntry& Entry = Section.AddEntry(
            FToolMenuEntry::InitToolBarButton(FOfficial_EditorToolbarButtonCommands::Get().PluginAction));
        Entry.SetCommandList(PluginCommands);
    }
}
```

### 自定义资产类型

[https://gmpreussner.com/reference/adding-new-asset-types-to-ue4](https://gmpreussner.com/reference/adding-new-asset-types-to-ue4)

### 自定义细节面板

[https://www.orfeasel.com/extending-the-details-panel/](https://www.orfeasel.com/extending-the-details-panel/)

## 推荐阅读

UE4的Modules和Plugins机制： [https://zhuanlan.zhihu.com/p/107270501](https://zhuanlan.zhihu.com/p/107270501)

【UE4 Renderer】<02> Slate系统：[https://zhuanlan.zhihu.com/p/28355166](https://zhuanlan.zhihu.com/p/28355166)

【UE4】插件与模块：[https://zhuanlan.zhihu.com/p/121152396](https://zhuanlan.zhihu.com/p/121152396)

B站阿棍【合集】UE4插件与Slate：[https://www.bilibili.com/video/BV1tv41117qg?p=1](https://www.bilibili.com/video/BV1tv41117qg?p=1)

基于元标记的UE4编辑器拓展库：[https://github.com/zhurongjun/QuickEditor](https://github.com/zhurongjun/QuickEditor)

在UE4中使用imgui：[https://github.com/zhurongjun/UEImgui](https://github.com/zhurongjun/UEImgui)
