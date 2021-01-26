---
title: Google.Protobuf篇：使用protoc.exe生成自己的类
tags: []
id: '823'
categories:
  - - C#
  - - c
    - Google.Protobuf
date: 2019-04-29 17:12:58
---

<meta name="referrer" content="no-referrer" />



### 正常生成

上次我们提到了一个比较特殊的类,Addressbook,然后他开头第二行有这末个东西 ![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190208203444131.png) 要编辑这个proto后缀的文件,需要用protoc来编辑 https://github.com/protocolbuffers/protobuf/releases 不要问我为什么不下载64位的,问就是64位的我不会搞 ![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190208205746177.png) 为.exe文件设置环境变量 这样做的好处是,在任何地方都可以运行.exe文件 此电脑-属性-高级系统设置-高级-环境变量-编辑用户变变量的Path变量-添加protoc.exe的所在目录. 比如我的目录是:D:\\Protoc\\protoc-3.7.0-rc-2-win32\\bin ![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190208205717618.png) 迁移include文件夹下的文件 如果没有include文件夹下的文件,在使用protoc.exe时会提示缺少google\\protobuf\\timestamp.proto 解决办法是将include/google文件夹移动到指定了环境变量的文件夹 ![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190208210027974.png) 然后就可以在cmd中输入protoc,他应该会像配置Java环境那样有一大长串 ![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190208205903700.png) 准备工作做好了,我们来找找这个万恶之源,address.protoc ![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190208211436129.png) 然后把它拖到Rider查看(报错可以不用管,因为我们是强行直接读取的文件,它所依赖的包或者文件并没有被我们当前的工程所涵盖,所以会报错) ![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190208214753316.png) 发现比较难懂 package 对应于c#中的命名空间  required 对应类的属性 (该变量必填) optional 创建一个具有默认值的属性，通过\[default=XXX\]设置默认值，不添加默认为空置。如string默认为“”，int默认为0  enum 创建枚举  message 创建自定义类或内部类  repeated 对应list列表数据  有同学发现了,明明package才代表命名空间,但我们的AdressBook类的命名空间却是 ![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190208214013946.png) 为什么呢?正好,我们做个实验,顺便学习一下使用protoc.exe生成cs文件 我们把这一行注释 ![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190208214110718.png) 由于我的addressbook.proto在D:\\protobuf-csharp-3.7.0-rc-2\\protobuf-3.7.0-rc-2\\examples 所以我的命令行输入如下(注意,第三句的/与a之间有空格,不然会出错)

```bash
d:
cd D:\protobuf-csharp-3.7.0-rc-2\protobuf-3.7.0-rc-2\examples
protoc --csharp_out=./ addressbook.proto
```

然后我们就发现生成了AddressBook.cs文件 至于它的命名空间,是我们的package名,tutorial(他底层把T大写了) 所以我们得出结论,是下面那行 `option csharp_namespace = "Google.Protobuf.Examples.AddressBook"` 把package所重写了 ![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190208215144716.png)

### 使用批处理命令生成代码

但是我们如果一直手动配置路径生成代码效率实在太低，于是我想到了使用批处理命令来提升效率 **打开proto文件所在文件夹（如果有特殊需求，则需要执行cd命令来打开对应的文件），在其中创建.bat文件** **在其中填写**

```bash
 @echo off
for %%i in (*.proto) do (
    protoc --csharp_out=./ %%i
    rem 从这里往下都是注释，可忽略
    echo From %%i To %%~ni.cs Successfully!  
)
pause
```

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/QQ截图20190429171213.png)