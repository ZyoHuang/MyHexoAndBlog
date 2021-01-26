---
title: ET篇：ETVoid和void，ETTask和Task的区别与使用时机
date:
updated:
tags:
categories:
keywords:
top_img:
cover:
katex: true
aplayer:
---
<meta name="referrer" content="no-referrer" />

**在学习ET的过程中，我们经常看到ETVoid和ETTask的身影
比如**
```csharp
        /// <summary>
        /// 异步加载assetbundle
        /// </summary>
        /// <param name="assetBundleName"></param>
        /// <returns></returns>
        public async ETTask LoadBundleAsync(string assetBundleName)
        {
            assetBundleName = assetBundleName.ToLower();
            string[] dependencies = AssetBundleHelper.GetSortedDependencies(assetBundleName);
            // Log.Debug($"-----------dep load {assetBundleName} dep: {dependencies.ToList().ListToString()}");
            foreach (string dependency in dependencies)
            {
                if (string.IsNullOrEmpty(dependency))
                {
                    continue;
                }

                await this.LoadOneBundleAsync(dependency);
            }
        }
```
**再比如**
```csharp
	    public async ETVoid TestActor()
	    {
		    try
		    {
			    M2C_TestActorResponse response = (M2C_TestActorResponse)await SessionComponent.Instance.Session.Call(
						new C2M_TestActorRequest() { Info = "actor rpc request" });
			    Log.Info(response.Info);
			}
		    catch (Exception e)
		    {
				Log.Error(e);
		    }
		}
```
### 异同点
我们先不管他们代表什么意思，先找找他们的共同点和不同点
共同点：都有async修饰符
不同点：
- `ETTask可以有返回值,可等待返回结果，他会等待自身内部任务完成再执行下面的语句，通俗点说可以把异步方法变成同步那样的写法`
- `ETVoid不能有返回值，不可等待返回结果，不做特殊处理，它的状态是不可获得的，他不会等待自身内部任务完成再执行下面的语句`

## 那他们又有什么用呢？

### ETTask
具体来说就是你异步加载一个超级大的模型，要花5,6秒钟，而下面的操作是在模型已经加载完成的基础上进行的，
这个时候，一般情况下是要写回调函数的，然而，使用ETTask修饰这个加载模型的函数，调用的时候加上await关键字，就会等到模型加载完成再进行下面的操作，
换句话说，await本身可以看成一个空的回调函数，异步操作处理完了，才会执行await下面的语句
### ETVoid
比如你Map服务器从Gate服务器收到创建一个游戏物体的请求，就可以使用ETVoid
再比如你点击一个按钮，所触发的事件函数可以用ETVoid（emmm，貌似ETTask更合理一点？因为你要等他处理完这一次的事件再处理即将到来的事件）
因为你不用关心它的状态
总来说网络层使用ETVoid要比客户端正常的逻辑层频繁的多得多

## 举个具体的例子
### 正确做法
```csharp
// 这是一个初始化资源的操作，如果这些资源没有添加成功就执行相关操作，就会报空
public async ETTask Init() 
{
	TimerComponent timerComponent = ETModel.Game.Scene.GetComponent<TimerComponent>();
	// 等待5秒
	await timerComponent.WaitAsync(5000);
	await ETModel.Game.Scene.GetComponent<FUIPackageComponent>().AddPackageAsync(FUIPackage.FUILogin);
	await ETModel.Game.Scene.GetComponent<FUIPackageComponent>().AddPackageAsync(FUIPackage.FUILobby);
}

// 下面代码与Init在同一调用层级

// 函数调用，会等到Init中的语句全部执行完毕再执行下面一行代码
await Init();
// 这里不会报空，因为所有资源已经添加完毕
Game.EventSystem.Run(EventIdType.ShowLoginUI);
```
### 错误做法
```csharp
// 这是一个初始化资源的操作，如果这些资源没有添加成功就执行相关操作，就会报空
public async ETVoid Init() 
{
	TimerComponent timerComponent = ETModel.Game.Scene.GetComponent<TimerComponent>();
	// 等待5秒
	await timerComponent.WaitAsync(5000);
	await ETModel.Game.Scene.GetComponent<FUIPackageComponent>().AddPackageAsync(FUIPackage.FUILogin);
	await ETModel.Game.Scene.GetComponent<FUIPackageComponent>().AddPackageAsync(FUIPackage.FUILobby);
}

// 下面代码与Init在同一调用层级

// 函数调用，运行到第一个await之后就会返回，准确的说是运行到第一个返回的ETTask就返回，
// 然后时间组件会自己走那个5s倒计时
Init().Coroutine();
// 这里会报空，因为资源并没有添加完毕
Game.EventSystem.Run(EventIdType.ShowLoginUI);

```
## 总结
**最后，`ETTask是单线程的，没有开新线程`
`注意，如果在调用async修饰的方法（返回类型为ETTask）时不加await修饰符，这个方法将不会等待自身任务完成，也就是说，不会等待返回结果，几乎（执行到第一个await语句里的返回的ETTask就返回）是立马执行下面的代码了。`
最后的最后推荐大家几篇文章
[这是一篇讲异步和多线程的文章](https://blog.csdn.net/qq_27825451/article/details/78853119 "一篇讲异步和多线程的文章链接")
[这是一篇讲async和await的文章](https://blog.csdn.net/a462533587/article/details/82261468 "这是一篇讲async和await的文章")
[这是一篇讲async 的三大返回类型的文章](https://www.cnblogs.com/liqingwen/p/6218994.html?tdsourcetag=s_pcqq_aiomsg "这是一篇讲async 的三大返回类型的文章")
[这是一篇讲async/await实现原理的文章](https://blogs.msdn.microsoft.com/seteplia/2017/11/30/dissecting-the-async-methods-in-c/ "这是一篇讲async/await实现原理的文章")**
**然后是大家最喜欢的熊猫语录时间**
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/05/QQ截图20190505103623.png)
