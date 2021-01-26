---
title: C#篇：扩展C#中的异步方法
tags: []
id: '1273'
categories:
  - - C#
  - - c
    - 数据结构
date: 2019-05-28 17:58:20
---

<meta name="referrer" content="no-referrer" />



## 本文翻译自

**[https://blogs.msdn.microsoft.com/seteplia/2018/01/11/extending-the-async-methods-in-c/](https://blogs.msdn.microsoft.com/seteplia/2018/01/11/extending-the-async-methods-in-c/)** **在前一篇博客文章中，我们讨论了c#编译器如何转换异步方法。在这篇文章中，我们将关注c#编译器为定制异步方法的行为提供的扩展点。 有三种方法可以控制异步方法的机制: 1.在System.Runtime.CompilerServices中提供您自己的async方法构建器。 2.使用自定义task awaiter。 3.定义自己的任务类型。**

### 自定义在System.Runtime.CompilerServices命名空间的类型\[toc\]

**从上一篇文章中我们知道，c#编译器将异步方法转换成一个生成的状态机，它依赖于一些预定义的类型。但是c#编译器并不期望这些已知的类型来自特定的程序集。例如，您可以在项目中提供自己的AsyncVoidMethodBuilder实现，c#编译器将“绑定”异步机制到您的自定义类型。 这是一个很好的方法来探索底层的转换是什么，看看在运行时发生了什么:**

```csharp
namespace System.Runtime.CompilerServices
{
    // AsyncVoidMethodBuilder.cs in your project
    public class AsyncVoidMethodBuilder
    {
        public AsyncVoidMethodBuilder()
            => Console.WriteLine(".ctor");

        public static AsyncVoidMethodBuilder Create()
            => new AsyncVoidMethodBuilder();

        public void SetResult() => Console.WriteLine("SetResult");

        public void Start<TStateMachine>(ref TStateMachine stateMachine)
            where TStateMachine : IAsyncStateMachine
        {
            Console.WriteLine("Start");
            stateMachine.MoveNext();
        }

        // AwaitOnCompleted, AwaitUnsafeOnCompleted, SetException 
        // and SetStateMachine are empty
    }   
}
```

**现在，项目中的每个async方法都将使用AsyncVoidMethodBuilder的自定义版本。我们可以测试这与一个简单的异步方法:**

```csharp
[Test]
public void RunAsyncVoid()
{
    Console.WriteLine("Before VoidAsync");
    VoidAsync();
    Console.WriteLine("After VoidAsync");

    async void VoidAsync() { }
}
```

**Test输出如下 Before VoidAsync .ctor Start SetResult After VoidAsync** 您可以实现UnsafeAwaitOnComplete方法来测试异步方法的行为，并使用await子句返回未完成的任务。完整的例子可以在github上找到。 要更改async Task和_async Task_方法的行为，您应该提供自己版本的AsyncTaskMethodBuilder和AsyncTaskMethodBuilder 关于这些类型的完整示例可以在我的github项目中找到，该项目名为 [EduAsync](https://github.com/SergeyTeplyakov/EduAsync "EduAsync")， 分别位于[AsyncTaskBuilder.cs](https://github.com/SergeyTeplyakov/EduAsync/blob/master/src/02_AsyncTaskBuilder/AsyncTaskMethodBuilder.cs "AsyncTaskBuilder.cs")和[AsyncTaskMethodBuilderOfT.cs](https://github.com/SergeyTeplyakov/EduAsync/blob/master/src/03_AsyncTaskBuilderOfT/AsyncTaskMethodBuilderOfT.cs "AsyncTaskMethodBuilderOfT.cs")中。 感谢Jon Skeet对这个项目的启发。这确实是深入学习异步机制的好方法。

### 自定义awaiters

**前面的示例是闹着玩的，不适合正式项目。我们可以通过这种方式学习异步机制，但是您肯定不希望在代码库中看到这样的代码。c#语言的作者在编译器中内置了适当的扩展点，允许在异步方法中“等待”不同的类型。 为了使类型成为“awaitable”(即在await表达式上下文中有效)，类型应该遵循一种特殊的模式: 编译器应该能够找到一个名为GetAwaiter的实例或扩展方法。该方法的返回类型应符合一定的要求: 该类型应该实现INotifyCompletion接口。 类型应该具有bool IsCompleted {get;}属性和T GetResult()方法。** 这意味着我们可以很容易地使Lazy可选:

```csharp
public struct LazyAwaiter<T> : INotifyCompletion
{
    private readonly Lazy<T> _lazy;

    public LazyAwaiter(Lazy<T> lazy) => _lazy = lazy;

    public T GetResult() => _lazy.Value;

    public bool IsCompleted => true;

    public void OnCompleted(Action continuation) { }
}

public static class LazyAwaiterExtensions
{
    public static LazyAwaiter<T> GetAwaiter<T>(this Lazy<T> lazy)
    {
        return new LazyAwaiter<T>(lazy);
    }
}
```

```csharp
public static async Task Foo()
{
    var lazy = new Lazy<int>(() => 42);
    var result = await lazy;
    Console.WriteLine(result);
}
```

这个例子可能看起来太做作了，但是这个扩展点实际上非常有用，并且大量应用。例如`Reactive Extensions for .NET`提供了一个定制的awaiter，用于在异步方法中等待IObservable实例。BCL本身有YieldAwaitable被Task.Yield和HopToThreadPoolAwaitable使用:

```csharp
public struct HopToThreadPoolAwaitable : INotifyCompletion
{
    public HopToThreadPoolAwaitable GetAwaiter() => this;
    public bool IsCompleted => false;

    public void OnCompleted(Action continuation) => Task.Run(continuation);
    public void GetResult() { }
}
```

**下面的单元测试演示了最后一个正在运行的awaiter:**

```csharp
[Test]
public async Task Test()
{
    var testThreadId = Thread.CurrentThread.ManagedThreadId;
    await Sample();

    async Task Sample()
    {
        Assert.AreEqual(Thread.CurrentThread.ManagedThreadId, testThreadId);

        await default(HopToThreadPoolAwaitable);
        Assert.AreNotEqual(Thread.CurrentThread.ManagedThreadId, testThreadId);
    }
}
```

**`任何“async”方法的第一部分(在await语句之前)都是同步运行的。`在大多数情况下，这对于急切的参数验证是很好的，也是可取的，但有时我们希望确保方法主体不会阻塞调用者的线程。`HopToThreadPoolAwaitable`确保方法的其余部分在线程池线程中执行，而不是在调用者的线程中执行。**

### 任务类型

这个扩展点非常有用，但是有限，因为所有的异步方法都应该返回void, Task或者Task。从C# 7.2开始，编译器支持任务类型。 Task-like是一个类或结构，它具有AsyncMethodBuilderAttribute(\*\*)标识的关联构建器类型。为了使类似于任务的类型有用，它应该以我们在前一节中描述的方式可用。基本上，task-like类型结合了前面描述的前两个扩展点，实现了官方支持的第一种方式。 今天，您必须自己定义这个属性。示例可以在我的[github repo](https://github.com/SergeyTeplyakov/EduAsync/blob/master/src/07_CustomTaskLikeTypes/AsyncMethodBuilder.cs#L9 "github repo")中找到。 下面是一个定义为struct的自定义类任务类型的简单示例:

```csharp
public sealed class TaskLikeMethodBuilder
{
    public TaskLikeMethodBuilder()
        => Console.WriteLine(".ctor");

    public static TaskLikeMethodBuilder Create()
        => new TaskLikeMethodBuilder();

    public void SetResult() => Console.WriteLine("SetResult");

    public void Start<TStateMachine>(ref TStateMachine stateMachine)
        where TStateMachine : IAsyncStateMachine
    {
        Console.WriteLine("Start");
        stateMachine.MoveNext();
    }

    public TaskLike Task => default(TaskLike);

    // AwaitOnCompleted, AwaitUnsafeOnCompleted, SetException 
    // and SetStateMachine are empty

}

[System.Runtime.CompilerServices.AsyncMethodBuilder(typeof(TaskLikeMethodBuilder))]
public struct TaskLike
{
    public TaskLikeAwaiter GetAwaiter() => default(TaskLikeAwaiter);
}

public struct TaskLikeAwaiter : INotifyCompletion
{
    public void GetResult() { }

    public bool IsCompleted => true;

    public void OnCompleted(Action continuation) { }
}
```

**现在我们可以定义一个返回类任务类型的方法，甚至在方法体中使用不同的类任务类型:**

```csharp
public async TaskLike FooAsync()
{
    await Task.Yield();
    await default(TaskLike);
}
```

**具有类似任务类型的主要原因是能够减少异步操作的开销。每个返回Task的异步操作都会在托管堆中分配至少一个对象——任务本身。这对于绝大多数应用程序来说都是非常好的，特别是当它们处理粗粒度异步操作时。但是，对于每秒可以跨越数千个小任务的基础结构级代码来说，情况并非如此。对于这种场景，减少每次调用的分配可以合理地提高性能。**

### 异步模式可扩展性

**1.c#编译器为扩展异步方法提供了多种方法。 2.您可以通过提供自己版本的AsyncTaskMethodBuilder类型来更改现有基于任务的异步方法的行为。 3.您可以通过实现“awaitable模式”使类型“awaitable”。 4.从c# 7开始，您可以构建自己的类似于任务的类型。**

### 额外的引用

剖析c#中的异步方法 EduAsync repo在github上。

### 鸽子

**下次我们将讨论异步方法的perf特性，并将看到最新的类任务值类型如何调用System.ValueTask会影响性能。**