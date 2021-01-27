---
title: 记一次检测到Mac文件格式请将源文件转换为DOS格式或UNIX格式解决方案
tags: Unity技术
categories:
  - - GamePlay
    - 实用工具
  - - 游戏引擎
    - Unity
date: 2020-06-26 14:39:39
---

<meta name="referrer" content="no-referrer" />



## 前言

事情起因是在用Lua根据文件模板生成CPP过程中出现的问题。 文件是正常生成了，但是编译遇到了错误，报错就是`检测到Mac文件格式：请将源文件转换为DOS格式或UNIX格式`

## 查找问题

几番百度谷歌，大家一致认为是每行末尾的`CR LF`出了问题

> **在Windows下换行使用CRLF两个字符来表示，其中CR为回车（ASCII=0x0D），LF为换行（ASCII=0x0A）**

但是无论是记事本和IDE都无法显示这两个字符 网上说Notepad++可以显示
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2020/06/QQ截图20200626143005.png) 我试了下，可行！ 
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2020/06/QQ截图20200626143054.png) 到这里问题就很明显了，就是有部分文本行结尾是CR，而不是Windows标准的CRLF **但是问题出在哪呢？** 文件模板没问题 
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2020/06/QQ截图20200626143226.png) 
我在Lua那边的字符串拼接（`..`）和替换（`string.gsub`）过程打印了一下，也都是正常的字符串 那问题只可能出在最后的往文件内写入字符串（`file:write`）了 想了想原因，`应该是我直接整个读入的文件模板，而不是逐行读入，造成了写入文件过程中他会自动用`CR`分割每行内容，所以就出现了这个问题`

## 解决问题

找到问题根源就很好解决了，我们只需要在最后写入文件前，逐行替换`'\r'`为`'\n'`即可 因为我文件模板是用'\\n'分割每行，所以先把内容存到一个lua 数组里

```lua
    --LF转CRLF避免报错
    local splitlist = {}
    string.gsub(fileContent, '[^\n]+', function(w)
        table.insert(splitlist, w)
    end)

    for i = 1, #splitlist do
        splitlist[i] = string.gsub(splitlist[i], '\r', '\n')
        file_h:write(splitlist[i])
    end
```

拿下