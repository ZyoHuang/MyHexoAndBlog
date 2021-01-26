---
title: ET篇：那些千万不能踩的坑
date:
updated:
tags:
categories:
keywords:
top_img:
cover:
katex: true
aplayer:
---
<meta name="referrer" content="no-referrer" />

**本篇博客记录ET中所有的坑，希望对大家有帮助
前方高能预警**

### 1.`不要在热更层的async语句中使用try Catch finally语法，因为finally语句不会执行（在Try或Catch中执行Return）！`**

### 2.报错信息：`BsonSerializationException: When using DictionaryRepresentation.Document key values must serialize as strings.`

​    原因：`Bson默认序列化需要string的key`
​    解决方案：[https://stackoverflow.com/questions/28111846/bsonserializationexception-when-serializing-a-dictionarydatetime-t-to-bson](https://stackoverflow.com/questions/28111846/bsonserializationexception-when-serializing-a-dictionarydatetime-t-to-bson "https://stackoverflow.com/questions/28111846/bsonserializationexception-when-serializing-a-dictionarydatetime-t-to-bson")**

### 3.报错信息：`Element 'Id' does not match any field or property of class`

​    原因：`使用Bson反序列化之前没有对牵扯到的继承关系进行注册`
​    解决方案：`对其进行注册`
​    举例：

```csharp
public class Component
{
}
public class Entity: Component
{
}
public sealed class Player: Entity
{
	public long Id;
 
	public string Account { get; set; }
 
	public long UnitId { get; set; }
}
```
**`在反序列化之前执行以下代码即可`**
```csharp
//这里的Game是一个包含在程序集的类，利用它来牵引出当前程序集，用以遍历所有类
Type[] types = typeof(Game).Assembly.GetTypes();
foreach (Type type in types)
{
	//这里的Component是父类
	if (!type.IsSubclassOf(typeof(Component)))
	{
		continue;
	}
	BsonClassMap.LookupClassMap(type);
}
```

### 4.Bson不支持反序列化struct！！！

### 5.Bson不支持反序列化委托！！！

### 6.在ILRT中使用UNITY_EDITOR将在真机测试时失效！！！

### 7.对Component的释放逻辑，尽量写在DestroySystem中

有两个原因

- 第一，写在Dispose中，不利于热更新逻辑拆分

- 第二，我们可能会在Dispose中获取Entity进行操作，但此时的Entity因为这些代码，已经不是我们想要的Entity了

```c#
        public void Recycle(Component obj)
        {
	        obj.Parent = this;
            Type type = obj.GetType();
	        ComponentQueue queue;
            if (!this.dictionary.TryGetValue(type, out queue))
            {
                queue = new ComponentQueue(type.Name);
	            queue.Parent = this;
#if !SERVER
	            queue.GameObject.name = $"{type.Name}s";
#endif
				this.dictionary.Add(type, queue);
            }
            queue.Enqueue(obj);
        }
```

