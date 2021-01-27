---
title: Odin篇：常用特性整理汇总
tags: [Unity技术, 编辑器拓展, 工具开发]
categories:
  - - 游戏引擎
    - Unity
  - 工具流
date: 2019-07-17 16:41:45
---

### 本博客将整合Odin常用特性，方便大家查阅

### 视频版本：

## 正文

### InfoBox

可用于任何属性上面，绘制一个消息盒子 构造函数中的参数含义 message：包含的信息 infoMessageType：信息类型 visibleIfMemberName：使此信息可见的成员名称

### Title

绘制一个标题 构造函数中的参数含义 title：标题名称 subtitle：标题 TitleAlignments：对齐方式 horizontalLine：是否绘制水平直线 bold：是否粗体显示标题

### Button

绘制一个按钮 ButtonSizes：按钮大小 name：按钮名称 ButtonStyle：按钮样式

### Range

绘制一个有范围的滚动条 min：最小值 max：最大值

### ReadOnly

使属性只读

### ListDrawerSettings

自定义List绘制设置

### PreviewField

为Unity属性绘制一个预览界面 Height：高度 Alignment：对齐方式 AlignmentHasValue：对齐方式是否指定

### EnumToggleButtons

绘制水平枚举类型的单选框

### EnumPaging

绘制一个可循环的枚举选择

### MinMaxSlider

绘制一个可编辑的最大最小值的滚动条

### MinValue，MinValue限制属性值域

### Wrap

限制属性的值，一旦超过边界值，将被重置到相反的临界值

### ColorPalette

颜色选择面板，可自定义

### AssetList

为asset文件绘制一个List

### DisableInInlineEditors

在编辑器预览中将不可得

### HideInInlineEditors

在编辑器预览中将不可见

### InlineEditor

预览unity内置的objects，例如shader，animation PreviewWidth：预览宽度 PreviewHeight：预览高度 IncrementInlineEditorDrawerDepth：允许绘制递增 Expanded：允许拉伸 DrawHeader：绘制标题头

### ProgressBar

绘制进度条 min：进度条最小值 max：进度条最大值 Segmented：允许进度条分段显示 ColorMember：可自定义颜色 BackgroundColorMember：背景颜色

### BoxGroup

绘制包围盒 ShowLabel：显示标题文字 CenterLabel：居中显示标题文字

### FolderPath

文件夹路径，可将文件夹拖进来 AbsolutePath：全路径 ParentFolder：父路径，如果不为空，FolderPath将显示相对于父路径的路径 RequireValidPath：确保合法路径 RequireExistingPath：确保需要一个已存在的路径 UseBackslashes：反斜杠

### FilePath

文件路径，可将文件拖进来 参数基本同FolderPath

### ValueDropdown

值的下拉列表

### ToggleLeftAttribute

绘制检查复选框在文字之前

### Multiline

绘制一个可编辑的string框 lines：行数

### MultiLineProperty

类似于Multiline，但是可以用于字段和属性

### EnableIf

将在指定成员到达指定值时显示 MemberName：成员名称 Value：值

### DisableIf

将在指定成员到达指定值时不显示 MemberName：成员名称 Value：值

### TypeInfoBox

也是添加一个信息框，不过不需要PropertyOrder和OnInspectorGUI，他会在InfoBox上方，一般直接放在class上方

### Required

如果此特性修饰的字段为空，将会出现一个错误信息 ErrorMessage：错误信息内容 InfoMessageType：信息类型

### ValidateInput

如果不符合条件，会出现错误信息

### DisableInPlayMode

在运行时不可得

### DisableInEditorMode

在编辑器模式下不可得

### TableMatrix

绘制矩阵形式

### TableList

平铺显示所有属性 ShowIndexLabels：显示序号 DrawScrollView：使用滚动条 ShowPaging：分页显示

### TabGroup

分组显示

### VerticalGroup

垂直分布的组 groupId：所属group名称 order：层级

### HorizontalGroup

水平分布的组 groupId：所属group名称 order：层级

### FoldoutGroup

分页显示组别

### BoxGroup

包围盒显示

### ToggleGroup

单选组

### 注意，组与组之间的嵌套用“/”分割