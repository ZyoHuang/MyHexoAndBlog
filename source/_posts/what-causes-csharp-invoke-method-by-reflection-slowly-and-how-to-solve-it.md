---
title: （译）C#的反射为什么慢？怎么加快反射调用？
tags: []
id: '2576'
categories:
  - - C#
date: 2020-02-05 22:05:53
---

<meta name="referrer" content="no-referrer" />



# 前言

我们知道C#反射慢，但是当中很多人不知道它为什么慢，并且如何解决反射调用方法慢的问题呢？ 这篇文章会给你一个答案。本文译自：[https://mattwarren.org/2016/12/14/Why-is-Reflection-slow/](https://mattwarren.org/2016/12/14/Why-is-Reflection-slow/ "https://mattwarren.org/2016/12/14/Why-is-Reflection-slow/")

# C#的反射为什么慢

## 反射的设计初衷

*   在`运行时`非常快的访问我们所需要的代码的信息。
*   在`编译时`非常直接的访问生成代码所需的信息。
*   垃圾回收器/计算堆栈能够在不对程序加锁/分配内存的情况下访问必要的信息。
*   能极大减少一次性需要加载的类型数量。
*   能极大减少给定类型加载时所需要加载的额外类型数目。
*   类型系统数据结构必须在NGEN映像中是可存储的。

我们可以看到，它只强调了最少依赖加载，并`没有说我们可以直接从元数据获取所有CLR数据类型`。也`没有说所有的反射用法都是快的`，只是说反射获取一些信息很快。 MethodTable的数据被分为“热”和“冷”两种数据结构来提升工作效率和缓存利用率，MethodTable本身只存储那些在`程序稳定状态（是否可以翻译成一般运行时？）`下被需求的“热”数据。EEClass储存那些只在`类型加载时`，`JIT编译时`，`反射时`需要的“冷”数据。

## 反射是如何工作的呢

那么到底是哪里让反射花费了额外的时间呢？ 我们来看看一次反射调用他所经历的非托管/托管代码的调用堆栈

*   System.Reflection.RuntimeMethodInfo.Invoke(..)：calling System.Reflection.RuntimeMethodInfo.UnsafeInvokeInternal(..)。[C++MethodInfo源码链接](https://github.com/dotnet/coreclr/blob/b638af3a4dd52fa7b1ea1958164136c72096c25c/src/mscorlib/src/System/Reflection/MethodInfo.cs#L619-L638 "MethodInfo源码链接")
*   System.RuntimeMethodHandle.PerformSecurityCheck(..)：calling System.GC.KeepAlive(..)。[C++reflectioninvocation源码链接](https://github.com/dotnet/coreclr/blob/e67851210d1c03d730a3bc97a87e8a6713bbf772/src/vm/reflectioninvocation.cpp#L949-L974 "reflectioninvocation源码链接")
*   System.Reflection.RuntimeMethodInfo.UnsafeInvokeInternal(..)：calling stub for System.RuntimeMethodHandle.InvokeMethod(..)。[C#MethodInfo源码链接](https://github.com/dotnet/coreclr/blob/b638af3a4dd52fa7b1ea1958164136c72096c25c/src/mscorlib/src/System/Reflection/MethodInfo.cs#L651-L665 "C#MethodInfo源码链接")
*   stub for System.RuntimeMethodHandle.InvokeMethod(..)：大部分工作是在这里完成的，它的源代码超过了400行。[C++reflectioninvocation源码链接](https://github.com/dotnet/coreclr/blob/e67851210d1c03d730a3bc97a87e8a6713bbf772/src/vm/reflectioninvocation.cpp#L949-L974 "reflectioninvocation源码链接")

### 获取方法信息

在你反射调用一个字段/属性/方法之前，你不得不获取FieldInfo/PropertyInfo/MethodInfo来处理，就像这样

```csharp
Type t = typeof(Person);
FieldInfo m = t.GetField("Name");
```

就像前面说的那样，这是有代价的，因为相关的元数据必须被执行获取，解析等操作。非常有趣的是，`运行时通过保留所有字段/属性/方法的内部缓存来帮助我们减少消耗。`这个缓存是由[RuntimeTypeCache类](https://github.com/dotnet/coreclr/blob/b638af3a4dd52fa7b1ea1958164136c72096c25c/src/mscorlib/src/System/RtType.cs#L178-L248 "RuntimeTypeCache类")实现的，其用法的一个例子是[RuntimeMethodInfo类。](https://github.com/dotnet/coreclr/blob/b638af3a4dd52fa7b1ea1958164136c72096c25c/src/mscorlib/src/System/Reflection/MethodInfo.cs#L95 "RuntimeMethodInfo类。") 通过运行上面的的代码，您可以看到缓存的运行情况，这些信息足够使用反射来检查运行时的内部信息! 在您进行任何反射以获得FieldInfo之前，上面的代码将打印以下内容:

```csharp
  Type: ReflectionOverhead.Program
  Reflection Type: System.RuntimeType (BaseType: System.Reflection.TypeInfo)
  m_fieldInfoCache is null, cache has not been initialised yet
```

但是一旦你获取过了哪怕一个字段，下面的内容就会被打印出来:

```csharp
  Type: ReflectionOverhead.Program
  Reflection Type: System.RuntimeType (BaseType: System.Reflection.TypeInfo)
  RuntimeTypeCache: System.RuntimeType+RuntimeTypeCache, 
  m_cacheComplete = True, 4 items in cache
    [0] - Int32 TestField1 - Private
    [1] - System.String TestField2 - Private
    [2] - Int32 <TestProperty1>k__BackingField - Private
    [3] - System.String TestField3 - Private, Static
```

就像这样

```csharp
class Program
{
    private int TestField1;
    private string TestField2;
    private static string TestField3;

    private int TestProperty1 { get; set; }
}
```

这意味着对GetField或GetFields的重复调用会比第一次调用消耗小很多，只需过滤已经创建的预先存在列表就可以了。这同样适用于GetMethod和GetProperty，当您第一次调用MethodInfo或PropertyInfo时，缓存将会被构建出来。

### 参数验证和错误处理

但是，一旦您获得了MethodInfo，当您调用Invoke时，还有很多工作要做。想象你写了一些这样的代码:

```csharp
PropertyInfo stringLengthField = typeof(string).GetProperty("Length", BindingFlags.Instance  BindingFlags.Public);
var length = stringLengthField.GetGetMethod().Invoke(new Uri(), new object[0]);
```

如果你运行它，你会得到以下异常:

```csharp
System.Reflection.TargetException: Object does not match target type.
   at System.Reflection.RuntimeMethodInfo.CheckConsistency(..)
   at System.Reflection.RuntimeMethodInfo.InvokeArgumentsCheck(..)
   at System.Reflection.RuntimeMethodInfo.Invoke(..)
   at System.Reflection.RuntimePropertyInfo.GetValue(..)
```

这是因为我们获得了字符串类的Length属性的PropertyInfo，但是把一个错误的Uri类对象当成它第一个参数（应该是一个字符串类才对），这显然是不对的! 除此之外，还必须对传递给调用的方法的任何参数进行验证。为了使参数传递起作用，反射api使用一个`object[]`的参数，里面保存所需要的参数。因此，如果您使用反射来调用方法`Add(int x, int y)`，您将调用methodInfo.Invoke(..， new\[\]{5,6})。在运行时，需要对传入的值的数量和类型进行检查，在这种情况下，要确保有2个值，而且它们都是int型的。所有这些工作的一个缺点是，它经常涉及`装箱`操作，会有额外的开销。

### 安全性检查

另一个主要任务是多重安全检查。例如，不允许使用反射来调用任何您想调用的方法。有一些受限制的或“危险的方法”，只能被.net框架代码调用。除了黑名单之外，还需要根据调用期间必须检查的当前代码访问安全权限进行动态安全检查。

# 反射到底有多少性能消耗

现在，我们已经知道了反射在幕后做了什么，现在就可以看看它的消耗了。请注意，这些基准测试是通过反射直接比较读取/写入属性。在.net中，属性实际上是编译器为我们生成的一对Get/Set方法，但是，当属性只有一个简单的支持字段时，出于性能原因，.net JIT会将方法调用内联。这意味着使用反射来访问一个属性将会以更糟糕的方式显示反射，但还是选择它因为它是最常见的用例，出现在[ORMs](https://github.com/StackExchange/Dapper "ORMs")、[Json序列化/反序列化库](http://www.newtonsoft.com/json "Json序列化/反序列化库")和[对象映射工具](http://automapper.org/ "对象映射工具")中。 下面是由[BenchmarkDotNet](http://benchmarkdotnet.org/ "BenchmarkDotNet")显示的原始结果，后面是在两个不同的表中显示的相同结果。([完整的测试代码](https://gist.github.com/mattwarren/a8ae31a197f4716a9d65947f4a20a069 "完整的基准代码")) !{}(https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2020/02/Reflection-Benchmark-Results.png) 因此，我们可以清楚地看到，常规反射代码(GetViaReflection和SetViaReflection)比直接访问属性(GetViaProperty和SetViaProperty)慢得多。但是其他的结果呢，让我们更详细地探讨一下。

# 开始优化反射

首先，我们从一个像这样的TestClass开始:

```csharp
public class TestClass
{
    public TestClass(String data)
    {
        Data = data;
    }

    private string data;
    private string Data
    {
        get { return data; }
        set { data = value; }
    }
}
```

和以下筛选设置代码，所有代码都可以获取到:

```csharp
// Setup code, done only once
TestClass testClass = new TestClass("A String");
Type @class = testClass.GetType();
BindingFlag bindingFlags = BindingFlags.Instance  BindingFlags.NonPublic  BindingFlags.Public;
```

## 正常反射

首先，我们使用常规的测试代码，它作为起始点和“最坏情况”:

```csharp
[Benchmark]
public string GetViaReflection()
{
    PropertyInfo property = @class.GetProperty("Data", bindingFlags);
    return (string)property.GetValue(testClass, null);
}
```

下面有五种优化手段

## 优化1：缓存PropertyInfo

接下来，通过保持对PropertyInfo的引用，而不是每次都获取它，我们可以获得一个小的速度提升。但是我们仍然比直接访问属性慢得多，这表明在反射的“调用”部分有相当大的成本。

```csharp
// Setup code, done only once
PropertyInfo cachedPropertyInfo = @class.GetProperty("Data", bindingFlags);

[Benchmark]
public string GetViaReflection()
{
    return (string)cachedPropertyInfo.GetValue(testClass, null);
}
```

## 优化2：使用快速成员

在这里，我们利用Marc Gravell的[优秀的快速成员库](http://blog.marcgravell.com/2012/01/playing-with-your-member.html "优秀的快速成员库")，你可以看到，它们使用起来非常方便!

```csharp
// Setup code, done only once
TypeAccessor accessor = TypeAccessor.Create(@class, allowNonPublicAccessors: true);

[Benchmark]
public string GetViaFastMember()
{
    return (string)accessor[testClass, "Data"];
}
```

注意，它所做的事情与其他做法略有不同。它创建了一个类型访问器（TypeAccessor），允许访问类型上的所有属性，而不仅仅是一个。但有一个不好的地方，它需要更长的运行时间。这是因为在内部，在获取它的值之前，它首先必须获取您请求的属性的委托(在本例中是‘Data’)。然而，这个开销非常小，FastMember仍然比反射快得多，而且它非常容易使用，所以我建议您先看看它。 `此选项和所有后续选项都将反射代码转换为可直接调用的委托，而无需每次都进行反射开销，因此可以提高速度!` 值得指出的是，创建委托是有成本的(更多信息见[“进一步阅读”](https://mattwarren.org/2016/12/14/Why-is-Reflection-slow/#further-reading "“进一步阅读”"))。简而言之，速度的提高是因为我们只做一次昂贵的工作(安全检查等)，并存储一个强类型的委托，我们可以一次又一次地使用它，而开销很小。如果您只做一次反射，就应当使用这些技术， 通过委托读取属性没有直接读取快的原因是.NET JIT不会像访问属性那样内联委托方法调用。对于委托，我们需要支付方法调用的成本，而直接访问不需要。

## 优化3：创建一个委托

在这个选项中，我们使用CreateDelegate函数将PropertyInfo转换为一个常规委托:

```csharp
// Setup code, done only once
PropertyInfo property = @class.GetProperty("Data", bindingFlags);
Func<TestClass, string> getDelegate = (Func<TestClass, string>)Delegate.CreateDelegate(typeof(Func<TestClass, string>), 
                                                                                       property.GetGetMethod(nonPublic: true));

[Benchmark]
public string GetViaDelegate()
{
    return getDelegate(testClass);
}
```

这种方法缺点是你需要在编译时知道具体的类型，即`Func<TestClass，string>`部分在上面的代码(不，你不能使用Func<object，string>，如果你这样做会抛出一个异常!)在大多数情况下，当你在做反射时，你也没想着会这么爽，否则你不会在一开始就使用反射，所以它不是一个完美的解决方案。 要想找到一个非常有趣/头脑风暴的方法来解决这个问题，请参阅Jon Skeet的博客文章[“让反射飞并探索委托”中的MagicMethodHelper代码](https://codeblog.jonskeet.uk/2008/08/09/making-reflection-fly-and-exploring-delegates/ "“让反射飞并探索委托”中的MagicMethodHelper代码")，或者阅读下面的优化4或5。

## 优化4：编译表达式树

这里我们生成了一个委托，但不同的是我们可以直接传递一个`object`，所以我们绕过了‘优化三：创建一个委托’的限制（需要知道明确的对象类型）。我们利用.NET表达式树API，允许动态代码生成:

```csharp
// Setup code, done only once
PropertyInfo property = @class.GetProperty("Data", bindingFlags);
ParameterExpression = Expression.Parameter(typeof(object), "instance");
UnaryExpression instanceCast = 
    !property.DeclaringType.IsValueType ? 
        Expression.TypeAs(instance, property.DeclaringType) : 
        Expression.Convert(instance, property.DeclaringType);
Func<object, object> GetDelegate = 
    Expression.Lambda<Func<object, object>>(
        Expression.TypeAs(
            Expression.Call(instanceCast, property.GetGetMethod(nonPublic: true)),
            typeof(object)), 
        instance)
    .Compile();

[Benchmark]
public string GetViaCompiledExpressionTrees()
{
    return (string)GetDelegate(testClass);
}
```

基于表达式的方法的完整代码可以在[使用表达式树的更快的反射博文](http://geekswithblogs.net/Madman/archive/2008/06/27/faster-reflection-using-expression-trees.aspx "使用表达式树的更快的反射博文")中找到

## 优化5：动态代码生成与IL emit

最后，我们来到了最底层的方法，emit原始IL代码，正所谓“能力越大，责任越大”:

```csharp
// Setup code, done only once
PropertyInfo property = @class.GetProperty("Data", bindingFlags);
Sigil.Emit getterEmiter = Emit<Func<object, string>>
    .NewDynamicMethod("GetTestClassDataProperty")
    .LoadArgument(0)
    .CastClass(@class)
    .Call(property.GetGetMethod(nonPublic: true))
    .Return();
Func<object, string> getter = getterEmiter.CreateDelegate();

[Benchmark]
public string GetViaILEmit()
{
    return getter(testClass);
}
```

使用表达式树(如优化4所示)不会像直接emit IL代码那样给您提供那么多的灵活性，尽管它确实可以防止您发出无效代码!正因为如此，如果你发现自己需要emit IL，我强烈建议你使用优秀的[Sigil库](https://github.com/kevin-montrose/Sigil "Sigil库")，因为当你出错时它会给出更好的错误消息!

# 总结

结论是，如果(且仅当)您发现自己在使用反射时遇到性能问题，有几种不同的方法可以使其更快。这些速度提升都是通过获得一个委托来实现的，该委托允许您直接访问属性/字段/方法，而不需要每次都通过反射进行处理。