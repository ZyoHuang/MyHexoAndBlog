---
title: 浅谈C++与C#泛型编程的区别与联系
date:
updated:
tags:
categories:
keywords:
top_img:
cover:
aplayer:
---
<meta name="referrer" content="no-referrer" />

## 前言

这阵子群友讨论了很多次C++模板元编程和C#泛型的区别，但是大家懂得，在QQ群讨论这种体量的论题，你一言我一语很难说清楚，尤其是大家对各种名词的理解和定义不一致的情况下，所以我决定今天整一篇文章，结合代码，浅谈一下C++和C#泛型编程的区别与联系。

**个人水平有限，欢迎大家指正，补充。**

## 名词定义

正如上面所说，每个人对于各种名词的认识之间可能存在着差别，所以在这里统一用我的方式定义一下以下名词

- 泛型：泛型是一种程序语言设计技术，具体体现为，我们事先写好一份泛型程序(这里叫模板更合适)，其中有一些部分或者全部是运行时根据我们的写法确定的，然后生成这部分的新实例，最后执行的是这些运行时生成的新实例
- C#泛型：C#2.0新增的泛型技术
- C++泛型/模板/模板元：即与template关键字相关的泛型技术

## 正文

由于C++的泛型功能可以算是C#泛型的超集，所以下面就以C++各项泛型功能为基准对比这两个语言的泛型功能

### 泛型函数

C++

```cpp
class Generic_A
{
public:
    //声明与定义
    template<typename T>
    void Run(){}
};

Generic_A *genericA = new Generic_A();
//调用
genericA->Run<int>();
delete genericA;
```

C#

```cs
public class Generic_A
{
    //声明与定义
    public void Run<T>()
    {
    }
}

Generic_A genericA = new Generic_A();
//调用
genericA.Run<int>();
```

### 泛型类

C++

```cs
template<typename T>
class Generic_A
{
};

Generic_A<int> *genericA = new Generic_A<int>();
delete genericA;
```

C#

```cs
public class Generic_A<T>
{
}

Generic_A<int> genericA = new Generic_A<int>();
```

### 泛型参数

#### 参数类型

C++：模板参数除了类型外（包括基本类型、结构、类类型等），**也可以是一个整型数（Integral Number）**。这里的整型数比较宽泛，包括布尔型，不同位数、有无符号的整型，甚至包括指针。

C#：只能是类型

#### 可变参数

C++

```cpp
template<typename ...Args>
void Run(const Args& ...args){};

Generic_A *genericA = new Generic_A();
genericA->Run<int>(1, 2, 3);
delete genericA;
```

C#

```cs
public class Generic_A
{
    public void Run<T>(params T[] args)
    {
    }
}

Generic_A genericA = new Generic_A();
genericA.Run(1, 2, 3);
```

#### 默认参数

C++

```cpp
class Generic_A
{
public:
    template<typename T = int>
    void Run(){};
};

Generic_A *genericA = new Generic_A();
genericA->Run<>();
delete genericA;
```

C#

**不支持**

### 控制实例化

C++
C++中的泛型被使用时会被实例化，这意味着，相同类的实例可能出现在多个文件中，针对这一问题，可以用**显式实例化（extern）**来避免不必要的开销

C#

CLR内核优化，例如，特定类型实参调用了一个方法，以后再用这个类型实参调用这个方法，CLR只会为这个方法/类型组合编译一次代码，

此外，CLR认为所有引用类型实参都一样，所以可以代码共享，例如，为`List<String>`方法编译的代码可以直接用于`List<Stream>`方法，这是因为所有引用类型实参/变量只是指向托管堆的一个8字节指针（这里假设64位系统），但是对于值类型，则必须每种类型都进行代码生成，因为值类型大小不定

### 特化/偏特化

C++

```cs
// 首先，要写出模板的一般形式（原型）
template <typename T> class AddFloatOrMulInt
{
    static T Do(T a, T b)
    {
        // 在这个例子里面一般形式里面是什么内容不重要，因为用不上
        // 这里就随便给个0吧。
        return T(0);
    }
};

// 其次，我们要指定T是int时候的代码，这就是特化：
template <> class AddFloatOrMulInt<int>
{
public:
    static int Do(int a, int b) // 
    {
        return a * b;
    }
};

// 再次，我们要指定T是float时候的代码：
template <> class AddFloatOrMulInt<float>
{
public:
    static float Do(float a, float b)
    {
        return a + b;
    }
};
```

C#

**不支持**

### 语法检测

C++

参照**双阶段名称查找**，部分报错无法在编辑时，甚至编译期给出，只有在运行时才报出

C#

强制转型方面，在没有泛型约束的时候，参数被当做object

其余语法错误可正常给出

## 参考资料

《CLR Via C#》

《C++ Primer》

《[CppTemplateTutorial](https://github.com/wuye9036/CppTemplateTutorial)》
