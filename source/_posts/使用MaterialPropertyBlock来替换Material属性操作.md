---
title: 测试
---
## **一、官方文档**

Unite 2017 国外技术专场中，Arturo Núñez在《Shader性能与优化专题》中的原话是：

*Use MaterialPropertyBlock Is faster to set properties using a MaterialPropertyBlock rather than material.SetFloat(); Material.SetColor();*

首先，我特意查找了下关于MaterialPropertyBlock的官方文档，文档是这样说的：材质属性块被用于Graphics.DrawMesh和Renderer.SetPropertyBlock两个API，当我们想要绘制许多相同材质但不同属性的对象时可以使用它。例如你想改变每个绘制网格的颜色，但是它却不会改变渲染器的状态。

我们来看看Renderer这个类，它包含了Material，SharedMaterial这两个属性；GetPropertyBlock，SetPropertyBlock这两个函数，其中两个属性是用来访问和改变材质的，而两个函数是用来设置和获取材质属性块的。

**我们知道，当我们操作材质共性时，可以使用SharedMaterial属性，改变这个属性，那么所有使用此材质的物件都将会改变，而我们需要改变单一材质时，需要使用Material属性，而在第一次使用Material时其实是会生成一份材质拷贝的，即Material(Instance)。**

------

## **二、实验**

首先声明两个数组，一个用来保存操作材质，另一个用来保存操作材质属性块。

```cs
GameObject[] listObj = null;
GameObject[] listProp = null;
```

再次声明一个公共变量，用来控制数组长度，以及一个材质属性块。

```cs
public int objCount = 100;
MaterialPropertyBlock prop = null;
```

然后在Start函数中做初始化工作，我们在屏幕左侧空间生成ObjCount个球体Sphere，用来处理材质，在屏幕右侧空间生成ObjCount个球体Sphere，用来处理材质属性块。

```cs
void Start () 
{
        colorID = Shader.PropertyToID("_Color");
        prop = new MaterialPropertyBlock();
        var obj = Resources.Load("Perfabs/Sphere") as GameObject;
        listObj = new GameObject[objCount];
        listProp = new GameObject[objCount];
        for (int i = 0; i < objCount; ++i)
        {
            int x = Random.Range(-6,-2);
            int y = Random.Range(-4, 4);
            int z = Random.Range(-4, 4);
            GameObject o = Instantiate(obj);
            o.name = i.ToString();
            o.transform.localPosition = new Vector3(x,y,z);
            listObj[i] = o;
        }
        for (int i = 0; i < objCount; ++i)
        {
            int x = Random.Range(2, 6);
            int y = Random.Range(-4, 4);
            int z = Random.Range(-4, 4);
            GameObject o = Instantiate(obj);
            o.name = (objCount + i).ToString();
            o.transform.localPosition = new Vector3(x, y, z);
            listProp[i] = o;
        }
}
```

然后我们在Update函数中响应我们的操作，这里我使用按键上下健位来操作。

```cs
void Update () 
{
        if (Input.GetKeyDown(KeyCode.DownArrow))
        {
            Stopwatch sw = new Stopwatch();
            sw.Start();
            for (int i = 0; i < objCount; ++i)
            {
                float r = Random.Range(0, 1f);
                float g = Random.Range(0, 1f);
                float b = Random.Range(0, 1f);
                listObj[i].GetComponent<Renderer>().material.SetColor("_Color", new Color(r, g, b, 1));
            }
            sw.Stop();     
            UnityEngine.Debug.Log(string.Format("material total: {0:F4} ms", (float)sw.ElapsedTicks *1000 / Stopwatch.Frequency));
        }
        if (Input.GetKeyDown(KeyCode.UpArrow))
        {
            Stopwatch sw = new Stopwatch();
            sw.Start();
            for (int i = 0; i < objCount; ++i)
            {
                float r = Random.Range(0, 1f);
                float g = Random.Range(0, 1f);
                float b = Random.Range(0, 1f);
                listProp[i].GetComponent<Renderer>().GetPropertyBlock(prop);
                prop.SetColor(colorID, new Color(r, g, b, 1));
                listProp[i].GetComponent<Renderer>().SetPropertyBlock(prop);             
            }
            sw.Stop();
            UnityEngine.Debug.Log(string.Format("MaterialPropertyBlock total: {0:F4} ms", (float)sw.ElapsedTicks * 1000 / Stopwatch.Frequency));
        }
}
```

这时，我们再来看一下对比数据：
![](http://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/Blog%2FSparkle_Material%2F1.png)
![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/08/QQ截图20190830220919.png)
从结果对比来看，确实使用材质属性块要快于使用材质，其消耗将近是操作材质耗时的四分之一。同时不管是材质还是材质属性块，第一次操作比后面的操作耗时要大。尤其是材质，可见在第一次使用材质改变属性操作时，其拷贝操作消耗还是非常大的。

当然上面的代码还是有优化空间的，因为每次去获取Renderer组件时都是GetComponent的形式来获取的，我们可以在Start时将其保存一下。

```cs
Renderer[] listRender = null;
Renderer[] listRenderProp = null;

...
listRender[i] = o.GetComponent<Renderer>();
...
listRenderProp[i] = o.GetComponent<Renderer>();
...
```

再来看下运行对比数据：
![请输入图片描述](http://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/Blog%2FSparkle_Material%2F2.png)
同时我也通过Profiler的Memory模块，切换进Detailed选项，对其进行采样，可以发现在Sence Memory下面会有Material的拷贝（材质操作导致，而材质属性操作不会）。这也验证了操作材质时会有实例化存在，而使用材质属性块则不存在实例化。

------

## **三、游戏中处理**

正如官方文档介绍材质属性块一样，Unity地形引擎正是使用材质属性块来绘制树的，所有的树使用的是相同材质，但是每棵树有不同的颜色、缩放和风因子。对于大场景大世界来说，我们肯定是动态加载地图的，这个时候我们可以配合GPU Instance来进一步提高性能，使用GPU Instance有两个优点：1）省去实体对象本身的开销；2）减少DrawCall的作用，同时还能减少动态合批的CPU开销和静态合批的内存开销，可谓一举多得。遗憾的是只能在Open GL ES 3.0以上的设备上使用。对于一些游戏中存在自定义皮肤颜色玩法的，材质属性块的优势就可以发挥出来了：当你想让100个不同玩家同屏时，如果使用材质操作颜色属性，那么首先就存在100份材质拷贝的实例；其次，材质操作属性本身就比材质属性块操作要慢那么点，在性能优化中一毫秒的优化就是胜利，那么这里一毫秒那里一毫秒，累积起来就不得了了。

------

## **四、相关工程**

Arturo Núñez 的shader性能与优化的工程下载地址：
https://github.com/ArturoNereu/ShaderProfilingAndOptimization

本次测试工程：
https://pan.baidu.com/s/1qXPGhTa