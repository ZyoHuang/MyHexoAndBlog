---
title: Unity使用NavMeshPath实现NavMeshAgent匀速移动
date:
updated:
tags: Unity技术
categories:
  - - GamePlay
    - 实用工具
  - - 游戏引擎
    - Unity
keywords:
top_img:
cover: https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2020/04/10.gif
aplayer:
---
<meta name="referrer" content="no-referrer" />

## 前言
为了准备老师的课设作业，我决定做一个RPG小游戏，在处理寻路的时候遇到了点小麻烦。
首先是解决方案的选取，Unity自带的Navgation Mesh挺强大的（至少在客户端是这样，hh），我之前一直用的`A*`，但是不知道为什么老版本的`A*`在Unity 2019.3的InspectorGUI是坏掉的。。。又没钱买正版。。。
emmm，勉为其难的选择Navgation Mesh吧。
但是Navgation Mesh好像只有带有加速度的移动方式，这就有点蛋疼，说实话我个人挺不喜欢那种手感的，所以查了查官方文档，发现了NavMeshPath这么个东西，用它就可以得到我们一次寻路过程中所有的转折点，从而自己处理寻路，那么，我们开始吧。
## 思路
通过调用NavMeshAgent的CalculatePath来得到NavMeshPath，并使用NavMeshPath的corners数组（记录了每一处拐点的位置）和Transform.Translate做匀速运动，`人物方向的改变需要自己处理`。
## 代码
重要部分已注释，核心就是RunToTarget函数
```csharp
namespace ETModel
{
    [ObjectSystem]
    public class NavComponentAwakeSystem : AwakeSystem<NavComponent>
    {
        public override void Awake(NavComponent self)
        {
            //添加NavMeshAgent组件
            self.NavMeshAgent = self.Entity.GameObject.AddComponent<NavMeshAgent>();
            //取得获取用户输入组件，用于右键导航
            self.UserInputComponent = Game.Scene.GetComponent<UserInputComponent>();
            //取得相机组件，可获得MainCamera
            self.CameraComponent = Game.Scene.GetComponent<CameraComponent>();
            //缓存Transform组件，用于移动
            self.Transform = self.Entity.GameObject.transform;
        }
    }

    [ObjectSystem]
    public class NavComponentUpdateSystem : UpdateSystem<NavComponent>
    {
        public override void Update(NavComponent self)
        {
            //调用Update
            self.Update();
        }
    }

    public class NavComponent : Component
    {
        public NavMeshAgent NavMeshAgent;

        public UserInputComponent UserInputComponent;

        public CameraComponent CameraComponent;

        private Ray m_Ray;

        public Transform Transform;
        
        private NavMeshPath m_NavMeshPath = new NavMeshPath();

        /// <summary>
        /// 前一个路径点索引
        /// </summary>
        private int m_PreviousPathPointIndex = 0;
        
        /// <summary>
        /// 后一个路径点索引
        /// </summary>
        private int m_CurrentPathPointIndex = 1;

        /// <summary>
        /// 移动到目标点（NavMeshPath.corners第0个路径点是当前游戏物体所在位置，以此类推）
        /// </summary>
        public void RunToTarget()
        {
            //防止数组越界
            if (m_CurrentPathPointIndex > m_NavMeshPath.corners.Length - 1) return;

            //如果游戏物体坐标与当前路径点坐标距离小于0.1即可认为已抵达，可以向下一个路径点导航
            if ((Transform.position - m_NavMeshPath.corners[m_CurrentPathPointIndex]).magnitude <= 0.1f)
            {
                //递增路径点索引
                m_PreviousPathPointIndex++;
                m_CurrentPathPointIndex++;
                //防止数组越界
                if (m_CurrentPathPointIndex > m_NavMeshPath.corners.Length - 1)
                {
                    //处理动画切换，请无视
                    Entity.GetComponent<StackFsmComponent>().AddState(StateTypes.Idle, "Idle", 1);
                    return;
                }

                //处理人物转向，请无视
                Entity.GetComponent<TurnComponent>().Turn(m_NavMeshPath.corners[m_CurrentPathPointIndex]);
            }

            //匀速运动。计算出前一个路径点到当前路径点方向，然后移动
            Transform.Translate(
                ((-m_NavMeshPath.corners[m_PreviousPathPointIndex] +
                  m_NavMeshPath.corners[m_CurrentPathPointIndex]).normalized) *
                (Time.deltaTime * 8.0f), Space.World);
        }

        /// <summary>
        /// 这里是当游戏物体被对象池回收时会调用的逻辑，因为我们是动态添加的NavMeshAgent组件，所以我们还要把它移除
        /// </summary>
        public override void Dispose()
        {
            if (IsDisposed)
            {
                return;
            }

            base.Dispose();
            NavMeshAgent = null;
            UserInputComponent = null;
            UnityEngine.Object.Destroy(Entity.GameObject.GetComponent<NavMeshAgent>());
        }

        public void Update()
        {
            //当按下右键
            if (UserInputComponent.RightMouseDown)
            {
                m_Ray = CameraComponent.mainCamera.ScreenPointToRay(Input.mousePosition);
                if (Physics.Raycast(m_Ray, out var hit, 2000))
                {
                    //计算NavMeshPath
                    NavMeshAgent.CalculatePath(hit.point, m_NavMeshPath);
                    //初始化路径点索引
                    m_PreviousPathPointIndex = 0;
                    m_CurrentPathPointIndex = 1;
                    //处理人物转向，请无视
                    Entity.GetComponent<TurnComponent>().Turn(m_NavMeshPath.corners[1]);
                    //处理动画切换，请无视
                    Entity.GetComponent<StackFsmComponent>().AddState(StateTypes.Run, "Run", 1);
                }
            }

            //开始导航
            RunToTarget();
        }
    }
}
```
## 效果演示
![寻路效果演示](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2020/04/10.gif)
