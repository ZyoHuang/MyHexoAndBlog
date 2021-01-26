---
title: C#篇：C#中异步方法的性能特征
tags: []
id: '1277'
categories:
  - - C#
  - - c
    - 数据结构
date: 2019-05-28 18:21:14
---

<meta name="referrer" content="no-referrer" />



## 本文翻译自

**[https://blogs.msdn.microsoft.com/seteplia/2018/01/25/the-performance-characteristics-of-async-methods/](https://blogs.msdn.microsoft.com/seteplia/2018/01/25/the-performance-characteristics-of-async-methods/)** 在前两篇博客文章中，我们讨论了c#中异步方法的内部原理，然后讨论了c#编译器为调整异步方法的行为提供的扩展点。今天我们将探讨异步方法的性能特征。 正如您在本系列的第一篇文章中已经知道的，编译器做了很多转换，使异步编程体验与同步编程非常相似。但要做到这一点，编译器创建一个状态机实例，将它传递给一个异步方法构建器，调用task awaiter等。显然，所有这些逻辑都有自己的代价，但我们要付出多少代价呢? 在tpl之前，异步操作通常是相当粗粒度的，因此异步操作的开销可能可以忽略不计。但今天，即使是相对简单的应用程序每秒也可能有数百甚至数千次异步操作。TPL的设计考虑到了这种工作负载，但它并不神奇，它有一些开销。 要度量异步方法的开销，将使用我们在第一篇博客文章中使用的稍微修改过的示例。

```csharp
public class StockPrices
{
    private const int Count = 100;
    private List<(string name, decimal price)> _stockPricesCache;

    // Async version
    public async Task<decimal> GetStockPriceForAsync(string companyId)
    {
        await InitializeMapIfNeededAsync();
        return DoGetPriceFromCache(companyId);
    }

    // Sync version that calls async init
    public decimal GetStockPriceFor(string companyId)
    {
        InitializeMapIfNeededAsync().GetAwaiter().GetResult();
        return DoGetPriceFromCache(companyId);
    }

    // Purely sync version
    public decimal GetPriceFromCacheFor(string companyId)
    {
        InitializeMapIfNeeded();
        return DoGetPriceFromCache(companyId);
    }

    private decimal DoGetPriceFromCache(string name)
    {
        foreach (var kvp in _stockPricesCache)
        {
            if (kvp.name == name)
            {
                return kvp.price;
            }
        }

        throw new InvalidOperationException($"Can't find price for '{name}'.");
    }

    [MethodImpl(MethodImplOptions.NoInlining)]
    private void InitializeMapIfNeeded()
    {
        // Similar initialization logic.
    }

    private async Task InitializeMapIfNeededAsync()
    {
        if (_stockPricesCache != null)
        {
            return;
        }

        await Task.Delay(42);

        // Getting the stock prices from the external source.
        // Generate 1000 items to make cache hit somewhat expensive
        _stockPricesCache = Enumerable.Range(1, Count)
            .Select(n => (name: n.ToString(), price: (decimal)n))
            .ToList();
        _stockPricesCache.Add((name: "MSFT", price: 42));
    }
}
```

StockPrices类使用来自外部源的股票价格填充缓存，并提供一个API来查询它。与第一篇文章示例的主要区别在于从字典切换到价格列表。为了度量不同形式的异步方法与同步方法的开销，操作本身至少应该做一些工作，并对股票价格模型进行线性搜索。 GetPricesFromCache故意使用普通循环构建，以避免任何分配。

### 同步版本对比

#### 基于任务的异步版本

在第一个基准测试中，我们比较了调用异步初始化方法的异步方法(GetStockPriceForAsync)、调用异步初始化方法的同步方法(GetStockPriceFor)和调用同步初始化方法的同步方法。

```csharp
private readonly StockPrices _stockPrices = new StockPrices();

public SyncVsAsyncBenchmark()
{
    // Warming up the cache
    _stockPrices.GetStockPriceForAsync("MSFT").GetAwaiter().GetResult();
}

[Benchmark]
public decimal GetPricesDirectlyFromCache()
{
    return _stockPrices.GetPriceFromCacheFor("MSFT");
}

[Benchmark(Baseline = true)]
public decimal GetStockPriceFor()
{
    return _stockPrices.GetStockPriceFor("MSFT");
}

[Benchmark]
public decimal GetStockPriceForAsync()
{
    return _stockPrices.GetStockPriceForAsync("MSFT").GetAwaiter().GetResult();
}
```

The results are:

Method

Mean

Scaled

Gen 0

Allocated

GetPricesDirectlyFromCache

2.177 us

0.96

\-

0 B

GetStockPriceFor

2.268 us

1.00

\-

0 B

GetStockPriceForAsync

2.523 us

1.11

0.0267

88 B

这些数据已经非常有趣: 异步方法相当快。GetPricesForAsync在这个基准测试中同步完成，它比纯同步方法慢15%。 (调用异步InitializeMapIfNeededAsync方法的同步GetPricesFor方法具有更低的开销，但最令人惊讶的是它根本没有分配(上表中分配的列对于GetPricesDirectlyFromCache和GetStockPriceFor都是0)。 当然，对于所有可能的情况，您不能说异步方法同步运行时异步机制的开销是15%。百分比与方法所做的工作量非常相关。测量异步方法(不做任何事)和同步方法(不做任何事)的纯方法调用开销将显示出巨大的差异。这个基准测试的目的是表明，做相对少量工作的异步方法的开销是适度的。 调用InitializeMapIfNeededAsync怎么可能完全没有分配呢?我在本系列的第一篇文章中已经提到，异步方法必须在托管头中分配至少一个对象—任务实例本身。让我们来探索这方面。

### 优化

#### 1。如果可能，缓存任务实例

前一个问题的答案非常简单:AsyncMethodBuilder为每个成功完成的异步操作使用一个任务实例。返回任务的async方法依赖于AsyncMethodBuilder，它在SetResult方法中具有以下逻辑:

```csharp
// AsyncMethodBuilder.cs from mscorlib
public void SetResult()
{
    // I.e. the resulting task for all successfully completed
    // methods is the same -- s_cachedCompleted.

    m_builder.SetResult(s_cachedCompleted);
}
```

SetResult方法只针对成功完成任务的异步方法调用，并且可以轻松共享每个基于任务的方法的成功结果。我们甚至可以通过下面的测试来观察这种行为:

```csharp
[Test]
public void AsyncVoidBuilderCachesResultingTask()
{
    var t1 = Foo();
    var t2 = Foo();

    Assert.AreSame(t1, t2);

    async Task Foo() { }
}
```

但这不是唯一可能发生的优化。AsyncTaskMethodBuilder做了类似的优化:它缓存任务和其他一些基本类型的任务。例如，它缓存了一系列整数类型的所有默认值，并为Task提供了一个特殊的缓存，用于\[-1;9)(更多信息请参见AsyncTaskMethodBuilder.gettaskforresult())。 下面的测试证明确实如此:

```csharp
[Test]
public void AsyncTaskBuilderCachesResultingTask()
{
    // These values are cached
    Assert.AreSame(Foo(-1), Foo(-1));
    Assert.AreSame(Foo(8), Foo(8));

    // But these are not
    Assert.AreNotSame(Foo(9), Foo(9));
    Assert.AreNotSame(Foo(int.MaxValue), Foo(int.MaxValue));

    async Task<int> Foo(int n) => n;
}
```

您不应该过分依赖这种行为，但是您应该知道，语言和框架的作者尽了最大的努力以各种可能的方式微调性能。缓存任务是一种常见的优化模式，也用于其他地方。例如，corefx repo中的新套接字实现严重依赖于这种优化，并尽可能使用缓存的任务。

#### 2:使用ValueTask

上面提到的优化只在少数情况下有效。我们可以使用ValueTask(\*\*)代替依赖它:一个特殊的类似于任务的值类型，如果方法同步完成，它将不会分配。 ValueTask实际上是T和Task的区别结合:如果完成了“value Task”，那么将使用底层值。如果底层承诺尚未完成，则将分配任务。 当操作同步完成时，这种特殊类型有助于避免不必要的堆分配。要使用ValueTask，我们只需要将GetStockPriceForAsync的返回类型从Task<decimal更改为ValueTask:

```csharp
public async ValueTask<decimal> GetStockPriceForAsync(string companyId)
{
    await InitializeMapIfNeededAsync();
    return DoGetPriceFromCache(companyId);
}
```

现在我们可以用这个额外的基准来衡量它们之间的差异:\[toc\]

```csharp
[Benchmark]
public decimal GetStockPriceWithValueTaskAsync_Await()
{
    return _stockPricesThatYield.GetStockPriceValueTaskForAsync("MSFT").GetAwaiter().GetResult();
}
```

结果如下:

Method

Mean

Scaled

Gen 0

Allocated

GetPricesDirectlyFromCache

1.260 us

0.90

\-

0B

GetStockPriceFor

1.399 us

1.00

\-

0B

GetStockPriceForAsync

1.552 us

1.11

0.0267

88B

GetStockPriceWithValueTaskAsync

1.519 us

1.09

\-

0B

正如您可能看到的，基于ValueTask的版本只是比基于Task的版本快一点。主要的区别是缺少堆分配。我们稍后将讨论是否值得进行这种切换，但在此之前，我想讨论一个棘手的优化问题。

#### 3.避免在公共路径上使用异步机制

如果您有一个使用非常广泛的异步方法，并且希望进一步减少开销，您可以考虑以下优化:您可以删除async修饰符，检查方法中的任务状态，并同步执行整个操作，而根本不需要处理异步机制。 听起来复杂吗?让我们来看看这个例子。

```csharp
public ValueTask<decimal> GetStockPriceWithValueTaskAsync_Optimized(string companyId)
{
    var task = InitializeMapIfNeededAsync();

    // Optimizing for acommon case: no async machinery involved.
    if (task.IsCompleted)
    {
        return new ValueTask<decimal>(DoGetPriceFromCache(companyId));
    }

    return DoGetStockPricesForAsync(task, companyId);

    async ValueTask<decimal> DoGetStockPricesForAsync(Task initializeTask, string localCompanyId)
    {
        await initializeTask;
        return DoGetPriceFromCache(localCompanyId);
    }
}
```

在本例中，方法getstockpricewithvaluetaskasync\_没有async修饰符，当它从InitializeMapIfNeededAsyncmethod获取任务时，它会检查任务是否已经完成。如果任务已经完成，它只调用DoGetPriceFromCache来立即获得结果。但如果初始化任务仍在运行，则调用本地函数等待结果。 使用局部函数不是唯一的选择，而是最简单的选择之一。但有一个警告。局部函数最自然的实现将捕获一个封闭状态:局部变量和参数:

```csharp
public ValueTask<decimal> GetStockPriceWithValueTaskAsync_Optimized2(string companyId)
{
    // Oops! This will lead to a closure allocation at the beginning of the method!
    var task = InitializeMapIfNeededAsync();

    // Optimizing for acommon case: no async machinery involved.
    if (task.IsCompleted)
    {
        return new ValueTask<decimal>(DoGetPriceFromCache(companyId));
    }

    return DoGetStockPricesForAsync();

    async ValueTask<decimal> DoGetStockPricesForAsync()
    {
        await task;
        return DoGetPriceFromCache(companyId);
    }
}
```

但不幸的是，由于编译器错误，即使方法在公共路径上完成，这段代码也会分配一个闭包。下面是这个方法的内幕:

```csharp
public ValueTask<decimal> GetStockPriceWithValueTaskAsync_Optimized(string companyId)
{
    var closure = new __DisplayClass0_0()
    {
        __this = this,
        companyId = companyId,
        task = InitializeMapIfNeededAsync()
    };

    if (closure.task.IsCompleted)
    {
        return ...
    }

    // The rest of the code
}
```

正如我们在“剖析c#中的局部函数”中讨论的，编译器对给定范围内的所有局部变量/参数使用共享闭包实例。因此，这种代码生成类型是有意义的，但它使整个堆分配斗争变得毫无意义。 提示:这种优化非常棘手。这样做的好处非常小，即使您正确地编写了原始的本地函数，您也可以在将来轻松地进行更改，并意外地捕获导致堆分配的封闭状态。如果您使用的是一个高度可重用的库，比如BCL，那么您仍然可以使用优化，方法肯定会在热路径上使用。 等待任务的开销 到目前为止，我们只讨论了一种特定的情况:同步完成的异步方法的开销。这是故意的。异步方法越“小”，它的总体性能开销就越明显。细粒度异步方法的工作量更少，并且更经常以同步方式完成。我们倾向于更频繁地给他们打电话。 但是，当一个方法“等待”一个未完成的任务时，我们应该知道异步机制的开销。为了测量这个开销，我们改变InitializeMapIfNeededAsync来调用Task.Yield()，即使缓存已经初始化:

```csharp
private async Task InitializeMapIfNeededAsync()
{
    if (_stockPricesCache != null)
    {
        await Task.Yield();
        return;
    }

    // Old initialization logic
}
```

让我们用以下方法扩展我们的性能基准测试套件:

```csharp
[Benchmark]
public decimal GetStockPriceFor_Await()
{
    return _stockPricesThatYield.GetStockPriceFor("MSFT");
}

[Benchmark]
public decimal GetStockPriceForAsync_Await()
{
    return _stockPricesThatYield.GetStockPriceForAsync("MSFT").GetAwaiter().GetResult();
}

[Benchmark]
public decimal GetStockPriceWithValueTaskAsync_Await()
{
    return _stockPricesThatYield.GetStockPriceValueTaskForAsync("MSFT").GetAwaiter().GetResult();
}
```

结果如下

Method

Mean

Scaled

Gen 0

Gen 1

Allocated

GetStockPriceFor

2.332 us

1.00

\-

\-

0 B

GetStockPriceForAsync

2.505 us

1.07

0.0267

\-

88 B

GetStockPriceWithValueTaskAsync

2.625 us

1.13

\-

\-

0 B

GetStockPriceFor\_Await

6.441 us

2.76

0.0839

0.0076

296 B

GetStockPriceForAsync\_Await

10.439 us

4.48

0.1577

0.0122

553 B

GetStockPriceWithValueTaskAsync\_Await

10.455 us

4.48

0.1678

0.0153

577 B

正如我们所看到的，无论是在速度还是内存方面，差异都更加明显。下面是对结果的简短解释。 对于未完成的任务，每个“wait”操作大约需要4us，每次调用分配近300B。这解释了为什么GetStockPriceFor的速度几乎是GetStockPriceForAsync的两倍，以及为什么它分配的内存更少。 当方法没有同步完成时，基于值的异步方法要比基于任务的异步方法慢一些。基于ValueTask方法的状态机需要比基于任务方法的状态机保存更多的数据。 它取决于平台(x64 vs. x86)和异步方法的一些局部变量/参数。 异步方法性能101 如果异步方法以同步方式完成，则性能开销相当小。 如果异步方法同步完成，则会发生以下内存开销:对于异步任务方法没有开销，对于异步任务方法，每个操作的开销为88字节(在x64平台上)。 ValueTask可以为同步完成的异步方法消除上述开销。 如果基于ValueTask的异步方法是同步完成的，那么它要比基于任务的异步方法快一点，否则会慢一点。 等待未完成任务的异步方法的性能开销要大得多(x64平台上的每个操作约300字节)。 和往常一样，先衡量。如果发现异步操作导致性能问题，可以从Task切换到ValueTask，缓存任务，或者尽可能使公共执行路径同步。但是您也可以尝试使异步操作的粒度更粗。这可以提高性能，简化调试，并使代码更容易推理。并不是每一小段代码都必须是异步的。