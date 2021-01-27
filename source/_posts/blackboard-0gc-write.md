---
title: 实现行为树黑板模块0GC赋值功能
date:
updated:
tags: [Unity技术, 行为树黑板, Blackboard]
categories:
  - - GamePlay
    - 实用工具
  - - 游戏引擎
    - Unity
keywords:
top_img:
cover:
katex: true
aplayer:
---
<meta name="referrer" content="no-referrer" />

## 引言

因为行为树中的黑板模块可以存储任意类型的数据，并且默认是以System.Object存储的，所以我们在进行赋值的时候，难免会出现装箱的情况，偶尔的一次装箱也没什么，如果是每帧的装箱。。。不会真有人能忍吧，不会吧，不会吧，不会吧



再多说几句，目前我在使用行为树做技能编辑器，之所以可能会出现上面说的*每帧的装箱*的情况，是因为我打算把行为树中所有的关键数据都放到黑板中，这样有很多好处

- 方便状态预测/回滚
- 方便Debug，逐帧调试
- 方便满足策划的所见即所得需求

上面说的这些我会在实现之后再和大家分享，有兴趣的可以先关注我的开源Moba项目

[![烟雨迷离半世殇/NKGMobaBasedOnET](https://gitee.com/NKG_admin/NKGMobaBasedOnET/widgets/widget_card.svg?colors=eae9d7,2e2f29,272822,484a45,eae9d7,747571)](https://gitee.com/NKG_admin/NKGMobaBasedOnET)

## 正文

要实现黑板赋值的0GC，就是要把他的装箱那一步给去掉，所以理所当然的，我们要自己封装数据类型

### 基类

```cs
public abstract class ANP_BBValue
{
    public abstract Type NP_BBValueType { get; }
}
```

```cs
public interface INP_BBValue<T>
{
    T GetValue();
    void SetValue(INP_BBValue<T> bbValue);
}
```

```cs
public abstract class NP_BBValueBase<T>: ANP_BBValue, INP_BBValue<T>
{
    public T Value;
    
    public T GetValue()
    {
        return Value;
    }
    
    public void SetValue(INP_BBValue<T> bbValue)
    {
        Value = bbValue.GetValue();
    }
    
    public void SetValue(T bbValue)
    {
        Value = bbValue;
    }
}
```

### 具体数据类

实现具体数据类就需要重写运算符，[参考C#官方文档](https://docs.microsoft.com/zh-cn/dotnet/csharp/programming-guide/statements-expressions-operators/how-to-define-value-equality-for-a-type)

下面是bool数据类型示例

```cs
public class NP_BBValue_Bool: NP_BBValueBase<bool>, IEquatable<NP_BBValue_Bool>
{
    public override Type NP_BBValueType
    {
        get
        {
            return typeof (bool);
        }
    }
    #region 对比函数
    public bool Equals(NP_BBValue_Bool other)
    {
        // If parameter is null, return false.
        if (Object.ReferenceEquals(other, null))
        {
            return false;
        }
        // Optimization for a common success case.
        if (Object.ReferenceEquals(this, other))
        {
            return true;
        }
        // If run-time types are not exactly the same, return false.
        if (this.GetType() != other.GetType())
        {
            return false;
        }
        // Return true if the fields match.
        // Note that the base class is not invoked because it is
        // System.Object, which defines Equals as reference equality.
        return this.Value == other.GetValue();
    }
    public override bool Equals(object obj)
    {
        if (ReferenceEquals(null, obj))
        {
            return false;
        }
        if (ReferenceEquals(this, obj))
        {
            return true;
        }
        if (obj.GetType() != this.GetType())
        {
            return false;
        }
        return Equals((NP_BBValue_Bool) obj);
    }
    public override int GetHashCode()
    {
        return this.Value.GetHashCode();
    }
    public static bool operator ==(NP_BBValue_Bool lhs, NP_BBValue_Bool rhs)
    {
        // Check for null on left side.
        if (Object.ReferenceEquals(lhs, null))
        {
            if (Object.ReferenceEquals(rhs, null))
            {
                // null == null = true.
                return true;
            }
            // Only the left side is null.
            return false;
        }
        // Equals handles case of null on right side.
        return lhs.Equals(rhs);
    }
    public static bool operator !=(NP_BBValue_Bool lhs, NP_BBValue_Bool rhs)
    {
        return !(lhs == rhs);
    }
    public static bool operator >(NP_BBValue_Bool lhs, NP_BBValue_Bool rhs)
    {
        return false;
    }
    public static bool operator <(NP_BBValue_Bool lhs, NP_BBValue_Bool rhs)
    {
        return false;
    }
    public static bool operator >=(NP_BBValue_Bool lhs, NP_BBValue_Bool rhs)
    {
        return false;
    }
    public static bool operator <=(NP_BBValue_Bool lhs, NP_BBValue_Bool rhs)
    {
        return false;
    }
    #endregion
}
```

在黑板中的存储方式就是

```cs
private Dictionary<string, ANP_BBValue> m_Data = new Dictionary<string, ANP_BBValue>();
```

然后我们在创建一个行为树的时候直接初始化黑板中所有键值对

```cs
//配置黑板数据，这段代码是个示例，前面还有一部分代码是用来从二进制文件反序列化出来数据的
//如果看不懂可以无视，总之就是创建行为树的时候要初始化黑板数据
Dictionary<string, ANP_BBValue> bbvaluesManager = tempTrcsee.GetBlackboard().GetDatas();
foreach (var bbValues in npDataSupportor.NP_BBValueManager)
{
    bbvaluesManager.Add(bbValues.Key, bbValues.Value);
}
```

最后我们就可以不用担心拆装箱问题来访问/写入数据了

```cs
public T Get<T>(string key)
{
    ANP_BBValue result = Get(key);
    if (result == null)
    {
        return default;
    }
    return (result as NP_BBValueBase<T>).GetValue();
}
```

```cs
/// <summary>
/// 设置黑板值，
/// 注意此处的T需要是被已注册的黑板数据类型包裹的，例如这里的T为int，那么必须存在一个例如NP_BBValue_Int
/// </summary>
/// <param name="key"></param>
/// <param name="value"></param>
/// <typeparam name="T"></typeparam>
public void Set<T>(string key, T value)
{
    if (this.m_ParentBlackboard != null && this.m_ParentBlackboard.Isset(key))
    {
        this.m_ParentBlackboard.Set(key, value);
    }
    else
    {
        if (!this.m_Data.ContainsKey(key))
        {
            Log.Error($"黑板中未注册key为{key}的键值对");
        }
        else
        {
            NP_BBValueBase<T> targetBBValue = this.m_Data[key] as NP_BBValueBase<T>;
            if ((targetBBValue == null && value != null) ||
                (targetBBValue != null && (targetBBValue.GetValue() == null || !targetBBValue.GetValue().Equals(value))))
            {
                targetBBValue.SetValue(value);
				......
            }
        }
    }
}
```

