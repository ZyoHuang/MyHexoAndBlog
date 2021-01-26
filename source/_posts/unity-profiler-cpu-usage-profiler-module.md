---
title: Unity Profiler学习笔记：CPU使用分析器模块
tags: []
id: '2037'
categories:
  - - '%e6%b8%b8%e6%88%8f%e5%bc%95%e6%93%8e'
    - Unity
date: 2019-09-24 23:10:17
---

<meta name="referrer" content="no-referrer" />



## CPU Usage Profiler module

CPU使用分析器模块的图表显示了在应用程序中花费的时间。它包含应用程序花费时间的所有重要领域的概述，比如渲染,它的脚本和动画\[toc\]。本文包括: - CPU使用图表 - CPU使用模块详细信息窗格 - Timeline视图 - 层次结构和原始层次结构视图 - 常见的样品 - 性能警告 - 分配调用堆栈 - 只有在编辑器模式下才会出现的样例

### Chart categories（图表类）

CPU使用分析器模块的图表跟踪应用程序主线程上花费的时间。时间花费分为九类。您可以通过在图表的图例中拖放类别来更改图表中的类别顺序。您还可以单击类别的彩色图例来切换其显示。

#### Rendering（渲染）

应用程序在渲染图形上花费的时间。

#### Scripts（脚本）

应用程序在运行脚本上花费的时间。

#### Physics（物理）

应用程序在物理引擎上花费的时间

#### Animation（动画）

您的应用程序在动画化SkinnedMeshRenderers、GameObjects上花费的时间 以及应用程序中的其他组件。这还包括用于系统动画和Animator组件的一些计算的时间

#### GarbageCollector（GC）

应用程序在运行垃圾收集器上花费的时间。

#### VSync（垂直同步）

在一个帧中等待targetFrameRate或下一个VBlank同步的时间。这是根据QualitySettings.vSyncCount值，或目标帧率，或VSync设置，该设置是应用程序运行的平台的默认或强制最大值。

#### Global Illumination（全局光照）

全局照明应用程序中用于照明的时间。

#### UI（UI）

显示应用程序的UI花费了多少时间。

#### Others（其他）

其他不属于任何其他类别的代码所花费的时间，例如整个EditorLoop，或者在编辑器中分析Playmode时的分析开销。

### 模块详细讯息面板

当您选择CPU Usage模块时，module details窗格将显示所选帧中所花费时间的详细信息。计时数据要么显示为timeline，要么显示为hierarchical table，您可以通过单击module details窗格中左上角的下拉菜单来更改。可提供的三种意见是:

#### Timeline（时间轴）

在帧长度的时间轴旁边显示特定帧的时间细分。 这是惟一一种视图模式，您可以使用它查看主线程以外的线程上的计时，并在线程之间关联计时，例如，作业系统工作线程在主线程上的系统调度它们之后开始运行。

#### hierarchy（层次结构）

根据时序数据的层次结构对时序数据进行分组。 此选项以降序列表格式显示应用程序调用的元素，按花费的时间(默认值)、分配的脚本内存量(GC.Alloc)或调用的数量排序。 要更改列表的谁许，请单击表列的标题。

#### Raw Hierarchy（原始层次结构）

以与计时发生的调用堆栈类似的层次结构显示计时数据。 Unity在这种模式下单独列出每个调用堆栈，而不是像在Hierarchy视图中那样合并它们。

### Timeline view（时间线视图）

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/09/profiler-cpu-timeline-view.png) 时间轴视图是CPU使用分析器模块的默认视图。 它包含应用程序中时间花在何处以及时间如何相互关联的概述。 时间轴视图显示来自所有线程的分析数据，这些数据位于它们自己的子节中，并且沿着相同的时间轴。 `这与层次结构视图不同，层次结构视图只显示来自主线程的分析数据。` 可以使用时间轴视图查看不同线程上的活动在并行执行时是如何相互关联的。 您可以看到您使用了多少不同的线程，例如作业系统的工作线程，这些线程上的工作是如何排队的，以及是否有任何线程正在空闲(“Idle”sample)或正在等待另一个线程或作业完成(“Wait for x”sample)。 在上面的截图中，作业系统的工作线程中有淡蓝色的动画示例，而主线程也处理动画数据。渲染工作在主线程和渲染线程之间分割。渲染线程与主线程不对齐。 在此特定帧的前0.4 ms期间，渲染线程仍在渲染最后一帧。同样，该帧占用下一帧的前0.1ms。属于其他帧的条形图是灰色的，模块详细窗格顶部的时间标尺上的垂直线在主线程上标记了帧的开始和结束。 当你分析GPU的使用时，工具栏上面的时间标尺显示了帧时间花在CPU上的时间和花在GPU上的时间。在这个例子中，游戏GPU时性能瓶颈的，在CPU上花费的渲染时间最多，所以这个应用程序需要优化它的图形性能。

### Navigating and selecting items（导航和选择项目）

要放大时间轴的区域，可以使用鼠标上的滚动轮，或者按住Alt键，同时按住鼠标右键拖动。您还可以使用水平滚动条的末端来放大。按下键盘上的A键来重置缩放，这样整个帧的时间都是可见的。 每当您看到线程底部有一个白色箭头时，您可以单击它来展开线程以显示所有的线，或者再次单击以只显示顶部的线。您还可以拖动分隔线程的行，以调整可以看到的行数。双击该行可以将线程部分的高度设置为调用堆栈的最大深度。 要平移视图，请按下鼠标中键或按住Alt键(macOS上的命令键)并按下鼠标左键。 要折叠和展开线程组，请单击视图最左侧线程名称旁边的折叠箭头。 要查看一个项目对CPU图表的贡献，请在下面的窗格中单击它，选择它，分析器将突出显示它的贡献，并使图表的其余部分变暗。若要取消选择该项，请单击视图中的其他位置。按下F键以聚焦您选择的当前样本，或者如果您没有选择任何内容，则显示默认的缩放级别。 要在工具提示中显示托管callstack，请在Profiler窗口工具栏的分配callstack下拉菜单中启用托管callstack选项。 在分析一个帧以显示该帧之前，需要启用Managed Callstacks设置。此选项仅当您在编辑器中进行概要分析时才有效。 您还可以手动测量时间轴视图中的任意时间跨度，方法是单击并水平拖动到任意位置，以在时间轴的某个部分上显示覆盖。您可以在顶部的时间标尺中看到覆盖层所包含的时间。按下F键，同时叠加显示框视图水平沿选定的时间部分。点击任何地方移除覆盖。

### Hierarchy and Raw Hierarchy view（层次结构和原始层次结构视图）

当切换到层次结构或原始层次结构视图时，只要示例位于主线程上，您的选择就会`保留下来`（emm，不太懂这里是啥意思，原文是carries over）。如果你不能立即找到你的选择，按F键聚焦。 ![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/09/profiler-cpu-hierarchy-view.png) Hierarchy视图列出了所有已进行概要分析的示例，并通过它们的共享调用堆栈和profilermarker的层次结构将它们分组在一起。原始层次结构视图不将样本分组在一起，这使得它非常适合在粒度级别上查看样本。在每一行旁边的层次结构中，每一步都显示以下详细信息:

#### Total（总花费）

在某一特定功能上的总时间百分比。

#### Self（自身）

在一个特定函数上花费的总时间百分比，不包括Unity调用子函数的时间。 例如，在截图中，41.7%的时间花在了相机渲染功能。这是因为它调用了很多绘图和筛选函数，但是当排除它调用的函数时，只有3.5%的时间花在相机渲染函数本身。

#### Calls（调用）

在这个帧中对这个函数的调用次数。 在Raw Hierarchy视图中，该列中的值始终为1，因为分析器不会合并示例的层次结构。

#### GC Alloc（GC）

GC分配当前帧中分配了多少脚本堆内存。 脚本堆内存由垃圾收集器管理。 每当调用GC.Collect()或存在脚本堆分配不适合堆当前大小时，垃圾收集器就会被触发。它标记所有不再引用它们的分配并收集它们。这个过程显示为GC。 在分析器中收集样本。 当您在堆上分配更多内存时，Unity会更频繁地运行垃圾收集器。 随着托管堆的增长，Unity需要更长的时间来标记和收集内存。 `因此，您应该在应用程序运行时将GC Alloc值保持为零，以防止垃圾收集器影响您的帧率，并保持总体堆大小较小。（这话说得就nmb离谱）`

#### Time ms（耗时）

时间ms在特定函数上花费的总时间(以毫秒为单位)。 `这一信息可能具有误导性，因为它只包含在主线程上花费的时间。` 如果您的应用程序使用Job系统或多线程渲染，您应该注意这一点。

#### Self（自身耗时）

将花费在特定函数上的总时间(以毫秒为单位)用ms表示，不包括Unity调用子函数的时间。

#### Warning（警告）

警告由警告图标指示，它显示在当前帧中触发警告的次数。 您还可以通过从模块详细信息窗格右上角的Details下拉菜单中选择Show Related Objects或Show calls视图来获得关于应用程序调用和使用已配置函数的位置的更多信息。 Show Related Objects视图显示UnityEngine的列表。 使用获取UnityEngine.Object的Begin()重载与分析器示例关联的对象。 一些Unity报告的例子内置了这些关联，比如Camera .Render示例，该示例链接到执行渲染的摄像机对象。 这些对象通过实例ID报告，并在Profiler窗口中解析为名称。 当你点击其中一个对象时，Unity会试图通过场景找到该对象 层次结构并ping它。 因为关联使用实例ID，所以只有在分析应用程序编辑器时，以及对象仍然存在时，ping才会起作用。 GC.Alloc示例中，这个视图显示了一个“N/A”项列表，每个项对应于发生在这个层次结构级别的分配，GC中列出了分配的大小。 如果在编辑器中配置应用程序，并启用了Allocation Callstacks，则在选择GC时,在这个视图中，即使没有启用Deep Profiling设置，也会显示所选择的已分配脚本对象的调用堆栈。 Show Calls面板显示所选的样例从何处被调用，以及它调用的其他函数。 此外，在模块详细信息窗格顶部的gear图标下，您可以启用或禁用Collapse Editor Only Sample设置。这将折叠播放器循环中的所有示例，这些示例只会因为编辑器安全检查而发生。 当样本被折叠时，它们的GC.Alloc值不会影响所内建的GC.Alloc值。 默认情况下启用此设置。

### Common samples（常见样例）

除了脚本代码生成的示例外，Unity还提供了大量的示例，这些示例可以让您了解应用程序中占用了哪些时间。下表解释了一些更常见的示例所做的工作。

#### Main thread base samples（主线程）

主线程基本示例提供了在应用程序上花费的时间与在编辑器和分析器活动上花费的时间之间的明确分隔。记录器还可以使用这些样本来获取主线程上帧的计时。

##### playerloop

将根循环到来自应用程序主循环的任何示例。当您在播放器以活动播放模式在编辑器中运行时启用Profile Editor设置时，此示例将嵌套在EditorLoop之下。

##### EditorLoop

将根循环到来自编辑器主循环的任何示例。这只在您在编辑器中配置播放器时才会出现。当您禁用Profile Editor设置时，此示例将显示帧的渲染和运行包含播放器的编辑器所花费的时间。

##### Profiler.CollectEditorStats

collecteditorstats是与为不同的活动分析器模块收集统计信息相关的任何示例的根。子样本分析器下的任何样本。CollectGlobalStats会给玩家带来开销。所有其他子样例只影响编辑器。 要关闭特定模块，请关闭它们的图表或调用profile.setareaenabled()。

#### Script update samples（脚本更新样品）

除非你使用的是Job系统，否则你的大部分脚本代码都嵌套在以下示例下面:

##### Update.ScriptRunBehaviourUpdate（更新）

这个示例包括对MonoBehaviour的调用。协同程序的更新和处理。

##### BehaviourUpdate（更新）

这个示例处理所有Update()方法。

##### CoroutinesDelayedCalls（协程调用）

在第一次生成后包含协程样本。

##### PreLateUpdate.ScriptRunBehaviourLateUpdate（后更新）

这个示例处理所有的LateUpdate()方法。

##### FixedBehaviourUpdate（固定间隔更新）

这个示例处理所有FixedUpdate()方法。

#### 渲染和VSync示例

这些示例显示CPU在哪些地方花费时间为GPU处理数据，或者在哪些地方等待GPU完成。如果GPU分析器不可用，或者它增加了太多的开销，工具栏不会显示这些信息。这些示例可以让您了解是cpu性能瓶颈还是gpu瓶颈。

##### WaitForTargetFPS（应用程序等待目标FPS的时间）

应用程序等待目标FPS的时间，由Application.targetFrameRate指定。 如果这个示例是Gfx.WaitForPresent的子示例。表示应用程序等待QualitySettings.vSyncCount中配置的VSync的时间。 注意:编辑器不会在GPU上进行VSync，而是使用WaitForTargetFPS来模拟VSync的延迟。 一些平台，特别是Android和iOS，强制执行VSync，或者默认帧速率上限为30或60。

##### Gfx.ProcessCommands（处理指令）

包含渲染线程上渲染命令的所有处理。其中一些时间可能用于等待来自主线程的VSync或新命令，您可以从它的子示例Gfx.WaitForPresent中看到这一点。

##### Gfx.WaitForCommands（等待指令）

表示渲染线程已经为新的命令做好了准备，并且可能表示在主线程上有瓶颈。

##### Gfx.PresentFrame（当前帧）

表示应用程序等待GPU渲染和呈现帧的时间，这可能包括等待VSync。 主线程上的WaitForTargetFPS示例显示了等待VSync花费了多少时间。

##### Gfx.WaitForPresent（等待当前帧）

表示主线程已经准备好开始渲染下一帧，但是渲染线程还没有完成对GPU渲染帧的等待。这可能表明您的应用程序是gpu性能瓶颈的。 要查看渲染线程同时花费了多少时间，请检查时间轴视图。 如果渲染线程在Camera.Rende中花费时间，你的应用程序是cpu性能瓶颈的，可能花费太多的时间发送draw calls或纹理到GPU。 如果渲染线程在Gfx.PresentFrame中花费时间。你的游戏是GPU性能瓶颈的，或者它可能正在GPU上等待VSync。一个GFX.WaitForTargetFPS子样例。 Gfx.WaitForPresent指示应用程序花费时间用于等待VSync的部分。

#### Physics samples

下表概述了一些高级物理分析器示例。 FixedUpdate调用所有这些示例。

##### Physics.Simulate（物理模拟）

通过指示物理引擎(PhysX)运行其模拟来更新当前物理的状态。

##### Physics.Processing（物理处理中）

所有非布物理作业。展开此示例，以显示物理引擎内部所做工作的底层细节。

##### Physics.ProcessingCloth（物理加工布料）

处理所有的布物理工作。展开此示例，以显示物理引擎内部所做工作的底层细节。

##### Physics.FetchResults（取得结果）

从物理引擎中收集物理仿真结果。

##### Physics.UpdateBodies（更新物理实体）

更新所有物理实体的位置和旋转。此示例还包含在发送这些更新时进行通信的消息。

##### Physics.ProcessReports（处理报告）

在物理FixedUpdate结束后运行。处理各个阶段对仿真结果的响应。Contacts、铰链。在这个示例中，中断并触发更新和消息。有四个不同的子阶段: Physics.TriggerEnterExits进程OnTriggerEnter和OnTriggerExit事件。 Physics.TriggerStays处理OnTriggerStay事件。 Physics.Contacts处理OnCollisionEnter、OnCollisionExit和OnCollisionStay事件。 Physics.JointBreaks处理更新和有关关节断裂的信息。

##### Physics.UpdateCloth（更新布料）

包含了与布及其蒙皮网格相关的更新。

##### Physics.Interpolation（插值）

管理所有物理对象的位置和旋转的插值。

### Performance warnings（性能预警）

CPU分析器可以检测一些常见的性能问题并警告您。这些将出现在模块详细信息窗格中的Hierarchy视图的警告列中。 ![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/09/ProfilerCPUWarningCount.png) Profiler可以检测到的具体问题有: Rigidbody.SetKinematic:为刚体重建非凸网格碰撞体 Animation.DestroyAnimationClip:触发RebuildInternalState Animation.AddClip:触发RebuildInternalState Animation.RemoveClip:触发RebuildInternalState Animation.Clone:触发RebuildInternalState AnimationDeactivate:触发RebuildInternalState

### Allocation Callstacks（分配调用堆栈）

如果在编辑器中配置应用程序，可以看到GC的完整调用堆栈样本。 为此，在Profiler窗口工具栏的Allocation Callstacks下拉菜单中启用Managed Allocations。在您打开此选项后GC.Alloc样本包含它们的调用堆栈。 每个脚本堆分配都显示为GC层次结构视图和时间轴视图中的示例。 在时间轴视图中，它是明亮的洋红色。要查看调用堆栈，请选择CPU分析器模块并选择GC.Alloc在时间轴视图中的样本。调用堆栈出现在选择突出显示中。 或者，您可以在层次结构或原始层次结构视图中查看调用堆栈。设置Details视图以显示相关对象。因为GC.Alloc样本没有名称，它们在这个面板中显示为N/A。当您选择一个N/A对象时，调用堆栈将显示在Details视图的下半部分。 ![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/09/profiler-cpu-allocation-callstacks.png) ![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/09/profiler-cpu-heirarchy-callstack.png)

### Editor only samples(只在编辑器模式下才有的样例)

有一些在编辑器中进行概要分析时才会出现的一些示例。这包括安全检查，比如GetComponentNullErrorWrapper，它帮助识别空组件的使用情况;校验一致性，验证对象设置;CheckAllowDestructionRecursive，这是一个销毁检查;和Prefab-related活动。所有这些示例都不存在于播放器中。 ![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/09/profiler-cpu-editor-collapsed.png) 默认情况下，Editor only samples在Hierarchy视图中折叠，并命名为`EditorOnly` SampleName。而`它们可能导致GC。`如果被折叠，它们对其所附样本的GC没有贡献。 要更改默认行为，请单击模块详细信息窗格右上角的gear图标，并禁用折叠编辑器only Samples选项。当您这样做时，您可以扩展样例并贡献它的GC将值赋给所附示例。 ![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/09/profiler-cpu-editor-expanded.png) 此选项不影响时间轴视图。这些示例通常可以被忽略，并提示在目标设备上配置Player构建，以发现实际问题。