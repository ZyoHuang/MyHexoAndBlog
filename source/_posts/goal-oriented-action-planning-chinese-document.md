---
title: ReGoap（目标导向的AI系统）中文文档
tags: []
id: '2761'
categories:
  - - GamePlay
date: 2020-04-16 21:54:17
---

<meta name="referrer" content="no-referrer" />



## 前言

昨天听群友提到了GOAP（Goal Oriented Action Planning）目标导向的AI系统，百度稍微了解了一下，看起来要比行为树高级一些，所以今天来好好研究一下，看看他的庐山真面目。 因为国内资料较少，所以就准备去Github找一下。就找到了这个库：[https://github.com/luxkun/ReGoap](https://github.com/luxkun/ReGoap "https://github.com/luxkun/ReGoap")。 本篇文档对于Goap本身介绍并不多，想要详细了解的可以去看FEAR在GDC做的分享：[https://www.lfzxb.top/gdc-sharing-of-ai-system-based-on-goap-in-fear-simple-cn/](https://www.lfzxb.top/gdc-sharing-of-ai-system-based-on-goap-in-fear-simple-cn/ "https://www.lfzxb.top/gdc-sharing-of-ai-system-based-on-goap-in-fear-simple-cn/") 或者去这个网站了解GOAP更多相关内容：[http://alumni.media.mit.edu/~jorkin/goap.html](http://alumni.media.mit.edu/~jorkin/goap.html "http://alumni.media.mit.edu/~jorkin/goap.html") 我的Goap开源库（多线程，JobSystem，路径规划算法优化）：[https://gitee.com/NKG\_admin/NKGGOAP](https://gitee.com/NKG_admin/NKGGOAP "https://gitee.com/NKG_admin/NKGGOAP") 出于学习的目的，我来翻译一下它的Readme。`文章中的一些链接将省略。` 注：

*   正文中的`规划/计划系统`字样一般都可以当做`GOAP系统`。
*   正文中的`动作/行为/操作`字样一般都可以当做`Action`。
*   正文中的`效果/影响`字样一般都可以当做`Effect`。
*   正文中的`目标/目的`字样一般都可以当做`Goal`。
*   正文中的`记忆/内存`字样一般都可以当做`Memory`。

# ReGoap

带有Unity例子和协助类的通用型C# GOAP(Goal Oriented Action Planning) 库。 这个库非常通用，可以用在任何游戏引擎上。

## 让我们开始（报废版解说）

通过点击这里下载[Unity的FSM示例](https://github.com/luxkun/ReGoap/tree/master/ReGoap/Unity/FSMExample "Unity的FSM示例")开始。 这个在Unity里使用ReGoap库的示例通过一个简单的有限状态机来处理巨量的行为（在大多数游戏中只需要三个状态就可以满足：默认，切换，动画）

## 让我们开始（标配版解说）

### 介绍GOAP

如果你仅仅想使用这个库并想了解一下案例请跳转`在Unity使用ReGoap`。 在向你解释如何在你的游戏中使用这个库之前，让我先向你解释一下一个GOAP系统是如何工作的，来自一句Jeff Orkin的名言。

> 面向目标的行动计划(又名GOAP，与soap押韵)是指专为实时控制游戏中自主角色的行为而设计的类似`斯坦福研究院问题解决系统`的简单规划架构。

基本上，他做的所有事情就是寻找一个解决方案（应该是一个action列表）来完成我们想要让Unit完成的目标。（这个寻找过程是Unit自主进行的）。 你需要理解的主要内容包括：State，Action，Goal，Memory以及Sensors。

#### State

它是世界的定义，在这个库中，他们被封装成一个`Dictionary<string,object>` 可以在这个文件中找到ReGoapState类：[https://github.com/luxkun/ReGoap/blob/master/ReGoap/Core/ReGoapState.cs](https://github.com/luxkun/ReGoap/blob/master/ReGoap/Core/ReGoapState.cs "https://github.com/luxkun/ReGoap/blob/master/ReGoap/Core/ReGoapState.cs") 它的内容就像这样 `"isAt":"enemy"`,`"isWarned":"true"`,`"hasWeapon":true`。

#### Action

可以被定义为一系列的先决条件和会造成的效果，这些都是AI代理（上面提及的Unit,下文都将称作AI代理）能够做的。 先决条件是这个Action想要执行所需的满足的东西，他被描述为一个状态。而效果，顾名思义，Action所造成的效果，也被称之为一个状态。 例子：

*   `'Open door': [pre: {'nearDoor': true, 'doorUnlocked': true}, effects: {'doorOpened': true}]`
*   `'Close combat attack': [pre: {'weaponEquipped': true, 'isAt': 'enemy'}, effects: {'hurtEnemy' true}]`
*   `'Go to enemy': [pre: {'enemyInLoS': true, 'canMove': true}, effects: {'isAt': 'enemy'}]`
*   `'Equip weapon': [pre: {'hasWeapon': true}, effects: {'weaponEquipped': true}]`
*   `'Patrol': [pre: {'canMove': true}, effects: {'isPatrolling': true}]`

重要：`值为False的先决条件不被支持` 重要：`当Action已经完成时，Action的效果将不会被写入记忆中，这并不是一个脑瘫设计，因为在大多数游戏中，你需要从记忆（Memory）或传感器（Sensor）设置这些变量。` `但如果确实需要，可以在ReGoapAction中重写Exit并将效果设置到记忆中，如下面的示例所示。`

#### Goal

可以被定义为一系列的请求，也被描述为状态，这是AI代理的目标 例子：

*   'Kill Enemy': {'hurtEnemy': true}
*   'Patrol': {'isPatrolling': true}

#### Memory

是AI代理的记忆，所有AI代理知道的，感受到的都需要放在这里。 一个记忆可以有很多感知，在这个库里，被称作Memory Helper（Sensor）。 记忆的最基本工作是创建并保持更新'世界'的状态。

#### Sensor

是一个Memory Helper。他应该监管一个特定的范围。 例子：

*   EyeSensor (检查是否有敌人在当前视线里)
*   EarsSensor (检查是否有敌人被听到)

`你当然也可以制作一个EnemySensor。他同时拥有EyeSensor和EarsSensor的功能。`

### 在Unity使用ReGoap

1.  把这个仓库Clone到你的Unity项目里。
2.  为你的AI代理创建一个GameObject。
3.  添加一个ReGoapAgent组件，选择一个名称（必须自定义继承自ReGoapAgent或IReGoapAgent接口的类）。
4.  添加一个ReGoapMemory组件，选择一个名称（必须自定义继承自ReGoapMemory或IReGoapMemory接口的类）。
5.  \[可选\] 添加你自己的继承ReGoapSensor或IReGoapSensor的感知类。
6.  \[可选\] 添加你自己的继承ReGoapAction或IReGoapAction的Action类（请务必的明智的设计这个Action类拥有什么样的先决条件和效果）并且继承Action逻辑并重写Run方法，这个方法将会被ReGoapAgent调用。
7.  \[可选\] 添加你自己的继承ReGoapGoa或IReGoapGoal的目标类。（请务必的明智的设计这个Goal类拥有什么样的目标状态）。
8.  添加一个ReGoapPlannerManager（必须自定义继承自ReGoapPlannerManager的类）到任意游戏物体身上（`除了AI代理！`），他将会处理所有的规划。

然后呢？没有然后了。这个库会处理所有的AI规划，选择一系列Action来完成一个目标，所有你需要做的就是实现自定义的Action和Goal。 在下一段中，我将解释如何创建自己的类(但对于大多数行为，您需要继承的只是GoapAction和GoapGoal)。

#### 自定义ReGoapAction

自定义Action可以参考这个例子：[https://github.com/luxkun/ReGoap/tree/master/ReGoap/Unity/FSMExample/Actions](https://github.com/luxkun/ReGoap/tree/master/ReGoap/Unity/FSMExample/Actions "https://github.com/luxkun/ReGoap/tree/master/ReGoap/Unity/FSMExample/Actions") 想要查看有哪些方法可以重写可以参考这个类：[https://github.com/luxkun/ReGoap/blob/master/ReGoap/Unity/ReGoapAction.cs](https://github.com/luxkun/ReGoap/blob/master/ReGoap/Unity/ReGoapAction.cs "https://github.com/luxkun/ReGoap/blob/master/ReGoap/Unity/ReGoapAction.cs") 你必须通过继承IReGoapAction接口或ReGoapAction类来自定义ReGoapAction。明智的设计好通用类型，他们在所有的代理中都是平等的。通常是string(Key)，object(Value)，当然也可以是int/enum(Key)，以及其他通用但轻量的类型。 对于一个简单的实现，你可以像下面这样做

```csharp
public class MyGoapAction : ReGoapAction<string, object>
{
    protected override void Awake()
    {
        base.Awake();
        preconditions.Set("myPrecondition", myValue);
        effects.Set("myEffects", myValue);
    }
    public override void Run(IReGoapAction<string, object> previous, IReGoapAction<string, object> next, ReGoapState<string, object> settings, ReGoapState<string, object> goalState, Action<IReGoapAction<string, object>> done, Action<IReGoapAction<string, object>> fail)
    {
        base.Run(previous, next, goalState, done, fail);
        // 在这里编写你的游戏逻辑
        // 当结束时，如果成功就执行done回调，否则就执行fail回调
        // 并把自身通过done或fail传递到AI代理来决定下一步的操作。
        if(hasSuccess)
        {
            done(this); // 这一步将会告知ReGoapAgent这个行为已经成功执行并且在整个AI规划中继续前进
        }
        else
        {
            fail(this);// 表示行为失败了，执行fail回调，这个ReGoapAgent会自动无效化当前规划并且询问ReGoapPlannerManager来请求一个新的规划
        }
    }
}
```

正如在ReGoapAction之前编写的那样，默认情况下不会在Memory上写入效果，但是Memory应检查效果是否有效完成，如果出于任何原因要在Action结束时设置效果，则可以添加如下代码到您的ReGoapAction实现中：

```csharp
    public override void Exit(IReGoapAction<string, object> next)
    {
        base.Exit(next);

        var worldState = agent.GetMemory().GetWorldState();
        foreach (var pair in effects) {
            worldState.Set(pair.Key, pair.Value);
        }
    }
```

你也可以让先决条件和效果在下一个Aciton的先决条件或效果的影响下动态更改，这个例子就是演示如何在你的AI代理中处理GoTo Action。[https://github.com/luxkun/ReGoap/blob/master/ReGoap/Unity/FSMExample/Actions/GenericGoToAction.cs](https://github.com/luxkun/ReGoap/blob/master/ReGoap/Unity/FSMExample/Actions/GenericGoToAction.cs "https://github.com/luxkun/ReGoap/blob/master/ReGoap/Unity/FSMExample/Actions/GenericGoToAction.cs")

#### 自定义ReGoapGoal

这并不会麻烦，大多数目标只会重写Awake函数来添加您自己的目标状态。 无论如何，请查看ReGoapGoal：[https://github.com/luxkun/ReGoap/blob/master/ReGoap/Unity/ReGoapGoal.cs](https://github.com/luxkun/ReGoap/blob/master/ReGoap/Unity/ReGoapGoal.cs "https://github.com/luxkun/ReGoap/blob/master/ReGoap/Unity/ReGoapGoal.cs")，就像通过实现IReGoapGoal接口或继承ReGoapGoal从头实现自己的类所需的一切一样。 同样的，查看这个例子里的Goal：[https://github.com/luxkun/ReGoap/tree/master/ReGoap/Unity/FSMExample/Goals](https://github.com/luxkun/ReGoap/tree/master/ReGoap/Unity/FSMExample/Goals "https://github.com/luxkun/ReGoap/tree/master/ReGoap/Unity/FSMExample/Goals")

```csharp
public class MyGoapGoal : ReGoapGoal<string, object>
{
    protected override void Awake()
    {
        base.Awake();
        goal.Set("myRequirement", myValue);
    }
}
```

`注意：如果要主动关心AI代理的目标可用/不可用，请使用ReGoapGoalAdvanced。`

#### 自定义GoapSensor

查看GoapSensor基类：[https://github.com/luxkun/ReGoap/blob/master/ReGoap/Unity/GoapSensor.cs](https://github.com/luxkun/ReGoap/blob/master/ReGoap/Unity/GoapSensor.cs "https://github.com/luxkun/ReGoap/blob/master/ReGoap/Unity/GoapSensor.cs") 查看自定义Sensors的例子：[https://github.com/luxkun/ReGoap/tree/master/ReGoap/Unity/FSMExample/Sensors](https://github.com/luxkun/ReGoap/tree/master/ReGoap/Unity/FSMExample/Sensors "https://github.com/luxkun/ReGoap/tree/master/ReGoap/Unity/FSMExample/Sensors") 与前面一样，您必须通过继承ReGoapSensor或实现IReGoapSensor接口来实现自己的类。 注意：想要操作传感器时，请确保使用ReGoapMemoryAdvanced，因为基类（ReGoapSensor）不会检查和更新传感器。

### Debuggin

当然，要调试自己的AI代理，您可以使用自己喜欢的编辑器自行调试。 但是ReGoap对于Unity中的示例中有一个非常有用的调试器。 [https://github.com/luxkun/ReGoap/blob/master/ReGoap/Unity/Editor/ReGoapNodeEditor.cs](https://github.com/luxkun/ReGoap/blob/master/ReGoap/Unity/Editor/ReGoapNodeEditor.cs "https://github.com/luxkun/ReGoap/blob/master/ReGoap/Unity/Editor/ReGoapNodeEditor.cs") [https://github.com/luxkun/ReGoap/blob/master/ReGoap/Unity/Editor/ReGoapNodeBaseEditor.cs](https://github.com/luxkun/ReGoap/blob/master/ReGoap/Unity/Editor/ReGoapNodeBaseEditor.cs "https://github.com/luxkun/ReGoap/blob/master/ReGoap/Unity/Editor/ReGoapNodeBaseEditor.cs") 要使用它，只需单击Unity的菜单窗口，然后单击ReGoap Debugger，将打开一个Unity窗口，这是代理调试器。 现在，如果您单击场景中的任何AI代理（运行时，仅对正在运行的AI代理起作用），窗口将自动更新，让您知道特工的“想法”（当前世界状态，选择的目标和当前计划，可能的目标，可能采取的行动，可以做什么，什么不可以，请尝试！）。