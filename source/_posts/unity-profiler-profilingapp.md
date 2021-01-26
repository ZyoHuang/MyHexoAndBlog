---
title: Unity Profiler学习笔记：监测你的应用
tags: []
id: '2034'
categories:
  - - '%e6%b8%b8%e6%88%8f%e5%bc%95%e6%93%8e'
    - Unity
date: 2019-09-24 21:10:22
---

<meta name="referrer" content="no-referrer" />



### 前言

要在目标发布平台上配置您的应用程序，请将目标设备连接到您的网络或通过电缆（emmm，应该指的是usb，typec，雷电这种吧）直接连接到您的计算机。您还可以在Unity编辑器中直接对应用程序进行分析，以便在应用程序开发期间大致分析结果。\[toc\]

### Remote profiling（远程性能分析）

你只能在Development Build勾选的情况下才能对应用程序进行性能分析。要设置此设置，请转到Build Settings(File > Build Settings)并选择应用程序的目标平台。 您还可以选中Autoconnect Profiler，使Unity编辑器在构建过程中将其IP地址烘焙到内置播放器中。当你启动播放器时，它会尝试连接到分析器 指向位于烤好的IP地址的编辑器。 ![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/09/profiler-build-settings.png) 当您构建并运行应用程序时，播放器将出现在Profiler窗口的Attach to player下拉框中。“Attach to Player”下拉菜单显示所有运行在你本地网络上的Unity应用。您可以根据应用类型和运行应用的主机名(例如，iPhonePlayer (Toms iPhone))来识别这些应用。 然后单击Record开始收集应用程序上的性能信息。 如果在Build Settings中启用了Autoconnect to Profiler, Unity将在应用程序启动时自动开始收集数据。 要在应用程序运行时连续收集数据，请在应用设置Run In Background (Edit > Project Settings > Player > Resolution and Presentation)。 当您启用此设置时，即使您让应用程序在后台运行，Profiler也会收集数据。 如果禁用它，则Profiler只在应用程序在活动窗口中运行时收集数据。

### Profiling in the Unity Editor（在Unity编辑器中进行概要分析）

如果使用Profiler窗口在编辑器中运行和配置应用程序，那么结果只是和目标平台运行应用程序时差不多，并不准确。 因为Play模式运行在与编辑器相同的进程中，所以您不能完全将应用程序的CPU、GPU和内存使用与Unity隔离开来。 这进一步扭曲了结果分析数据。 为了获得更准确的分析结果，您应该始终在目标设备上分析应用程序。

### WebGL

你可以在WebGL中使用Unity分析器，但是你不能通过WebGL把Profiler附加到正在运行的应用身上。 这是因为WebGL使用WebSockets进行通信，它不允许在浏览器端引入连接。 要附加到正在运行的播放器，您需要在Build Settings(菜单:File > Build Settings)中启用Autoconnect Profiler复选框。 `Unity不能为WebGL监测draw calls消耗。`

### Profiling on mobile devices（监测移动端）

iOS和Android设备都支持通过网络进行远程分析。 如果您正在使用防火墙，请在防火墙的出站规则中打开端口54998到55511。 这些是Unity用于远程分析的端口。 注意:有时，当您设置远程概要时，Unity编辑器可能不会自动连接到设备。 如果发生这种情况，您可以手动启动Profiler连接。 为此，在Profiler窗口中选择Attach To Player下拉菜单并选择适当的设备。 您还可以将目标设备直接插入计算机，以避免网络或连接问题。

#### iOS远程配置

要在iOS设备上启用远程分析，请遵循以下步骤: 1. 将iOS设备连接到WiFi网络。分析器使用本地WiFi网络将分析数据从设备发送到Unity编辑器。 2. 通过电缆把你的设备连接到你的电脑上。转到Build Settings(菜单:File > Build Settings)，启用Development Build和Autoconnect Profiler复选框，然后选择Build & Run。 3. 当应用程序在设备上启动时，在Unity编辑器中打开Profiler窗口(菜单:window > Analysis > Profiler)。

#### Android远程配置

Android设备支持两种远程分析方法:通过WiFi或通过Android Debug Bridge (adb)。 关于WiFi配置，请遵循以下步骤: 1. 禁用Android设备上的移动数据。 2. 将Android设备连接到WiFi网络。分析器使用本地WiFi网络将分析数据从设备发送到Unity编辑器。 3. 通过电缆把你的设备连接到你的电脑上。转到Build Settings(菜单:File > Build Settings)，启用Development Build和Autoconnect Profiler复选框，然后选择Build & Run。 4. 当应用程序在设备上启动时，在Unity编辑器中打开Profiler窗口(菜单:window > Analysis > Profiler)。 注意:运行Unity编辑器的Android设备和主机必须位于同一子网上，这样才能检测设备。 对于Android Debug Bridge (adb)概要，请遵循以下步骤:

1.  将您的设备通过电缆连接到计算机上，并确保它在adb中显示
2.  设备列表。
3.  转到Build Settings(菜单:File > Build Settings)，启用Development Build复选框，然后选择Build & Run
4.  当应用程序在设备上启动时，在Unity编辑器中打开Profiler窗口(菜单:window > Analysis > Profiler)。
5.  从“ Attach to Player ”下拉菜单中，选择AndroidProfiler(ADB@127.0.0.1:34999)。下拉菜单中的条目只有在所选目标为Android时才可见。 当您选择Build & Run时，Unity编辑器会自动为您的应用程序创建一个adb通道。 如果希望配置另一个应用程序，或者重启adb服务器，则必须手动配置此隧道。 为此，打开终端窗口或CMD提示符并输入:

`adb forward tcp:34999 localabstract:Unity-{insert bundle identifier here}` 要在Android构建中使用 `Deep Profiling` ，你需要在Android Player设置中启用Mono Scripting Backend(菜单:Edit > Project Settings > Player > Android > Other Settings)，并输入以下命令，通过adb命令开始游戏: ~$ adb shell am start -n {insert bundle identifier here}/com.unity3d.player.UnityPlayerActivity -e 'unity' '-deepprofiling'