---
title: Unity编辑器拓展基础总结
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

从事Unity编辑器拓展也有一段时间了，该记录一下常见的知识点了，也方便自己日后查阅

## 结构

Unity编辑器拓展主要分为3大类

- UnityEngine.GUI：可用于编辑器和运行时，需要自行计算Rect
- UnityEditor.EditorGUI：只可用于编辑器，需要自行计算Rect
- UnityEditor.EditorGUILayout：只可用于编辑器，自动计算Rect

其中UnityEditor.EditorGUILayout基于UnityEditor.EditorGUI实现

## 常见类

### Rect

这个类型在编辑器拓展中十分常见，官方解释为

> A 2D Rectangle defined by X and Y position, width and height.
>
> 一个由X，Y坐标，width，height宽高定义的2D矩形

其以左上角为坐标原点，X往右递增，Y往下递增

更加详细介绍可参照：[Unity Rect官方文档](https://docs.unity3d.com/ScriptReference/Rect.html)

### GUIContent

GUIContent定义了一个GUI Item内容，最完整的构造函数如下

```cs
//构建同时包含文本，图片和定义的tooltip的GUIContent。 当用户将鼠标悬停在其上方时，全局GUI.tooltip会被设置为这个tooltip
public GUIContent(string text, Texture image, string tooltip)
```

### GUIStyle

一个GUIStyle定义了一个GUI Item的样式，最完整的构造函数如下

```cs
public GUIStyle(GUIStyle other)
```

但是我们最常用的还是string的隐式转换

```cs
public static implicit operator GUIStyle(string str)
```

常用GUIStyle样式可以从[Unity常用GUIStyle收集](https://gist.github.com/masa795/5797164)查看

## 常用编辑器拓展知识

### 菜单栏拓展

```cs
[MenuItem("TargetCatalogName")]
```

### Inspector拓展

```cs
[CustomEditor(typeof(TargetClassName))]
```

### 双击资源回调

```cs
[UnityEditor.Callbacks.OnOpenAsset(TargetCallBackOrder)]
```

### UnityEditor生命周期

```cs
UnityEditor.EditorApplication
```

### Undo/Redo

```cs
UnityEditor.Undo
```

### ScriptableObject序列化相关

- 如果一个ScriptableObject（下面简称为SO）的HideFlag设置为HideInHierarchy，那么LoadAllAssetRepresentationsAtPath会将其忽略
- 如果不将SO保存为Asset，运行游戏后将会被GC，从而丢失引用

### 资源管理相关

```cs
//添加子Asset
UnityEditor.AssetDatabase.AddObjectToAsset(subAsset, mainAsset);
```

保存资源时，保险起见应当进行

1. EditorUtility.SetDirty
2. AssetDatabase.SaveAssets
3. AssetDatabase.Refresh

### TreeView

[Unity Tree View官方文档](https://docs.unity3d.com/Manual/TreeViewAPI.html)

![image-20200925142638931](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20200925142645.png)

### Unity 折叠树

![image-20200925142848554](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20200925142848.png)

https://blog.csdn.net/e295166319/article/details/52370575

这个特性其实EditorGUILayout.BeginFoldoutHeaderGroup也可与之对位

### AdvancedDropdown

[Unity AdvancedDropdown官方文档](https://docs.unity3d.com/ScriptReference/IMGUI.Controls.AdvancedDropdown.html)

![image-20200925144255685](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20200925144255.png)

## 其他资源

[Unity Editor中所有图标](https://github.com/nukadelic/UnityEditorIcons)，包含所有Unity Editor中图标，方便使用

[Unity开源的CSharp部分](https://github.com/Unity-Technologies/UnityCsReference)，可在其中找到自己想要实现的Editor样式源码

[Unity编辑器拓展手册，超全](http://49.233.81.186/guicreation.html)，日文版，但是内容十分详尽

## 总结

Unity编辑器拓展知识点十分琐碎，是一个不断积累的过程，共勉。
