---
title: 探索C#中值类型/引用类型为空时所占用的内存大小
tags: []
id: '2597'
categories:
  - - C#
  - - 技术博客
  - - blog
    - 计算机基础知识
date: 2020-02-09 00:07:12
---

<meta name="referrer" content="no-referrer" />



## 前言

又看到一个比较让人害怕的面试题，实例化一个C#的空class会占用多少内存空间。 。。。为什么会有这种问题，当时第一反应肯定不是零，至少要像C++那样有一个区别地址的偏移吧，不然找都找不到，所以就猜了1。 结果果然不出我所料，`我蒙错了`，不会就学。 找了一大圈，终于找到一个讲的全面的帖子：[https://docs.microsoft.com/en-us/archive/msdn-magazine/2005/may/net-framework-internals-how-the-clr-creates-runtime-objects](https://docs.microsoft.com/en-us/archive/msdn-magazine/2005/may/net-framework-internals-how-the-clr-creates-runtime-objects "https://docs.microsoft.com/en-us/archive/msdn-magazine/2005/may/net-framework-internals-how-the-clr-creates-runtime-objects") 下面的内容基本就是围绕这篇文章来说的，也可以直接去看原文。\[toc\]

## 正文

环境：.Net Framework 4.7.2 IDE：Rider 2019.3.3 编译环境：Debug prefer x86（32bit）

### 值类型

值类型相较于引用类型好理解多了，他直接分配到栈上，不需要GC，不需要引用机制。所以它空间的计算也是最简单的。直接计算其中包含的字段空间即可（有一些特性会影响内存的编排，不过不在本篇文章讨论范围内了）。 计算机中的内存通常以字节的形式组织，在对象地址位置可用的最小内存为1字节。所以空值类型的对象大小是1字节

### 引用类型

先来看代码

```csharp
using System;

class SmallClass
{
    private byte[] _largeObj;

    public SmallClass(int size)
    {
        _largeObj = new byte[size];
        _largeObj[0] = 0xAA;
        _largeObj[1] = 0xBB;
        _largeObj[2] = 0xCC;
    }

    public byte[] LargeObj
    {
        get { return this._largeObj; }
    }
}

class SimpleProgram
{
    static void Main(string[] args)
    {
        SmallClass smallObj = SimpleProgram.Create(84930, 10, 15, 20, 25);
        return;
    }

    static SmallClass Create(int size1, int size2, int size3, int size4, int size5)
    {
        int objSize = size1 + size2 + size3 + size4 + size5;
        SmallClass smallObj = new SmallClass(objSize);
        return smallObj;
    }
}
```

它的运行时堆栈是这样的 !{}(https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2020/02/QQ截图20200208224828.png) 引用类型变量（如smallObj）以固定大小（4字节）存储在堆栈上，并指向在托管堆上分配的对象实例的地址。 smallObj的实例包含指向相应类型的MethodTable的TypeHandle（类型对象指针）和syncblk index（同步块索引，用来做线程同步的，这里就不详细讲了，大家可以去原文查看）。 !{}(https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2020/02/QQ截图20200208225348.png) 每种声明的类型都有一个`MethodTable`，并且`同一类型的所有对象实例`都指向`同一MethodTable`。其中包含大量信息，包含有关类型的信息（接口，抽象类，具体类，COM Wrapper和代理），实现的接口数，用于方法调用的接口映射，方法表中的插槽数以及表的信息，指向实现的插槽数量（包括虚函数的实现也在这个MethodTable中）。 当前的GC实现对于一个空类来说，需要至少12个字节的对象实例。如果一个类没有定义任何实例字段，它将产生4个字节的开销（用于分配到栈上来对他进行引用）。其余部分(8个字节)的将由同步块索引和类型对象指针占用。 所以对于一个空引用类型来说，他所占用的空间大小就是`12`。

## 踩过的坑（针对引用类型）

*   使用sizeof操作得到的只是指向对象的指针，也就是说全是4。
*   Marshal.SizeOf得到的大小也不是正确的对象大小，具体原因未知。

## 推荐一个可以查看reference/value type的库

[https://github.com/sidristij/dotnetex](https://github.com/sidristij/dotnetex)

## 饭后甜点1，C++空类型的空间占用

我们都知道，C++当中并没有对位C#的值类型和引用类型概念，只有一般数据类型，指针，引用，这几个概念。所以概括起来也比较容易一些。 即所有空数据类型所占用的内存大小都是`1`字节！ 这是因为，对象需要有不同的地址。有了不同的地址，就可以比较指针和对象的身份。 如果不为空的话，编译器会对其进行优化，会是正常的字段大小相加+内存对齐的结果。也就是说，如果class A{int i;};。它的大小就是4字节。

## 饭后甜点2，C++非空类型的空间占用

非空类型就比较麻烦了，涉及内存对齐的问题。 下面是一个笔试题

```cpp
#include <iostream>
#pragma pack(2)
struct S1
{
    S1() { f = 0; s = 0; i = 0; c = 0; }
    float f;
    short s;
    int i;
    char c;
};

#pragma pack(push)
#pragma pack(16)
struct S2
{
    S2() { d = 0; c = 0; i = 0; }
    double d;
    S1 s1;
    char c;
    int i;
};
#pragma pack(pop)

int main()
{
    std::cout << sizeof(S2) << std::endl;
}
```

这是我当时做的总结：

*   S1
*   float占4个字节
*   short占2个字节
*   int占4个字节
*   char占1个字节
    
*   S2
    
*   double占8个字节
*   S1展开另算
*   char占1个
*   int占4个

结构体在内存中的存放按单元进行存放，每个单元的大小取决于结构体中最大基本类型的大小，为了优化性能会自动进行内存对齐，整个结构体的大小必须是最大成员变量类型所占字节数的整数倍 #pragma pack(16)定义补齐方式为16字节（在这里其实和32位默认的对齐方式也没差，用S1的double作为最宽长度即可） 所以，S2就是 先算S2自带的：double(8)+char(1->4)+int(4)=16 再单独算S1：float(4)+short(2->4)+int(4)+char(1->4)=16 所以S2所占总空间为32 对于C++类，结构体分配内存拓展阅读： [https://blog.csdn.net/chen1234520nnn/article/details/83341266](https://blog.csdn.net/chen1234520nnn/article/details/83341266 "https://blog.csdn.net/chen1234520nnn/article/details/83341266") [https://www.cnblogs.com/-zhangnian/p/6422559.html](https://www.cnblogs.com/-zhangnian/p/6422559.html) [https://www.cnblogs.com/linuxAndMcu/p/10389096.html](https://www.cnblogs.com/linuxAndMcu/p/10389096.html) [https://blog.csdn.net/GAMEloft9/article/details/47440941](https://blog.csdn.net/GAMEloft9/article/details/47440941)