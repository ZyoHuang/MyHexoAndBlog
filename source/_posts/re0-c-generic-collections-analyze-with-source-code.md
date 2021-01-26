---
title: 从零开始分析C#所有常用集合类的设计（源码向）
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

# 前言
此些天来，武疫兴然，吾甚惧焉，居家不出，项目做完，游戏玩腻，根亦秃噜，百无聊赖，猛然惊醒，吾当面试，基础犹菜，从何下手，唯有C#，C#之何，集合为多，多而不晓，遂有此文。
# 环境
.Net Framework 4.8
# 前置知识
## Hash
Hash算法无论是在项目开发还是底层语言设计上都是非常常见的一个算法，既然如此常用，那么它的重要性自然不必多说，这也是把它放在第一位讲的原因。
### Hash算法与Hash函数
`Hash算法`是一种数字摘要算法，它能将`不定长度`的二进制数据集给映射到一个`较短`的二进制长度数据集。常见的MD5算法就是一种Hash算法，通过MD5算法可对任何数据生成数字摘要。而`实现`了Hash算法的函数我们叫它`Hash函数`。Hash函数有以下几点特征。

1. 相同的数据进行Hash运算，得到的结果一定相同。
2. 不同的数据进行Hash运算，其结果也可能会相同。
3. Hash运算是不可逆的，不能由key获取原始的数据。

#### 常见的Hash算法

1. 直接寻址法：取keyword或keyword的某个线性函数值为散列地址。即H(key)=key或H(key) = a•key + b，当中a和b为常数（这样的散列函数叫做自身函数）
2. 平方取中法：取keyword平方后的中间几位作为散列地址。
3. 折叠法：将keyword切割成位数同样的几部分，最后一部分位数能够不同，然后取这几部分的叠加和（去除进位）作为散列地址。
4. 随机数法：选择一随机函数，取keyword的随机值作为散列地址，通经常使用于keyword长度不同的场合。
5. 除留余数法：取keyword被某个不大于散列表表长m的数p除后所得的余数为散列地址。即 H(key) = key MOD p, p<=m。不仅能够对keyword直接取模，也可在折叠、平方取中等运算之后取模。对p的选择非常重要，一般取素数或m，若p选的不好，容易产生碰撞。

#### Hash桶算法
Hash桶算法主要是用来归类Hash值的，一般用来配合其他算法计算最终Hash，而不是单独用的，不然Hash碰撞会非常频繁。（其实就是除留余数法的最简化版。）

### Hash碰撞
上面的Hash函数特征介绍中有一条：`不同的数据进行Hash运算，其结果也可能会相同。`。这就是Hash碰撞，即`不同的数据经过Hash函数处理后得到了相同的输出`。这并不是我们所期望看到的，所以为了解决Hash碰撞，又产生了很多解决Hash碰撞的方法。
#### 常见解决Hash碰撞的方法

1. 拉链法：这种方法的思路是将产生冲突的元素建立一个单链表，并将头指针地址存储至Hash表对应`桶`的位置。这样定位到Hash表桶的位置后可通过遍历单链表的形式来查找元素。其实这种方法还是有瑕疵的，例如如果字典内所有键都在一个桶的链表里，那么查找起来时间复杂度还是$O(n)$。在Java的HashMap(对位C#的字典)中，当单链表的长度大于8时，会把它变成`红黑树`，优化增删改查效率。`但是在.Net Framework 8里面暂时还没有哦`。
2. 再Hash法：顾名思义就是将key使用其它的Hash函数再次Hash，直到找到不冲突的位置为止。

## C#集合相关的原生类
我们在看C#源码的时候经常能看到一些接口，例如：IDictionary，ICollection，IEnumerable，IEnumerator，IComparer等，其中有些我们在自定义集合类型的时候也能用到，所以总的来说还是比较重要的，所以这里全都列举出来。

- IEnumerable：如果需要自定义的类型支持foreach语义，就需要继承这个接口。有泛型版本（支持协变）。
- IEnumerator：所有泛型计数器（我习惯叫他们迭代器）的基类，提供了一个方法，能够迭代到集合当前的元素。有泛型版本（支持协变）。
- IComparer：用以对比两个对象x,y，如果x>y返回一个大于0的整数，如果x<y返回一个小于0的整数，如果x=y返回一个等于0的整数。会被Sort和BinarySearch调用。
- IEqualityComparer：通常用于对比集合内部两个对象是否相等。返回一个bool变量。有泛型版本（支持逆变）。
- ICollection：继承自IEnumerable。所有集合类型的基础接口。定义迭代器，大小，以及用于同步的方法。有泛型版本。
- KeyValuePairs：键值对类。有泛型版本。其中的TKey和TValue可以为任意类型。
- IList：所有泛型集合的基类。有泛型版本。
- IDictionary：所有泛型字典基础接口。有泛型版本。
# 起飞
## List
### 概述
List是常用的集合类型，他保证了数据的有序性，也就是说放进去的时候是什么顺序，不对List进行更改的话拿出来还是什么顺序。如果容量达到上限，还会自动扩容。
相关字段
```csharp
    private static readonly T[] s_emptyArray = new T[0];
    private T[] _items;
    private int _size;
    private int _version;
```
默认构造函数
```csharp
    public List()
    {
      this._items = List<T>.s_emptyArray;
    }
```
### 增加元素
将给定的对象添加到此列表的末尾。列表的大小增加了一个。在必要的情况下，列表的容量将在添加新元素之前增加一倍。
```csharp
public void Add(T item)
{
	if (_size == _items.Length) EnsureCapacity(_size + 1);
	_items[_size++] = item;
	_version++;
}

private void EnsureCapacity(int min)
{
    if (_items.Length < min)
    {
        int newCapacity = _items.Length == 0 ? _defaultCapacity : _items.Length * 2;
        if ((uint) newCapacity > Array.MaxArrayLength) newCapacity = Array.MaxArrayLength;
        if (newCapacity < min) newCapacity = min;
        Capacity = newCapacity;
    }
}
```
### 删除元素
```csharp
public bool Remove(T item)
{
    int index = IndexOf(item);
    if (index >= 0)
    {
        RemoveAt(index);
        return true;
    }

    return false;
}

public void RemoveAt(int index)
{
    if ((uint) index >= (uint) _size)
    {
        //抛出异常
    }
    _size--;
    if (index < _size)
    {
		//将数组从目标元素后一位开始，全部复制到自身上去
		//这里复制目标数组起始位置刚好是目标元素的位置
		//也就是把目标元素从数组移除
        Array.Copy(_items, index + 1, _items, index, _size - index);
    }

    _items[_size] = default(T);
    _version++;
}
```
### 查找元素
在我们对List调用foreach的时候，会执行List的GetEnumerator方法得到新的迭代器。然后会`进行版本号对比`，一旦在遍历过程中对List做了`增删操作`，版本号就会变化，从而与迭代器的版本号不一致，我们也就会看到`报错异常`。所以不能在遍历过程中对List进行增删操作。会这样设计是有原因的，因为一旦允许在遍历过程中进行增删操作，遍历就不再是遍历了，要么会重复，要么会缺漏，再严重点就直接空指针了
```csharp
public Enumerator GetEnumerator()
{
    //注意每次进行foreach遍历都是返回新的迭代器结构体
    return new Enumerator(this);
}

internal Enumerator(List<T> list)
{
    this.list = list;
    index = 0;
    //注意这里的版本号赋值
    version = list._version;
    current = default(T);
}

private bool MoveNextRare()
{
    if (version != list._version)
    {
        //抛出版本不一致异常
    }

    index = list._size + 1;
    current = default(T);
    return false;
}
```
## Dictionary
### 概述
字典相对于集合就复杂的多了，因为涉及到了Hash碰撞。
需要注意的是，HashTable是线程安全的，而字典不是。
其实字典这一块已经有前辈做了更加详尽的分析：[https://www.cnblogs.com/InCerry/p/10325290.html](https://www.cnblogs.com/InCerry/p/10325290.html "https://www.cnblogs.com/InCerry/p/10325290.html")，我在这里只是再粗略的按自己的理解概括一遍，大家自行斟酌。
相关字段
```csharp
private struct Entry
{
    public int hashCode;    // 低31位hashCode值（忽略符号位）, 如果没有被使用，那么为-1
    public int next;        // 下一个元素的下标索引，如果没有下一个就为-1
    public TKey key;        // 存放元素的键
    public TValue value;    // 存放元素的值
}

private int[] buckets;      // Hash桶，用于依据HashCode归类Entry
private Entry[] entries;
private int count;          // 数组entries元素的数目
private int version;        // 版本号
private int freeList;       // 被删除Entry在entries中的下标index，这个位置是空闲的
private int freeCount;      // 有多少个被删除的Entry，有多少个空闲的位置
private IEqualityComparer<TKey> comparer;   // Key比较器，用于计算两个Key的Hash是否相等
private KeyCollection keys;     // 存放Key的集合
private ValueCollection values;     // 存放Value的集合
```
### 增加元素
当Hash桶内发生碰撞时，会变成单链表。当entries数组满的时候，字典会进行扩容，也就是数组拷贝，最好提前设定好字典容量，避免频繁的字典扩容。
另外这里为了方便讲解，省略了Hash碰撞次数统计。`Hash碰撞次数到达一个阈值，也会触发扩容操作`。
```csharp
private void Insert(TKey key, TValue value, bool add)
{
    if (key == null)
    {
        //抛出异常
    }

    if (buckets == null) Initialize(0);
    //这里进行的与操作（0111 1111 ....）就是为了忽略符号位
    int hashCode = comparer.GetHashCode(key) & 0x7FFFFFFF;
    //计算出对应的Hash桶
    int targetBucket = hashCode % buckets.Length;
    
    //第一步：排查Hash桶内是否有相同的键
    for (int i = buckets[targetBucket]; i >= 0; i = entries[i].next)
    {
        if (entries[i].hashCode == hashCode && comparer.Equals(entries[i].key, key))
        {
            //如果是默认的Add，说明要添加的键值对已存在，抛出异常
            if (add)
            {
                //抛出异常
            }
            //覆盖原本的值，返回
            entries[i].value = value;
            version++;
            return;
        }
    }

    int index;
    if (freeCount > 0)//如果有空闲位置就直接指向此位置
    {
        index = freeList;
        freeList = entries[index].next;
        freeCount--;
    }
    else
    {
        //如果没有空闲位置就扩容数组
        if (count == entries.Length)
        {
			//这个内部对于Hash桶和entries数组都做了扩容操作，即进行了数组拷贝
			//性能消耗比较大，所以最好提前设定好字典容量，避免频繁的字典扩容
            Resize();
            targetBucket = hashCode % buckets.Length;
        }

        index = count;
        count++;
    }
	//如果有空闲位置就以单链表插入数据的方式插入这个entry
    entries[index].hashCode = hashCode;
    entries[index].next = buckets[targetBucket];
    entries[index].key = key;
    entries[index].value = value;
    buckets[targetBucket] = index;
    version++;
}
```
### 删除元素
```csharp
public bool Remove(TKey key)
{
    if (key == null)
    {
        ThrowHelper.ThrowArgumentNullException(ExceptionArgument.key);
    }

    if (buckets != null)
    {
        int hashCode = comparer.GetHashCode(key) & 0x7FFFFFFF;
        int bucket = hashCode % buckets.Length;
		//之所以这里要设置last是因为单链表的删除需要得知前一个节点
        int last = -1;
        for (int i = buckets[bucket]; i >= 0; last = i, i = entries[i].next)
        {
            if (entries[i].hashCode == hashCode && comparer.Equals(entries[i].key, key))
            {
                //如果是桶内最后一个元素，直接指向-1
                if (last < 0)
                {
                    buckets[bucket] = entries[i].next;
                }
                else
                {
                    //如果不是最后一个元素，需要把此元素的前一个节点和后一个节点相连接
                    entries[last].next = entries[i].next;
                }

                //对当前元素位置进行善后操作
                entries[i].hashCode = -1;
                entries[i].next = freeList;
                entries[i].key = default(TKey);
                entries[i].value = default(TValue);
                freeList = i;
                freeCount++;
                version++;
                return true;
            }
        }
    }

    return false;
}
```
### 查找元素
`同样是不能在遍历过程中对字典进行修改`。
```csharp
public bool MoveNext()
{
    if (version != dictionary.version)
    {
        //版本不一致，发出异常
    }
    
    while ((uint) index < (uint) dictionary.count)
    {
        if (dictionary.entries[index].hashCode >= 0)
        {
            current = new KeyValuePair<TKey, TValue>(dictionary.entries[index].key,
                dictionary.entries[index].value);
            index++;
            return true;
        }

        index++;
    }

    index = dictionary.count + 1;
    current = new KeyValuePair<TKey, TValue>();
    return false;
}
```
## SortedDictionary
### 概述
SortedDictionary相对于Dictionary来说它是有序的，并且里面用到了`红黑树`优化查找速率。
`红黑树（RB-Tree）`的查找时间复杂度其实比`自平衡二叉搜索树（AVL）`要高一些，内存占用也和AVL差不多，它的真正优势在于插入删除的效率高。
对于红黑树，大家可以去[https://zhuanlan.zhihu.com/p/95892351](https://zhuanlan.zhihu.com/p/95892351 "https://zhuanlan.zhihu.com/p/95892351")学习，自认为是比较好的教程了。
但是实际上由于排序的存在，它的各项性能其实并不如Dictionary
```csharp
private KeyCollection keys;//键集合，封装了自身的SortedDictionary
private ValueCollection values;//值集合，封装了自身的SortedDictionary
private TreeSet<KeyValuePair<TKey, TValue>> _set;//数据树，继承自SortedSet，实现为红黑树
```
### 增加元素
```csharp
/// <summary>
/// 添加一个元素，并返回一个是否成功的标识
/// </summary>        
internal virtual bool AddIfNotPresent(T item)
{
    if (root == null)
    {
        // 如果是空树，直接添加
        root = new Node(item, false);
        count = 1;
        version++;
        return true;
    }

    //在底部搜索一个节点以插入新节点。
    //如果我们能找到不是4-节点的节点，插入就很容易了。
    //如果没找到，那就需要做一些操作了。
    //我们沿着搜索路径划分了4个节点。
    Node current = root;
    Node parent = null;
    Node grandParent = null;
    Node greatGrandParent = null;

    //即使我们没有真的把它加到这个集合里，我们也可能改变它的结构(通过旋转等)。
    //所以版本号需要递增
    version++;

    int order = 0;
    //这里的遍历主要维护红黑树平衡和颜色
    while (current != null)
    {
        order = comparer.Compare(item, current.Item);
        //说明已有相同项，返回
        if (order == 0)
        {
            //可以在搜索过程中将根节点更改为红色。
            //但是必须在方法结束前把它设回黑色。
            root.IsRed = false;
            return false;
        }

        // 如果是一个4-结点，就把他分成两个2-结点             
        if (Is4Node(current))
        {
            Split4Node(current);
            // 我们可以在分裂后插入两个连续的红结点。并通过旋转来修正红黑树。
            if (IsRed(parent))
            {
                //插入结点，并修正树
                InsertionBalance(current, ref parent, grandParent, greatGrandParent);
            }
        }

        //设置数据准备继续往下遍历
        greatGrandParent = grandParent;
        grandParent = parent;
        parent = current;
        current = (order < 0) ? current.Left : current.Right;
    }

    Debug.Assert(parent != null, "Parent node cannot be null here!");
    // 准备插入这个新结点
    Node node = new Node(item);
    if (order > 0)
    {
        parent.Right = node;
    }
    else
    {
        parent.Left = node;
    }

    // 新节点将是红色的，所以如果父节点也是红色的，我们需要调整颜色与平衡
    if (parent.IsRed)
    {
        InsertionBalance(node, ref parent, grandParent, greatGrandParent);
    }

    // 根节点总是黑色的
    root.IsRed = false;
    ++count;
    return true;
}

/// <summary>
/// 调用InsertionBalance后，我们需要确保当前和父元素是最新的
/// 我们是否及时更新祖父和曾祖父结点并不重要,因为我们不需要在下一个节点中再次拆分。
/// 等到我们需要再次拆分的时候，一切都会正确设置。
/// </summary>
/// <param name="current"></param>
/// <param name="parent"></param>
/// <param name="grandParent"></param>
/// <param name="greatGrandParent"></param>
private void InsertionBalance(Node current, ref Node parent, Node grandParent, Node greatGrandParent)
{
    Debug.Assert(grandParent != null, "Grand parent cannot be null here!");
    bool parentIsOnRight = (grandParent.Right == parent);
    bool currentIsOnRight = (parent.Right == current);

    Node newChildOfGreatGrandParent;
    if (parentIsOnRight == currentIsOnRight)
    {
        //方向相同，旋转方向相同
        newChildOfGreatGrandParent = currentIsOnRight ? RotateLeft(grandParent) : RotateRight(grandParent);
    }
    else
    {
        // 不同方向，两次不同方向的旋转
        newChildOfGreatGrandParent = currentIsOnRight ? RotateLeftRight(grandParent) : RotateRightLeft(grandParent);
        // 当前节点现在成为曾祖父节点的子节点
        parent = greatGrandParent;
    }

    // 祖父结点将成为当前结点的任意父结点的子结点。
    grandParent.IsRed = true;
    newChildOfGreatGrandParent.IsRed = false;

    //替换相关结点
    ReplaceChildOfNodeOrRoot(greatGrandParent, grandParent, newChildOfGreatGrandParent);
}
```
### 删除元素
```csharp
internal virtual bool DoRemove(T item)
{
    if (root == null)
    {
        return false;
    }
    
    version++;

    //搜索一个结点，然后找到它的。
    //然后将项目从成功节点复制到匹配结点，并删除后续结点。
    //如果一个结点没有后续结点，我们可以用它的左子结点来替换它(如果不是空的)。
    //或删除匹配的结点。
    //在自顶向下实现中，确保要删除的节点不是2-结点非常重要。
    //下面的代码将确保路径上的结点不是2-结点。
    Node current = root;
    Node parent = null;
    Node grandParent = null;
    Node match = null;
    Node parentOfMatch = null;
    bool foundMatch = false;
    while (current != null)
    {
        if (Is2Node(current))
        {
            // 修正2-结点
            if (parent == null)
            {
                // 当前结点为根结点，标记他为红色
                current.IsRed = true;
            }
            else
            {
                //获取它的兄弟结点
                Node sibling = GetSibling(current, parent);
                if (sibling.IsRed)
                {
                    //如果父节点是3节点，则翻转红色的方向。
                    //我们可以通过一次旋转得到它
                    Debug.Assert(!parent.IsRed, "parent must be a black node!");
                    if (parent.Right == sibling)
                    {
                        RotateLeft(parent);
                    }
                    else
                    {
                        RotateRight(parent);
                    }

                    parent.IsRed = true;
                    sibling.IsRed = false; 
                    // 兄弟结点旋转后成为祖父母或根的子结点。
                    ReplaceChildOfNodeOrRoot(grandParent, parent, sibling);
                    grandParent = sibling;
                    if (parent == match)
                    {
                        parentOfMatch = sibling;
                    }

                    // 更新兄弟结点
                    sibling = (parent.Left == current) ? parent.Right : parent.Left;
                }

                Debug.Assert(sibling != null || sibling.IsRed == false,
                    "sibling must not be null and it must be black!");

                if (Is2Node(sibling))
                {
                    Merge2Nodes(parent, current, sibling);
                }
                else
                {
                    //current是2-节点，兄弟结点是3-节点或4-节点。
                    //我们可以通过旋转将current的颜色变为红色。
                    TreeRotation rotation = RotationNeeded(parent, current, sibling);
                    Node newGrandParent = null;
                    switch (rotation)
                    {
                        case TreeRotation.RightRotation:
                            Debug.Assert(parent.Left == sibling, "sibling must be left child of parent!");
                            Debug.Assert(sibling.Left.IsRed, "Left child of sibling must be red!");
                            sibling.Left.IsRed = false;
                            newGrandParent = RotateRight(parent);
                            break;
                        case TreeRotation.LeftRotation:
                            Debug.Assert(parent.Right == sibling, "sibling must be left child of parent!");
                            Debug.Assert(sibling.Right.IsRed, "Right child of sibling must be red!");
                            sibling.Right.IsRed = false;
                            newGrandParent = RotateLeft(parent);
                            break;

                        case TreeRotation.RightLeftRotation:
                            Debug.Assert(parent.Right == sibling, "sibling must be left child of parent!");
                            Debug.Assert(sibling.Left.IsRed, "Left child of sibling must be red!");
                            newGrandParent = RotateRightLeft(parent);
                            break;

                        case TreeRotation.LeftRightRotation:
                            Debug.Assert(parent.Left == sibling, "sibling must be left child of parent!");
                            Debug.Assert(sibling.Right.IsRed, "Right child of sibling must be red!");
                            newGrandParent = RotateLeftRight(parent);
                            break;
                    }

                    newGrandParent.IsRed = parent.IsRed;
                    parent.IsRed = false;
                    current.IsRed = true;
                    ReplaceChildOfNodeOrRoot(grandParent, parent, newGrandParent);
                    if (parent == match)
                    {
                        parentOfMatch = newGrandParent;
                    }

                    grandParent = newGrandParent;
                }
            }
        }

        // 一旦我们找到匹配的，就不需要再进行比较了
        int order = foundMatch ? -1 : comparer.Compare(item, current.Item);
        if (order == 0)
        {
            // 保存匹配结点
            foundMatch = true;
            match = current;
            parentOfMatch = parent;
        }

        grandParent = parent;
        parent = current;

        if (order < 0)
        {
            current = current.Left;
        }
        else
        {
            current = current.Right; // 基于二叉搜索树特性，如果要搜索的结点比当前结点大，需要去搜索右子树
        }
    }

    // 如果找到了匹配结点
    if (match != null)
    {
        //用匹配结点的后继结点去替代它，从而达到将匹配结点从树中移除的目的。
        ReplaceNode(match, parentOfMatch, parent, grandParent);
        --count;
    }

    if (root != null)
    {
        root.IsRed = false;
    }

    return foundMatch;
}
```
### 查找元素
它的迭代器的遍历设置的非常之巧妙。你细品。
```csharp
private void Intialize()
{
    current = null;
    SortedSet<T>.Node node = tree.root;
    Node next = null, other = null;
    //这里的初始化只是按照DFS的顺序一直遍历到左下角的叶子结点
    while (node != null)
    {
        //reverse默认为false
        //所以next为左节点
        next = (reverse ? node.Right : node.Left);
        //other为右结点
        other = (reverse ? node.Left : node.Right);
        if (tree.IsWithinRange(node.Item))
        {
            stack.Push(node);
            node = next;
        }
        else if (next == null || !tree.IsWithinRange(next.Item))
        {
            node = other;
        }
        else
        {
            node = next;
        }
    }
}

public bool MoveNext()
{
    //版本检测
    tree.VersionCheck();

    if (version != tree.version)
    {
        ThrowHelper.ThrowInvalidOperationException(ExceptionResource.InvalidOperation_EnumFailedVersion);
    }

    if (stack.Count == 0)
    {
        current = null;
        return false;
    }

    //出栈。current为当前迭代器的值
    current = stack.Pop();
    SortedSet<T>.Node node = (reverse ? current.Left : current.Right);
    Node next = null, other = null;
    //这里算法呼应初始化那里的算法
    //能达到完全遍历红黑树的作用
    while (node != null)
    {
        next = (reverse ? node.Right : node.Left);
        other = (reverse ? node.Left : node.Right);
        if (tree.IsWithinRange(node.Item))
        {
            //入栈，因为要访问所有结点
            stack.Push(node);
            node = next;
        }
        else if (other == null || !tree.IsWithinRange(other.Item))
        {
            node = next;
        }
        else
        {
            node = other;
        }
    }

    return true;
}
```
## SortedList
### 概述
SortedList内部是使用数组实现的，添加和删除元素的时间复杂度是O(n)，查找元素的时间复杂度是O($log_n$)。是典型的用时间换空间。
SortedList和SortedDictionary同时支持快速查询和排序，SortedList优势在于使用的内存比 SortedDictionary 少；但是SortedDictionary可对未排序的数据执行更快的插入和删除操作：它的时间复杂度为 O($log_n$)，而 SortedList为 O(n)。所以SortedList适用于既需要快速查找又需要顺序排列但是添加和删除元素较少的场景。
他两个的对比测试我也做了，只是。。。SortedList在插入1000w数据的时候我等了10来分钟都没好。。。所以也没有测下去的必要了。
相关字段
```csharp
private TKey[] keys;//键数组
private TValue[] values;//值数组
private int _size;//大小
private int version;//版本号
private IComparer<TKey> comparer;//对比器
private KeyList keyList;//键列表，用于外界获取
private ValueList valueList;//值列表，用于外界获取

static TKey[] emptyKeys = new TKey[0];
static TValue[] emptyValues = new TValue[0];

private const int _defaultCapacity = 4;//默认大小
```
### 增加元素
基本上和List一致，只是要处理两个数组
```csharp
private void Insert(int index, TKey key, TValue value)
{
    if (_size == keys.Length) EnsureCapacity(_size + 1);
    if (index < _size)
    {
        Array.Copy(keys, index, keys, index + 1, _size - index);
        Array.Copy(values, index, values, index + 1, _size - index);
    }

    keys[index] = key;
    values[index] = value;
    _size++;
    version++;
}
```
### 删除元素
基本上和List一致，只是要处理两个数组
```csharp
public void RemoveAt(int index)
{
    if (index < 0 || index >= _size)
        ThrowHelper.ThrowArgumentOutOfRangeException(ExceptionArgument.index,
            ExceptionResource.ArgumentOutOfRange_Index);
    _size--;
    if (index < _size)
    {
        Array.Copy(keys, index + 1, keys, index, _size - index);
        Array.Copy(values, index + 1, values, index, _size - index);
    }

    keys[_size] = default(TKey);
    values[_size] = default(TValue);
    version++;
}
```
### 查找元素
```csharp
public bool MoveNext()
{
    if (version != _sortedList.version)
        ThrowHelper.ThrowInvalidOperationException(ExceptionResource.InvalidOperation_EnumFailedVersion);

    if ((uint) index < (uint) _sortedList.Count)
    {
        key = _sortedList.keys[index];
        value = _sortedList.values[index];
        index++;
        return true;
    }

    index = _sortedList.Count + 1;
    key = default(TKey);
    value = default(TValue);
    return false;
}
```
## Stack
### 概述
栈其实也是比较常用的一个数据结构了，它的实现要简单很多。内部实现还是数组，所以Push的时候时间复杂度可能为O(n)，因为可能会发生扩容行为，Pop的话就是O(1)。
相关字段
```csharp
private T[] _array;
private int _size;
private int _version;

private const int _defaultCapacity = 4;
static T[] _emptyArray = new T[0];
```
### 增加元素
```csharp
public void Push(T item)
{
    //如果到达容量上限会进行扩容
    if (_size == _array.Length)
    {
        T[] newArray = new T[(_array.Length == 0) ? _defaultCapacity : 2 * _array.Length];
        Array.Copy(_array, 0, newArray, 0, _size);
        _array = newArray;
    }

    _array[_size++] = item;
    _version++;
}
```
### 删除元素
```csharp
public T Pop()
{
    if (_size == 0)
    {
        //如果当前栈为空就报出异常
    }
    
    //版本控制
    _version++;
    T item = _array[--_size];
    _array[_size] = default(T);
    return item;
}
```
### 查找元素
反倒是它的迭代器这一块设计的倒是挺有意思。
注意同样不能遍历的时候对栈进行增删操作。
先来看他的迭代器构造函数
```csharp
internal Enumerator(Stack<T> stack)
{
    _stack = stack;
    _version = _stack._version;
    _index = -2;
    currentElement = default(T);
}
```
注意到那个index设置为了-2，这里是有特殊含义的，如果index为-2，代表这个迭代器是新生的，或者说是被重置的，迭代开始。如果为-1，说明迭代器已经由于某种原因被终止了，迭代结束。
```csharp
public bool MoveNext()
{
    bool retval;
    if (_version != _stack._version)
        ThrowHelper.ThrowInvalidOperationException(ExceptionResource.InvalidOperation_EnumFailedVersion);
    //如果是第一次的迭代器调用
    if (_index == -2)
    {
        _index = _stack._size - 1;
        retval = (_index >= 0);
        if (retval)
            currentElement = _stack._array[_index];
        return retval;
    }

    //如果栈为空就返回
    if (_index == -1)
    {
        return false;
    }

    retval = (--_index >= 0);
    if (retval)
        currentElement = _stack._array[_index];
    else
        currentElement = default(T);
    return retval;
}
```
## Queue
### 概述
队列内部实现使用了`环形队列算法`，相对于传统的顺序队列，它具有更强的空间可复用性以及安全性。它的入队时间复杂度为O(n)，出队时间复杂度为O(1)。
环形队列算法可以参考：[https://www.jianshu.com/p/f93d0449e410](https://www.jianshu.com/p/f93d0449e410 "https://www.jianshu.com/p/f93d0449e410")
相关字段
```csharp
private T[] _array;
//第一个数据在数组中的位置，即队首
private int _head;
//最后一个数据在数组中的位置，即队尾
private int _tail;
//数组大小
private int _size;
//版本控制
private int _version;

private const int _MinimumGrow = 4;
private const int _ShrinkThreshold = 32;
private const int _GrowFactor = 200;
private const int _DefaultCapacity = 4;
static T[] _emptyArray = new T[0];
```
### 增加元素
```csharp
public void Enqueue(T item)
{
    //需要进行扩容操作
    if (_size == _array.Length)
    {
        int newcapacity = (int) ((long) _array.Length * (long) _GrowFactor / 100);
        if (newcapacity < _array.Length + _MinimumGrow)
        {
            newcapacity = _array.Length + _MinimumGrow;
        }

        SetCapacity(newcapacity);
    }

    _array[_tail] = item;
    //确保_tail一直指向队尾，这里如果直接自增1，因为逻辑上的闭环，可能就会回到前面的位置。
    _tail = (_tail + 1) % _array.Length;
    _size++;
    _version++;
}
```
`_head`一直确保指向队首。
然后这里的扩容就有意思了
```csharp
private void SetCapacity(int capacity)
{
    T[] newarray = new T[capacity];
    if (_size > 0)
    {
        //如果队首的值小于队尾的值，直接复制即可
        if (_head < _tail)
        {
            Array.Copy(_array, _head, newarray, 0, _size);
        }
        else//如果队首的值大于等于队尾的值（这种情况将在先添加后移除再添加的条件下出现）
        {
            //先把从队首指向的数组索引一直到数组最后一位范围内的值复制给新数组（从新数组的0索引开始）
            Array.Copy(_array, _head, newarray, 0, _array.Length - _head);
            //再从数组第一位一直到队尾指向的数组索引范围内的值复制给新数组（从新数组的_array.Length - _head值索引开始）
            //这样才能确保数据顺序正确且完整
            Array.Copy(_array, 0, newarray, _array.Length - _head, _tail);
        }
    }

    _array = newarray;
    _head = 0;
    _tail = (_size == capacity) ? 0 : _size;
    _version++;
}
```
!{环形队列队尾的值小于队首的值情形}(https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2020/01/QQ截图20200128204059.png)
### 删除元素
```csharp
public T Dequeue()
{
    if (_size == 0)
        ThrowHelper.ThrowInvalidOperationException(ExceptionResource.InvalidOperation_EmptyQueue);

    T removed = _array[_head];
    _array[_head] = default(T);
    //同样的，保证_head一直指向队首
    _head = (_head + 1) % _array.Length;
    _size--;
    _version++;
    return removed;
}
```
### 查找元素
队列的查找部分和栈的设计一样，但是具体的数值意义是相反的，如果index为-1，代表这个迭代器是新生的，或者说是被重置的，迭代开始。如果为-2，说明迭代器已经由于某种原因被终止了，迭代结束。
```csharp
public bool MoveNext()
{
    if (_version != _q._version)
        ThrowHelper.ThrowInvalidOperationException(ExceptionResource.InvalidOperation_EnumFailedVersion);

    if (_index == -2)
        return false;

    _index++;

    if (_index == _q._size)
    {
        _index = -2;
        _currentElement = default(T);
        return false;
    }

    _currentElement = _q.GetElement(_index);
    return true;
}
```
## LinkedList
### 概述
LinkedList就是普通的单链表了，内部实现也是直接用的类似C++的方式，并没什么好说的。
## HashSet
### 概述
HashSet就是List的无序版本，查找删除之类的操作性能要比List强，因为内部实现和Dictionary一样用到了Hash桶和单链表。
## 其他
还有很多不是很常用的集合，比如OrderedDictionary，ListDictionary等就不在这一一列举了，如果大家有需要可以自行去[官网](https://referencesource.microsoft.com/#mscorlib "官网")搜索定义。
## 参考文献
[https://www.cnblogs.com/InCerry/p/10325290.html](https://www.cnblogs.com/InCerry/p/10325290.html "https://www.cnblogs.com/InCerry/p/10325290.html")
[https://www.cnblogs.com/songwenjie/p/9185790.html](https://www.cnblogs.com/songwenjie/p/9185790.html "https://www.cnblogs.com/songwenjie/p/9185790.html")
[https://zhuanlan.zhihu.com/p/95892351](https://zhuanlan.zhihu.com/p/95892351 "https://zhuanlan.zhihu.com/p/95892351")
