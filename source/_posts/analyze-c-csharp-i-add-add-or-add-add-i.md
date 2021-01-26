---
title: 对于C++/C#中的i++，++i性能问题探究（汇编分析）
tags: []
id: '2562'
categories:
  - - C#
  - - 技术博客
  - - blog
    - 计算机基础知识
date: 2020-02-03 22:14:41
---

<meta name="referrer" content="no-referrer" />



## 前言

这阵子在看面经，里面有一道题，C++中的i++和++i哪个效率更高。说实话，看到这道题当场就蒙了，平时用C#写项目都是怎么高兴怎么来，到C++这里还有这一说了？ 不懂就看答案，答案给出的是++i性能更高，理由是`i++会有一次临时变量的分配消耗，存储初始i值用来返回，而++i则直接返回i+1后的值。` 嗯，看上去很有道理，但是咱也不知道到底实现是不是这样的啊，看汇编去。

## C++汇编分析

### 环境

*   C++环境：MinGW64 w64 3.4
*   CMake：Bundled 3.15.3
*   Debugger：MinGW-w64 GDB 7.8.1
*   IDE：CLion 2019.3.3
*   汇编语法：标准的GAS AT&T语法
*   优化等级 O0（无优化）

### 前提分析

C++只在早期的时候借助C编译器把自己翻译成汇编语言，很久之前就有了自己的编译器，所以直接反编译得到的就是C++的汇编代码。

### 测试用例

源代码

```cpp
int main()
{
    int i = 0;
    i++;
    ++i;
    return 0;
}
```

汇编代码

```
Dump of assembler code for function main():
   0x0000000000401530 <+0>: push   %rbp//入栈，堆栈基指针
   0x0000000000401531 <+1>: mov    %rsp,%rbp//建立被调用者函数的对栈框架
   0x0000000000401534 <+4>: sub    $0x30,%rsp//栈顶指针减去48，也就是向下扩充
   0x0000000000401538 <+8>: callq  0x402100 <__main>
   0x000000000040153d <+13>:    movl   $0x0,-0x4(%rbp)//设置栈底指针往下4位的寄存器值为0（i）

   0x0000000000401544 <+20>:    addl   $0x1,-0x4(%rbp)//i++
=> 0x0000000000401548 <+24>:    addl   $0x1,-0x4(%rbp)//++i

   0x000000000040154c <+28>:    mov    $0x0,%eax//归零通用寄存器
   0x0000000000401551 <+33>:    add    $0x30,%rsp//重置栈顶指针
   0x0000000000401555 <+37>:    pop    %rbp//出栈
   0x0000000000401556 <+38>:    retq   //栈顶的返回地址弹出到IP，然后按照IP此时指示的指令地址继续执行程序
End of assembler dump.
```

我们可以看到，只是两种自增写法都只是单纯的+1操作，没有任何区别。 难道是因为没有赋值对象？ 我又添加了赋值对象，源代码变成这样：

```cpp
int main()
{
    int i = 0, j = 0;
    j = i++;
    j = ++i;
    return 0;
}
```

增加赋值对象后的汇编代码是这样

```
Dump of assembler code for function main():
   0x0000000000401530 <+0>: push   %rbp//入栈，堆栈基指针
   0x0000000000401531 <+1>: mov    %rsp,%rbp//建立被调用者函数的对栈框架
   0x0000000000401534 <+4>: sub    $0x30,%rsp//栈顶指针减去48，也就是向下扩充
   0x0000000000401538 <+8>: callq  0x402110 <__main>

   0x000000000040153d <+13>:    movl   $0x0,-0x4(%rbp)//设置栈底指针往下4位的寄存器值为0（i）
   0x0000000000401544 <+20>:    movl   $0x0,-0x8(%rbp)//设置栈底指针往下8位的寄存器值为0（j）

   //下面的ax后缀都是使用的同一个寄存器，只是高位和低位的区别
   0x000000000040154b <+27>:    mov    -0x4(%rbp),%eax//i++：寄存器eax低32位存储当前i值
   0x000000000040154e <+30>:    lea    0x1(%rax),%edx//i++：寄存器rax高64位+1（也就是i+1），并把值地址赋给edx寄存器
   0x0000000000401551 <+33>:    mov    %edx,-0x4(%rbp)//i++：把i+1赋值给i
   0x0000000000401554 <+36>:    mov    %eax,-0x8(%rbp)//j=i++：把eax的值（原始i值）赋值给j

=> 0x0000000000401557 <+39>:    addl   $0x1,-0x4(%rbp)//i=i+1
   0x000000000040155b <+43>:    mov    -0x4(%rbp),%eax//把i的值赋值给通用寄存器
   0x000000000040155e <+46>:    mov    %eax,-0x8(%rbp)//把通用寄存器的值赋值给j

   0x0000000000401561 <+49>:    mov    $0x0,%eax//归零通用寄存器
   0x0000000000401566 <+54>:    add    $0x30,%rsp//重置栈顶指针
   0x000000000040156a <+58>:    pop    %rbp//出栈
   0x000000000040156b <+59>:    retq   //栈顶的返回地址弹出到IP，然后按照IP此时指示的指令地址继续执行程序
End of assembler dump.
```

对比可以发现，j=i++;与j=++i;差别就是，前者需要一次数据传送，一次地址赋值才能完成相对的一个ADD操作，但是后者只做了一次算术运算——add就达成了。所以硬要说差距的话，就是对比一次数据传送+地址赋值的效率是否高于一次加法运算。 但是我们也看到，第一版的代码里都是使用了add运算，所以是不是可以推断在`需要使用i++/++i值的情况下确实++i的性能比i++要强`呢？

## C#汇编分析

### 环境

*   .Net Framework 4.7.2
*   IDE：Rider 2019.3.3
*   汇编语法：IL

### 前提分析

C#源代码会被编译器编译成IL代码，运行时会被CLR（公共语言运行时）翻译成汇编语言运行。其实IL在语言的位置和汇编也差不多了，但是要比汇编高级一些，IL对汇编语言做了一些封装，就是这些封装让IL代码可读性高了很多。我们这里直接看IL代码就可以了。 也就是说，在jit编译IL到汇编的时候，是有可能再次优化的！ 但是我一时间找不到看C#汇编的方法，所以我们这里直接看IL代码就可以了。(说到底还是条懒狗) 要查看一个编译好的IL代码，一般需要ILDasm.exe来对PE文件进行反编译了。 !{ILDasm界面}(https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2020/02/QQ截图20200203210854.png) 但是Rider中内置了一个IL Viewer工具，可以实时查看IL代码，所以就不需要那么麻烦了（起飞）。

### 测试用例

一样的，先看看不赋值版本的 源代码

```csharp
static void Main(string[] args)
{
    int i = 0;
    i++;
    ++i;
}
```

IL代码

```csharp
.class private auto ansi beforefieldinit
  Program
    extends [mscorlib]System.Object
{

  .method private hidebysig static void
    Main(
      string[] args
    ) cil managed
  {
    .entrypoint
    .maxstack 2
    .locals init (
      [0] int32 i//初始化内存快
    )
//
    // [8 5 - 8 6]
    IL_0000: nop
//
    // [9 9 - 9 19]
    IL_0001: ldc.i4.0//把0推送到计算堆栈上
    IL_0002: stloc.0      // i//从计算堆栈顶部弹出值并存储到0索引处
//
    // [10 9 - 10 13]
    IL_0003: ldloc.0      // i//把0索引处的局部变量加载到计算堆栈上
    IL_0004: ldc.i4.1//把1推送到计算堆栈上
    IL_0005: add//把上面两个值相加并推送到计算堆栈上
    IL_0006: stloc.0      // i//从计算堆栈顶部弹出值并存储到0索引处
//
    // [11 9 - 11 13]
    IL_0007: ldloc.0      // i
    IL_0008: ldc.i4.1
    IL_0009: add
    IL_000a: stloc.0      // i
//
    // [12 5 - 12 6]
    IL_000b: ret
//
  } // end of method Program::Main
//
  .method public hidebysig specialname rtspecialname instance void
    .ctor() cil managed
  {
    .maxstack 8
//
    IL_0000: ldarg.0      // this
    IL_0001: call         instance void [mscorlib]System.Object::.ctor()
    IL_0006: nop
    IL_0007: ret
//
  } // end of method Program::.ctor
} // end of class Program

```

可以看到对于i++/++i生成的IL代码连顺序都没有变。。。 再看看有赋值类型的 C#源代码

```csharp
static void Main(string[] args)
{
    int i = 0, j = 0;
    j = i++;
    j = ++i;
}
```

IL代码

```
.class private auto ansi beforefieldinit
  Program
    extends [mscorlib]System.Object
{

  .method private hidebysig static void
    Main(
      string[] args
    ) cil managed
  {
    .entrypoint
    .maxstack 3
    .locals init (//初始化内存快
      [0] int32 i,
      [1] int32 j
    )
//
    // [8 5 - 8 6]
    IL_0000: nop
//
    // [9 9 - 9 18]
    IL_0001: ldc.i4.0
    IL_0002: stloc.0      // i
//
    // [9 20 - 9 25]
    IL_0003: ldc.i4.0
    IL_0004: stloc.1      // j
//
    // [10 9 - 10 17]
    IL_0005: ldloc.0      // i
    IL_0006: dup//复制计算堆栈上当前最顶端的值，然后将副本推送到计算堆栈上
    IL_0007: ldc.i4.1
    IL_0008: add
    IL_0009: stloc.0      // i
    IL_000a: stloc.1      // j
//
    // [11 9 - 11 17]
    IL_000b: ldloc.0      // i
    IL_000c: ldc.i4.1
    IL_000d: add
    IL_000e: dup
    IL_000f: stloc.0      // i
    IL_0010: stloc.1      // j
//
    // [12 5 - 12 6]
    IL_0011: ret
//
  } // end of method Program::Main
//
  .method public hidebysig specialname rtspecialname instance void
    .ctor() cil managed
  {
    .maxstack 8

    IL_0000: ldarg.0      // this
    IL_0001: call         instance void [mscorlib]System.Object::.ctor()
    IL_0006: nop
    IL_0007: ret

  } // end of method Program::.ctor
} // end of class Program
```

可以看到IL层的C#对于i++/++i这种自增操作一视同仁，基本就可以得出C#i++与++i性能没有差异。