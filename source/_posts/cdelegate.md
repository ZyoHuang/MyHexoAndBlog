---
title: C#：委托和事件
tags: []
id: '971'
categories:
  - - C#
date: 2019-04-29 22:36:25
---

<meta name="referrer" content="no-referrer" />



### 前言

委托是对函数的封装，可以当做给方法的特征指定一个名称。而事件则是委托的一种特殊形式，当发生有意义的事的时候，事件对象 处理通知过程。\[toc\]

1.  委托是一种引用方法的类型。一旦为委托分配了方法，委托就与该方法具有完全相同的行为。
2.  事件是对委托的封装，被用于表示一个委托对象，
3.  事件是在发生其它类或对象关注的事情的时候，类或对象可通过事件通知他们。
4.  委托用delegate声明，事件用event声明。
5.  事件只能被外部订阅，不能在外部触发，而委托没有这个限制。

### 初始版本

```csharp
using System;

internal class Program
{
    static void Main(string[] args)
    {
        Cat cat = new Cat("Tom");
        Mouse mouse1 = new Mouse("Jerry");
        Mouse mouse2 = new Mouse("Jack");
        /*
         * 表示将Mouse的Run方法通过实例化委托Cat.CatShoutEventHandler登记到Cat的事件CatShout当中，
         * 其中+=表示‘add_CatShout’的意思。即给委托事件添加方法。同理-=给委托事件减少方法
         * 注意，添加的方法参数必须与委托的参数一致。（个数，类型）！！！
         * */
        cat.CatShoutEventHandler += mouse1.Run;
        cat.CatShoutEventHandler += mouse2.Run;
        //当事件触发时，通知所有登记过的对象，并将发送通知的自己以及需要的数据传递过去。
        cat.Shout();
    }
}


class Cat
{
    private string name;

    public Cat(string name)
    {
        this.name = name;
    }

    public delegate void CatShoutDelegate(object sender, CatShoutEventArgs catShoutEventArgs);

    //声明事件CatShout，它的事件类型是委托CatShoutEventHandler
    public event CatShoutDelegate CatShoutEventHandler;

    public void Shout()
    {
        Debug.Log("喵，我是" + name);
        //当执行Shout()方法时，如果CatShout中有对象登记事件，则执行CatShout()
        if (CatShoutEventHandler != null)
        {
            CatShoutEventArgs e = new CatShoutEventArgs();
            e.Name = this.name;
            CatShoutEventHandler(this, e);
        }
    }
}

class Mouse
{
    private string name;

    public Mouse(string name)
    {
        this.name = name;
    }

    public void Run(object sender, CatShoutEventArgs args)
    {
        Debug.Log(args.Name + "来了，" + name + "快跑！");
    }
}


public class CatShoutEventArgs : EventArgs
{
    public string Name { get; set; }
}


public class Debug
{
    public static void Log(string content)
    {
        Console.WriteLine(content);
    }
}
```

### 简化版本

系统内置了EventHandler这个委托类，我们可以直接使用这个泛型委托类来简化我们的代码

```csharp
using System;

internal class Program
{
    static void Main(string[] args)
    {
        Cat cat = new Cat("Tom");
        Mouse mouse1 = new Mouse("Jerry");
        Mouse mouse2 = new Mouse("Jack");
        /*
         * 表示将Mouse的Run方法通过实例化委托Cat.CatShoutEventHandler登记到Cat的事件CatShout当中，
         * 其中+=表示‘add_CatShout’的意思。即给委托事件添加方法。同理-=给委托事件减少方法
         * 注意，添加的方法参数必须与委托的参数一致。（个数，类型）！！！
         * */
        cat.CatShoutEventHandler += mouse1.Run;
        cat.CatShoutEventHandler += mouse2.Run;
        //当事件触发时，通知所有登记过的对象，并将发送通知的自己以及需要的数据传递过去。
        //因为event不能直接在外部触发，所以需要调用函数
        cat.Shout();
    }
}


class Cat
{
    private string name;

    public Cat(string name)
    {
        this.name = name;
    }

    public event EventHandler<CatShoutEventArgs> CatShoutEventHandler;

    public void Shout()
    {
        Debug.Log("喵，我是" + name);
        //当执行Shout()方法时，如果CatShout中有对象登记事件，则执行CatShout()
        if (CatShoutEventHandler != null)
        {
            CatShoutEventArgs e = new CatShoutEventArgs();
            e.Name = this.name;
            CatShoutEventHandler(this, e);
        }
    }
}

class Mouse
{
    private string name;

    public Mouse(string name)
    {
        this.name = name;
    }

    public void Run(object sender, CatShoutEventArgs args)
    {
        Debug.Log(args.Name + "来了，" + name + "快跑！");
    }
}


public class CatShoutEventArgs : EventArgs
{
    public string Name { get; set; }
}


public class Debug
{
    public static void Log(string content)
    {
        Console.WriteLine(content);
    }
}
```