---
title: C#篇：剖析c#中的异步方法
tags: [编程语言, 文献翻译]
categories:
  - - 编程语言
    - CSharp
cover: https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/05/Async_sequence_state_machine_thumb.png
date: 2019-05-28 17:15:33
---

<meta name="referrer" content="no-referrer" />



### 本文翻译自

**[https://blogs.msdn.microsoft.com/seteplia/2017/11/30/dissecting-the-async-methods-in-c/](https://blogs.msdn.microsoft.com/seteplia/2017/11/30/dissecting-the-async-methods-in-c/)**

### 前言

**c#语言对于开发人员的工作效率非常好，我很高兴最近的努力使它更适合于高性能应用程序。 下面是一个例子:C#5引入了“async”方法。从用户的角度来看，该特性非常有用，因为它有助于将几个基于任务的操作组合成一个操作。但是这种抽象是有代价的。任务是引用类型，在创建它们的任何地方都会导致堆分配，即使在“async”方法同步完成的情况下也是如此。使用C#7，异步方法可以返回类似于任务的类型，比如`ValueTask`，以减少堆分配的数量，或者在某些场景中完全避免它们。 为了理解所有这些是如何实现的，我们需要深入了解异步方法是如何实现的。**

### 历史回顾

**但首先，让我们回顾一下历史。 类`Task`和`Task<T>`是在.Net 4.0中引入的，在我看来，在.Net中的异步和并行编程领域发生了巨大的思想转变。与以前的异步模式(如.Net 1.0中的BeginXXX/EndXXX模式，也称为“异步编程模型”)或基于事件的异步模式(如.Net 2.0中的BackgroundWorker类)不同，任务是可组合的。 任务表示一个工作单元，承诺在将来返回结果。这个承诺可以由IO-operation支持，也可以表示计算密集型的操作。没关系。重要的是，行动的结果是自给自足的，是一等公民（笑）。您可以传递一个`方法`:您可以将它存储在一个变量中，从一个方法返回它，或者将它传递给另一个方法。您可以将`两个方法合并（委托链）`在一起形成另一个，您可以同步等待结果，或者通过在方法中添加`continuation`来“等待”结果。仅仅使用task实例，您就可以决定如果操作成功、出现故障或被取消该怎么办。 任务并行库(TPL)改变了我们对并发性的看法，C#5语言通过引入`async/await`向前迈出了一步。`async/await`有助于组合任务，使用户能够使用熟悉的结构，如try/catch、using等。但是像其他抽象一样，异步/等待特性也有它的代价。要想知道成本是多少，你就得看看编译器底层的东西。**

### 异步方法内部机制

常规方法只有一个入口点和一个出口点(它可以有多个返回语句，但在运行时，给定调用只存在一个点)。但是异步方法和迭代器(具有返回的方法)是不同的。在异步方法的情况下，方法调用者几乎可以立即获得结果(例如Task或Task)，然后通过生成的任务“等待”方法的实际结果。 `我们将术语“async方法”定义为使用上下文关键字async标记的方法。这并不一定意味着方法是异步执行的。这也不意味着该方法是异步的。它只意味着编译器对方法执行一些特殊的转换。`

### 让我们考虑以下异步方法:

```csharp
class StockPrices
{
    private Dictionary<string, decimal> _stockPrices;
    public async Task<decimal> GetStockPriceForAsync(string companyId)
    {
        await InitializeMapIfNeededAsync();
        _stockPrices.TryGetValue(companyId, out var result);
        return result;
    }

    private async Task InitializeMapIfNeededAsync()
    {
        if (_stockPrices != null)
            return;

        await Task.Delay(42);
        // Getting the stock prices from the external source and cache in memory.
        _stockPrices = new Dictionary<string, decimal> { { "MSFT", 42 } };
    }
}
```

**方法GetStockPriceForAsync确保\_stockPrices字典已初始化，然后从缓存中获取值。 为了更好地理解编译器的功能，让我们尝试手工编写一个转换。**

### 手写异步方法

**TPL提供了两个主要构建块来帮助我们构造和连接任务:使用任务的任务延续。`ContinueWith`和`TaskCompletionSource<T>`类，用于手工构造任务。**

```csharp
class GetStockPriceForAsync_StateMachine
{
    enum State { Start, Step1, }
    private readonly StockPrices @this;
    private readonly string _companyId;
    private readonly TaskCompletionSource<decimal> _tcs;
    private Task _initializeMapIfNeededTask;
    private State _state = State.Start;

    public GetStockPriceForAsync_StateMachine(StockPrices @this, string companyId)
    {
        this.@this = @this;
        _companyId = companyId;
    }

    public void Start()
    {
        try
        {
            if (_state == State.Start)
            {
                // The code from the start of the method to the first 'await'.

                if (string.IsNullOrEmpty(_companyId))
                    throw new ArgumentNullException();

                _initializeMapIfNeededTask = @this.InitializeMapIfNeeded();

                // Update state and schedule continuation
                _state = State.Step1;
                _initializeMapIfNeededTask.ContinueWith(_ => Start());
            }
            else if (_state == State.Step1)
            {
                // Need to check the error and the cancel case first
                if (_initializeMapIfNeededTask.Status == TaskStatus.Canceled)
                    _tcs.SetCanceled();
                else if (_initializeMapIfNeededTask.Status == TaskStatus.Faulted)
                    _tcs.SetException(_initializeMapIfNeededTask.Exception.InnerException);
                else
                {
                    // The code between first await and the rest of the method

                    @this._store.TryGetValue(_companyId, out var result);
                    _tcs.SetResult(result);
                }
            }
        }
        catch (Exception e)
        {
            _tcs.SetException(e);
        }
    }

    public Task<decimal> Task => _tcs.Task;
}

public Task<decimal> GetStockPriceForAsync(string companyId)
{
    var stateMachine = new GetStockPriceForAsync_StateMachine(this, companyId);
    stateMachine.Start();
    return stateMachine.Task;
}
```

**代码冗长，但相对简单。所有来自`GetStockPriceForAsync`的逻辑都被移动到`GetStockPriceForAsync_StateMachine`。使用“延续传递样式”的Start方法。异步转换的一般算法是将原始方法在`await`边界处分割为多个块。第一个块是从方法开始到第一个await的代码。第二个块：从第一个await到第二个await。第三个块：从上面的代码到第三个块或者直到方法结束，以此类推:**

```csharp
// Step 1 来自生成的异步状态机:

if (string.IsNullOrEmpty(_companyId)) throw new ArgumentNullException();
_initializeMapIfNeededTask = @this.InitializeMapIfNeeded();
```

**现在，每个等待的任务都成为状态机的一个字段，Start方法将自己订阅为每个任务的延续:**

```csharp
_state = State.Step1;
_initializeMapIfNeededTask.ContinueWith(_ => Start());
```

**然后，当任务完成时，返回Start方法，并检查\_state字段，以了解我们所处的阶段。然后逻辑检查任务`是否成功完成`、`是否被取消`或`是否成功`。在后一种情况下，状态机向前移动并运行下一个代码块。当所有操作完成后，状态机将`TaskCompletionSource<T>`实例的结果设置为完成，从`GetStockPricesForAsync`返回的结果任务将其状态更改为完成**

```csharp
// The code between first await and the rest of the method

@this._stockPrices.TryGetValue(_companyId, out var result);
_tcs.SetResult(result); // The caller gets the result back
```

**这种“实现”有几个严重的缺点:** 1. 大量堆分配:1个分配给状态机，1个分配给TaskCompletionSource,1个分配给TaskCompletionSource中的任务，1个分配给延续委托。 2. 缺少“热路径优化”:如果等待的任务已经完成，就没有理由创建延续。 3. 缺乏可扩展性:实现与基于任务的类紧密耦合，这使得不可能与其他场景一起使用，比如等待其他类型或返回Task或Task之外的其他类型。 现在让我们看一下实际的异步机制，以了解如何解决这些问题。

### 真正的异步机制\[toc\]

**编译器对异步方法转换所采用的总体方法与上面提到的方法非常相似。为了得到想要的行为，编译器依赖于以下类型:** 1.生成的状态机，其作用类似于异步方法的堆栈框架，并包含来自原始异步方法的所有逻辑。 2.AsyncTaskMethodBuilder，它保存完成的任务(非常类似于TaskCompletionSource类型)，并管理状态机的状态转换。 3.TaskAwaiter，它封装了一个任务，并在需要时安排任务的延续。 4.调用IAsyncStateMachine的MoveNextRunner。在正确的执行上下文中使用MoveNextmethod。 `生成的状态机是处于调试模式的类和处于发布模式的结构。`所有其他类型(MoveNextRunner类除外)都在BCL中定义为struct。 编译器为状态机生成一个类似于d\_ 1的类型名。为了避免名称冲突，生成的名称包含无效的标识符字符，这些字符不能由用户定义或引用。但是为了简化下面所有示例，我将使用有效标识符，方法是用\_替换<和>字符，并使用更容易理解的名称。 原来的方法 原始的“异步”方法创建一个状态机实例，用捕获的状态初始化它(如果方法不是静态的，包括这个指针)，然后通过调用AsyncTaskMethodBuilder开始执行。从引用传递的状态机实例开始

```csharp
[AsyncStateMachine(typeof(_GetStockPriceForAsync_d__1))]
public Task<decimal> GetStockPriceFor(string companyId)
{
    _GetStockPriceForAsync_d__1 _GetStockPriceFor_d__;
    _GetStockPriceFor_d__.__this = this;
    _GetStockPriceFor_d__.companyId = companyId;
    _GetStockPriceFor_d__.__builder = AsyncTaskMethodBuilder<decimal>.Create();
    _GetStockPriceFor_d__.__state = -1;
    var __t__builder = _GetStockPriceFor_d__.__builder;
    __t__builder.Start<_GetStockPriceForAsync_d__1>(ref _GetStockPriceFor_d__);
    return _GetStockPriceFor_d__.__builder.Task;
}
```

**通过引用传递是一个重要的优化，因为状态机往往是相当大的结构体(>100字节)，通过引用传递它可以避免冗余的复制。 状态机**

```csharp
struct _GetStockPriceForAsync_d__1 : IAsyncStateMachine
{
    public StockPrices __this;
    public string companyId;
    public AsyncTaskMethodBuilder<decimal> __builder;
    public int __state;
    private TaskAwaiter __task1Awaiter;

    public void MoveNext()
    {
        decimal result;
        try
        {
            TaskAwaiter awaiter;
            if (__state != 0)
            {
                // State 1 of the generated state machine:
                if (string.IsNullOrEmpty(companyId))
                    throw new ArgumentNullException();

                awaiter = __this.InitializeLocalStoreIfNeededAsync().GetAwaiter();

                // Hot path optimization: if the task is completed,
                // the state machine automatically moves to the next step
                if (!awaiter.IsCompleted)
                {
                    __state = 0;
                    __task1Awaiter = awaiter;

                    // The following call will eventually cause boxing of the state machine.
                    __builder.AwaitUnsafeOnCompleted(ref awaiter, ref this);
                    return;
                }
            }
            else
            {
                awaiter = __task1Awaiter;
                __task1Awaiter = default(TaskAwaiter);
                __state = -1;
            }

            // GetResult returns void, but it'll throw if the awaited task failed.
            // This exception is catched later and changes the resulting task.
            awaiter.GetResult();
            __this._stocks.TryGetValue(companyId, out result);
        }
        catch (Exception exception)
        {
            // Final state: failure
            __state = -2;
            __builder.SetException(exception);
            return;
        }

        // Final state: success
        __state = -2;
        __builder.SetResult(result);
    }

    void IAsyncStateMachine.SetStateMachine(IAsyncStateMachine stateMachine)
    {
        __builder.SetStateMachine(stateMachine);
    }
}
```

**生成的状态机看起来很复杂，但在本质上，它非常类似于我们手工创建的状态机。 尽管状态机类似于手工制作的状态机，但它有几个非常重要的区别:**

#### 生成状态机与我们手工制作状态机区别

**1.热路径优化 与我们的简单方法不同，生成的状态机知道等待的任务可能已经完成。**

```csharp
awaiter = __this.InitializeLocalStoreIfNeededAsync().GetAwaiter();

// Hot path optimization: if the task is completed,
// the state machine automatically moves to the next step
if (!awaiter.IsCompleted)
{
    // Irrelevant stuff

    // The following call will eventually cause boxing of the state machine.
    __builder.AwaitUnsafeOnCompleted(ref awaiter, ref this);
    return;
}
```

**如果等待的任务已经完成(成功与否)，状态机将进入下一步:**

```csharp
// GetResult returns void, but it'll throw if the awaited task failed.
// This exception is catched later and changes the resulting task.
awaiter.GetResult();
__this._stocks.TryGetValue(companyId, out result);
```

这意味着，如果所有等待的任务都已经完成，那么整个状态机将留在堆栈上。即使在今天，如果所有等待的任务已经完成或将同步完成，异步方法的内存开销也可能非常小。剩下的唯一分配将是任务本身! 2.错误处理 没有什么特殊的逻辑来覆盖所等待任务的故障或取消状态。状态机调用一个awaiter.getresult()，如果任务被取消，它将抛出TaskCancelledException，如果任务失败，则抛出另一个异常。这是一个很好的解决方案，因为GetResult()在错误处理方面与task.Wait()或task.Result稍有不同。 wait()和task。即使只有一个异常导致任务失败，也要抛出AggregateException。原因很简单:任务不仅可以表示通常只有一次失败的io绑定操作，还可以表示并行计算的结果。在后一种情况下，操作可能有多个错误，AggregateException被设计为在一个地方携带所有这些错误。 但是async/await模式是专门为异步操作设计的，异步操作通常最多有一个错误。因此，该语言的作者决定，如果awaiter.GetResult()将“打开”AggregateException并抛出第一个失败，则更有意义。这个设计决策并不完美，在下一篇文章中，我们将看到这个抽象什么时候会泄漏。 异步状态机只表示拼图的一部分。要了解整个情况，我们需要知道状态机实例如何与 TaskAwaiter和AsyncTaskMethodBuilder 交互。 [![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/05/Async_sequence_state_machine_thumb.png)](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/05/Async_sequence_state_machine_thumb.png) **图表看起来过于复杂，但每一块都设计得很好，发挥着重要作用。最有趣的协作发生在一个等待的任务没有完成的时候(图中用棕色矩形标记): 状态机调用\_builder。`AwaitUnsafeOnCompleted`(ref awaiter, ref this);将自己注册为任务的延续。 构建器确保任务完成时使用`IAsyncStateMachine`。调用`MoveNext`方法: 构建器捕获当前ExecutionContext并创建MoveNextRunner实例来将其与当前状态机实例关联。然后它从MoveNextRunner创建一个Action实例。运行它将在捕获的执行上下文中向前移动状态机。 构建器调用TaskAwaiter.UnsafeOnCompleted(action)，它将给定的操作调度为等待任务的延续。 当等待的任务完成时，调用给定的回调函数，状态机运行异步方法的下一个代码块。 执行上下文 有人可能会问:执行上下文是什么，为什么我们需要这么复杂的东西? 在同步世界中，每个线程都将环境信息保存在线程本地存储中。它可以是与安全相关的信息、特定于文化的数据或其他东西。当在一个线程中依次调用3个方法时，这些信息在所有方法之间自然地流动。但是对于异步方法不再是这样了。异步方法的每个“部分”都可以在不同的线程中执行，这使得线程本地信息不可用。 执行上下文保存一个逻辑控制流的信息，即使它跨越多个线程。 方法类似的任务。或ThreadPool运行。QueueUserWorkItem自动执行此操作。的任务。Run方法从调用线程捕获ExecutionContext，并将其与任务实例一起存储。当与任务关联的任务调度程序运行给定的委托时，它通过ExecutionContext运行委托。使用存储的上下文运行。 我们可以使用AsyncLocal来演示这个概念:**

```csharp
static Task ExecutionContextInAction()
{
    var li = new AsyncLocal<int>();
    li.Value = 42;

    return Task.Run(() =>
    {
        // Task.Run restores the execution context
        Console.WriteLine("In Task.Run: " + li.Value);
    }).ContinueWith(_ =>
    {
        // The continuation restores the execution context as well
        Console.WriteLine("In Task.ContinueWith: " + li.Value);
    });
}
```

在这些情况下，执行上下文通过Task.Run然后Task.ContinueWith方法。如果你运行这个方法，你会看到 In Task.Run: 42 In Task.ContinueWith: 42 但并不是BCL中的所有方法都会自动捕获并恢复执行上下文。两个例外是TaskAwaiter.UnsafeOnComplete和AsyncMethodBuilder.AwaitUnsafeOnComplete。看起来很奇怪，语言作者决定使用AsyncMethodBuilder和movenextr手动添加“不安全”方法来轮流执行上下文，而不是依赖于像AwaitTaskContinuation这样的内置工具。我怀疑现有实现存在一些性能原因或其他限制。 下面是一个例子来说明两者的区别:

```csharp
static async Task ExecutionContextInAsyncMethod()
        {
            var li = new AsyncLocal<int>();
            li.Value = 42;
            await Task.Delay(42);

            // The context is implicitely captured. li.Value is 42
            Console.WriteLine("After first await: " + li.Value);

            var tsk2 = Task.Yield();
            tsk2.GetAwaiter().UnsafeOnCompleted(() =>
            {
                // The context is not captured: li.Value is 0
                Console.WriteLine("Inside UnsafeOnCompleted: " + li.Value);
            });

            await tsk2;

            // The context is captured: li.Value is 42
            Console.WriteLine("After second await: " + li.Value);
        }
```

输出为 After first await: 42 Inside UnsafeOnCompleted: 0 After second await: 42

### 结论

**`异步方法与同步方法非常不同。 编译器为每个方法生成一个状态机，并将原始方法的所有逻辑移到那里。 生成的代码针对同步场景进行了高度优化:如果所有等待的任务都完成了，那么异步方法的开销就很小。 如果等待的任务没有完成，则逻辑依赖于许多helper类型来完成任务。`**