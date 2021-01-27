---
title: NPBehave行为树架构
date:
updated:
tags: [GamePlay, AI, Behaviour Tree]
categories:
  - - GamePlay
    - AI
  - - GamePlay
    - 实用工具
keywords:
top_img:
cover: https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200801204710082.png
aplayer:
---
<meta name="referrer" content="no-referrer" />

前提概要

为了避免歧义，我修改了Node.cs文件中一些函数命名

```c#
DoStop->DoCancel
Stop->CancelWithoutReturnResult
```

我们都知道行为树中有三大组合节点，分别是

- *Selector*：选择组合器，一遇到子结点返回成功则其本身返回成功，否则继续执行下一个子结点，全部失败则其本身返回失败
- *Sequence*：序列组合器，一遇到子结点返回失败则其本身返回失败，否则继续执行下一个子结点。全部成功则其本身返回成功
- *Parallel*：并行组合器，全部子节点执行成功则其本身成功，有一个子结点执行失败，则终止其余子结点执行，其本身返回失败

### 架构流程图

NPBehave本身就是通过Start，DoStart，Stop，DoStop以及Stopped来控制整个行为树运转的，但是有一些函数命名容易引起起义，所以我做了修改

![image-20200801204710082](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200801204710082.png)

与生命周期相关的函数基本就这几个，最重要的，也就是最开始提到的会影响我们三个组合器运行状态结果的，就是*Stopped函数*

### 示例

举个例子，就以Selector为例

开始执行时，会开始处理子节点

```C#
        protected override void DoStart()
        {
            foreach (Node child in Children)
            {
                Debug.Assert(child.CurrentState==State.INACTIVE);
            }

            currentIndex = -1;

            ProcessChildren();
        }
		
		//处理子结点
        private void ProcessChildren()
        {
            //如果当前索引小于子结点个数
            if (++currentIndex < Children.Length)
            {
                //如果Selector需要被终止了（比如一个子节点返回了成功）
                if (IsStopRequested)
                {
                    //就表示此次Selector以失败告终，返回失败状态
                    Stopped(false);
                }
                else
                {
                    //否则就继续执行下一个子结点
                    Children[currentIndex].Start();
                }
            }
            else//如果当前索引等于了子结点个数，说明已经遍历完成，但是没有一个子结点返回成功，所以以失败告终，返回失败状态
            {
                Stopped(false);
            }
        }
```

在我们执行子结点的过程中，子结点可能会执行这一段

```c#
        protected override void DoStart()
        {
            if (this.action != null)
            {
                this.action.Invoke();
                this.Stopped(true);
            }
        }

        /// <summary>
        /// 节点被终止，内含状态，成功或失败
        /// </summary>
        /// <param name="success"></param>
        protected virtual void Stopped(bool success)
        {
            // Assert.AreNotEqual(this.currentState, State.INACTIVE, "The Node " + this + " called 'Stopped' while in state INACTIVE, something is wrong! PATH: " + GetPath());
            Debug.Assert(this.currentState != State.INACTIVE,
                "Called 'Stopped' while in state INACTIVE, something is wrong!");
            this.currentState = State.INACTIVE;
            if (this.ParentNode != null)
            {
                this.ParentNode.ChildStopped(this, success);
            }
        }

```

那么我们的Selector组合器就已经找到了返回成功的结点，其本身也将返回成功

```C#
        protected override void DoChildStopped(Node child, bool result)
        {
            if (result)
            {
                Stopped(true);
            }
            else
            {
                ProcessChildren();
            }
        }
```

### BlackBoradConditionNode规则

每次轮询的时候开始监测变化，并且进行一次条件检查，如果条件符合就执行装饰的节点，不符合就Stop

```cs
protected override void DoStart()
{
    if (stopsOnChange != Stops.NON
    {
        if (!isObserving)
        {
            isObserving = true;
            StartObserving();
        }
    }
    if (!IsConditionMet())
    {
        Stopped(false);
    }
    else
    {
        Decoratee.Start();
    }
}
```

每次订阅的黑板键改变的时候进行一次条件判断，如果当前节点正在处于激活态且条件不满足，就停止

如果处于失活状态且条件满足，就根据设置进行其他节点的终止操作，等待下一次轮询（下一帧）执行装饰的结点

```cs
protected void Evaluate()
{
    if (IsActive && !IsConditionMet())
    {
        if (stopsOnChange == Stops.SELF || stopsOnChange == Stops.BOTH || stopsOnChange == Stops.IMMEDIATE_RESTART)
        {
            // Debug.Log( this.key + " stopped self ");
            this.CancelWithoutReturnResult();
        }
    }
    else if (!IsActive && IsConditionMet())
    {
        if (stopsOnChange == Stops.LOWER_PRIORITY || stopsOnChange == Stops.BOTH || stopsOnChange == Stops.IMMEDIATE_RESTART ||
            stopsOnChange == Stops.LOWER_PRIORITY_IMMEDIATE_RESTART)
        {
            // Debug.Log( this.key + " stopped other ");
            Container parentNode = this.ParentNode;
            Node childNode = this;
            while (parentNode != null && !(parentNode is Composite))
            {
                childNode = parentNode;
                parentNode = parentNode.ParentNode;
            }
            Debug.Assert(parentNode != null, "NTBtrStops is only valid when attached to a parent composite");
            Debug.Assert(childNode != null);
            if (parentNode is Parallel)
            {
                Debug.Assert(stopsOnChange == Stops.IMMEDIATE_RESTART,
                    "On Parallel Nodes all children have the same priority, thus Stops.LOWER_PRIORITY or Stops.BOTH are unsupported in this context!");
            }
            if (stopsOnChange == Stops.IMMEDIATE_RESTART || stopsOnChange == Stops.LOWER_PRIORITY_IMMEDIATE_RESTART)
            {
                if (isObserving)
                {
                    isObserving = false;
                    StopObserving();
                }
            }
            ((Composite) parentNode).StopLowerPriorityChildrenForChild(childNode,
                stopsOnChange == Stops.IMMEDIATE_RESTART || stopsOnChange == Stops.LOWER_PRIORITY_IMMEDIATE_RESTART);
        }
    }
}
```



### 总结

在NPBehave中，真正会返回/包含状态的函数是Stopped，记住这一点，看代码的时候会轻松很多
