---
title: xasset入门指南
date:
updated:
tags: [Unity资源管理]
categories:
  - - GamePlay
    - 实用工具
  - - 游戏引擎
    - Unity
  - 热更新技术
keywords:
top_img:
cover: https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200711145551782.png
aplayer:
---
<meta name="referrer" content="no-referrer" />

## 什么是xasset

众所周知，Unity资产管理方面的知识十分细碎，很多细节稍不注意就会导致资源冗余或者内存泄漏，很多前辈也在为解决这个问题不懈的努力。

今天为大家介绍的是之前有直播过的一个开源的Unity项目资源管里利器，因为它发布了新的4.0版本，支持了很多新的特性所以需要重新给大家再介绍下。

我本人的风格一向是从运行Demo开始，逐步分析理解它的架构，所以这个指南也不会一开始就从宏观上带大家去理解（其实是功课没做足，确实不知道是什么个情况 XD），不过有一说一，个人觉得以这种行文方式非常适合做入门指南

## 下载

https://github.com/xasset/xasset

git大家肯定都会用，如果速度慢，可以从我的xasset码云镜像拉取：https://gitee.com/NKG_admin/xasset_Gitsync.git

## 环境

游戏引擎： Unity 2019.4.0 LTF

.Net框架：.Net Framework 4.7.2

IDE：Rider 2019.3

xasset版本：截至此Commmit https://github.com/xasset/xasset/commit/3d4983cd24ff92a63156c8078caf34b20d2d4c02

## 运行

来到Init场景，直接点击运行（我们可以看到UI界面相当有内味，作者下了血本在Asset Store购买的，泪目）

![image-20200711145215706](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200711145215706.png)

这个VFS，全名Virtual File System，用于提高IO性能(android)和安全性，建议开启，后面会细谈。

### 资源热更新

作者已经配置好了远程资源服务器路径

![image-20200711145408391](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200711145408391.png)

如果有内容更新，就会出现这个界面（不过速度很慢就是了，因为现在我们还整不明白怎么打出AB包，所以就先用Demo的这个云端文件服务器，后面推荐给大家一个本地的虚拟文件服务器，用于学习和研究框架）

![image-20200711145422755](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200711145422755.png)

![image-20200711145551782](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200711145551782.png)

下载完成后出现此界面

![image-20200711150537867](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200711150537867.png)

我们先不要着急往下探索，看看控制台输出

![image-20200711150619497](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200711150619497.png)

第一个Log主要来自Versions.cs和Updater.cs

第二个和第三个Log主要来自Download.cs和Downloader.cs，我们一个一个看



#### Versions.cs

通过翻看其相关联的部分源码，可以看到他主要是负责**资源版本信息的构建与加载**的，并且在构建和加载版本信息时就已经用到了VFS。

```c#
//构建版本信息
public static void BuildVersions(string outputPath, string[] bundles, int version)
{
    var path = outputPath + "/" + Filename;
    if (File.Exists(path))
    {
        File.Delete(path);
    }

    var dataPath = outputPath + "/" + Dataname;
    if (File.Exists(dataPath))
    {
        File.Delete(dataPath);
    }

    var disk = new VDisk();
    foreach (var file in bundles)
    {
        using (var fs = File.OpenRead(outputPath + "/" + file))
        {
            disk.AddFile(file, fs.Length, Utility.GetCRC32Hash(fs));
        }
    }

    disk.name = dataPath;
    disk.Save();

    using (var stream = File.OpenWrite(path))
    {
        var writer = new BinaryWriter(stream);
        writer.Write(version);
        writer.Write(disk.files.Count + 1);
        using (var fs = File.OpenRead(dataPath))
        {
            var file = new VFile {name = Dataname, len = fs.Length, hash = Utility.GetCRC32Hash(fs)};
            file.Serialize(writer);
        }

        foreach (var file in disk.files)
        {
            file.Serialize(writer);
        }
    }
}
```

```c#
//加载版本信息
public static List<VFile> LoadVersions(string filename, bool update = false)
{
    var data = update ? _updateData : _baseData;
    data.Clear();
    using (var stream = File.OpenRead(filename))
    {
        var reader = new BinaryReader(stream);
        var list = new List<VFile>();
        var ver = reader.ReadInt32();
        Debug.Log("LoadVersions:" + ver);
        var count = reader.ReadInt32();
        for (var i = 0; i < count; i++)
        {
            var version = new VFile();
            version.Deserialize(reader);
            list.Add(version);
            data[version.name] = version;
        }

        return list;
    }
}
```

此外，Versions因为底层实现依赖了VFS，所以支持任意格式的资源文件的版本管理，可以非常方便的对Wwise、Fmod等自定义格式的文件进行版本控制。



#### Updater.cs

```c#
//向服务器请求版本信息
private IEnumerator RequestVersions()
{
    OnMessage("正在获取版本信息...");
    var request = UnityWebRequest.Get(GetDownloadURL(Versions.Filename));
    request.downloadHandler = new DownloadHandlerFile(_savePath + Versions.Filename);
    yield return request.SendWebRequest();
    if (!string.IsNullOrEmpty(request.error))
    {
        var mb = MessageBox.Show("提示", string.Format("获取服务器版本失败：{0}", request.error), "重试", "退出");
        yield return mb;
        if (mb.isOk)
        {
            StartUpdate();
        }
        else
        {
            Quit();
            MessageBox.Dispose();
        } 
        yield break; // yield break;
    } 
    request.Dispose();
    //在这个方法里打印了  LoadVersions:5
    _versions = Versions.LoadVersions(_savePath + Versions.Filename, true);
}
```

既然这里提到了Updater，就拔丝抽茧把这个Update流程看一下吧，先来看下它的初始化部分

```C#
private void Start()
{
    //初始化downloder并绑定委托
    _downloader = gameObject.AddComponent<Downloader>();
    _downloader.onUpdate = OnUpdate;
    _downloader.onFinished = OnComplete;
	//获取版本信息文件保存的本地位置
    _savePath = Application.persistentDataPath + '/';
    Assets.updatePath = _savePath;
    //获取云端Bundles的目标平台
    _platform = GetPlatformForAssetBundles(Application.platform);
}
```

当我们点击 TOUCH TO START 按钮时，会执行

```c#
public void StartUpdate()
{
    //告知UI进行初始化
    OnStart();
    //如果当前Check协程不为空，就终止
    if (checking != null)
    {
        StopCoroutine(checking);
    }

    checking = Checking();
    //重新启用Check协程
    StartCoroutine(checking);
}
```

```c#
private IEnumerator Checking()
{
    if (!Directory.Exists(_savePath))
    {
        Directory.CreateDirectory(_savePath);
    } 
    //询问是否开启VFS
    yield return RequestVFS();
    //加载本地版本信息文件，如果StreamingAssets下面有资源会询问是否复制资源
    yield return RequestCopy(); 
    //请求并加载云端版本信息文件
    yield return RequestVersions(); 
    //与本地版本信息进行对比，并询问进行更新
    if (_versions.Count > 0)
    {
        OnMessage("正在检查版本信息..."); 
        //准备下载内容，根据是否开启VFS而选择下载不同的文件
        PrepareDownloads();
        var totalSize = _downloader.size;
        if (totalSize > 0)
        {
            var tips = string.Format("发现内容更新，总计需要下载 {0} 内容", Downloader.GetDisplaySize(totalSize));
            var mb = MessageBox.Show("提示", tips, "下载", "跳过");
            yield return mb;
            if (mb.isOk)
            {
                //开始正式下载资源，并记录当前的下载进度，用于做断点续传
                _downloader.StartDownload();
                yield break;
            }
        }
    }
    //所有文件更新完成，再次更新本地版本信息文件
    OnComplete();  
}
```

总结一下，热更新流程



#### Download.cs和Downloader.cs

Download.cs继承DownloadHandlerScript实现了一套自己的下载处理逻辑，而Downloader.cs就是用来管理所有的Download对象的，并且做了一些附加功能，比如记录当前下载进度，用于做断点续传，不过Demo作者并没有演示，只是预留了接口，大家可以自行查看。

![image-20200711190631370](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200711190631370.png)

### 加载场景

好了，这个时候我们已经把所有资源更新完毕了，开始进入场景。

```c#
private IEnumerator LoadGameScene()
{
    OnMessage("正在初始化");
    Assets.runtimeMode = true;
    //初始化AB系统，加载Manifest文件
    var init = Assets.Initialize();
    yield return init;  
    if (string.IsNullOrEmpty(init.error))
    {
        init.Release(); 
        OnProgress(0);
        OnMessage("加载游戏场景");
        //异步加载 Game.unity 这里使用了更智能的寻址模式，在上一个版本中 需要输出 Assets/XAsset/Demo/Scenes/Game.unity, 具体参考 SearchPath
        var scene = Assets.LoadSceneAsync(gameScene, false);
        while (!scene.isDone)
        {
            OnProgress(scene.progress);
            yield return null;
        }
    }
    else
    {
        init.Release();
        var mb = MessageBox.Show("提示", "初始化异常错误：" + init.error + "请联系技术支持");
        yield return mb;
        Quit();
    }
}
```

其中最主要的，是Assets.Initialize();初始化工作，以及加载场景的那两句代码

其实在xasset中，加载AB资源非常方便

```c#
//异步加载场景
public static SceneAssetRequest LoadSceneAsync(string path, bool additive)
{
    if (string.IsNullOrEmpty(path))
    {
        Debug.LogError("invalid path");
        return null;
    }

    path = GetExistPath(path);
    var asset = new SceneAssetAsyncRequest(path, additive);
    if (! additive)
    {
        if (_runningScene != null)
        {
            _runningScene.Release();;
            _runningScene = null;
        }
        _runningScene = asset;
    }
    //加载资源
    asset.Load();
    //资源引用计数
    asset.Retain();
    _scenes.Add(asset);
    Log(string.Format("LoadScene:{0}", path));
    return asset;
}

//异步加载资源，path为资源寻址路径，Type为资源类型
public static AssetRequest LoadAssetAsync(string path, Type type)
{
    return LoadAsset(path, type, true);
}

//同步加载资源，path为资源寻址路径，Type为资源类型
public static AssetRequest LoadAsset(string path, Type type)
{
    return LoadAsset(path, type, false);
}
```

### 加载资源

我们接着看Demo，点击顶部下拉列表，随便选择一个资源，点击加载即可看到效果

![image-20200711210806913](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200711210806913.png)

我们来看看他代码怎么实现的

```c#
private IEnumerator LoadAsset ()
{
	if (_assets == null || _assets.Length == 0) 
    {
		yield break;
	} 
    //根据当前下拉框选择的值去选取AB路径
	var path = _assets [_optionIndex];
    //获取拓展名
	var ext = Path.GetExtension (path);
	if (ext.Equals (".png", StringComparison.OrdinalIgnoreCase)) 
    {
        //拿着这个路径去加载精灵图片
		var request = LoadSprite (path);
		yield return request;
		if (!string.IsNullOrEmpty (request.error)) 
        {
			request.Release ();
			yield break;
		} 
        //实例化
		var go = Instantiate (temp.gameObject, temp.transform.parent);
		go.SetActive (true);
		go.name = request.asset.name;
		var image = go.GetComponent<Image> ();
        //设置从AB加载出来的精灵图片
		image.sprite = request.asset as Sprite; 
		_gos.Add (go);
	}
}
```

那么这个AB路径是怎么回事呢，我们断点看一下，发现都是**全路径**，所幸xasset提供了Assets.GetAllAssetPaths();来获取所有AB路径名，我们可以自己封装一个API，做一个Dictionary<string,string> AllAssetPathShortName，Key为单纯的资产名，例如Btn_Buy1_h，Value就是全路径名，这样使用起来也比较方便。

![image-20200711212020829](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200711212020829.png)

我们来看看资源加载这一块的底层源码实现

```c#
private static AssetRequest LoadAsset(string path, Type type, bool async)
{
    if (string.IsNullOrEmpty(path))
    {
        Debug.LogError("invalid path");
        return null;
    }

    path = GetExistPath(path);

    //先尝试从已加载Asset获取目标Asset
    AssetRequest request;
    if (_assets.TryGetValue(path, out request))
    {
        //引用计数+1
        request.Retain();
        _loadingAssets.Add(request);
        return request;
    }

    //如果没找到就需要去获取AB
    string assetBundleName;
    //如果此AB已存在于本地记录（从Manifest文件读取的），就直接取得AB名，准备加载AB
    if (GetAssetBundleName(path, out assetBundleName))
    {
        request = async
            ? new BundleAssetAsyncRequest(assetBundleName)
            : new BundleAssetRequest(assetBundleName);
    }
    else
    {
        //如果此AB在本地记录未找到
        //如果是网络路径/本地路径（注意一定要有以下前缀之一，否则会被忽略而取不到对象）
        if (path.StartsWith("http://", StringComparison.Ordinal) ||
            path.StartsWith("https://", StringComparison.Ordinal) ||
            path.StartsWith("file://", StringComparison.Ordinal) ||
            path.StartsWith("ftp://", StringComparison.Ordinal) ||
            path.StartsWith("jar:file://", StringComparison.Ordinal))
            request = new WebAssetRequest();
        else
            //如果是本地路径（事实上这个是用AssetDatabase.LoadAssetAtPath去编辑器找的，所以想要读取本地的非AB文件还是用上面的UWR吧，注意加上前缀)
            request = new AssetRequest();
    }

    request.url = path;
    request.assetType = type;
    //新增资产请求
    AddAssetRequest(request);
    //引用计数+1
    request.Retain();
    Log(string.Format("LoadAsset:{0}", path));
    return request;
}
```

总结，对与资源加载，我们只需要提供AB全路径名以及目标类型，即可加载AB，并且通过**.asset**取得目标对象。

### 卸载资源

只加载资源，不卸载资源可不行，我们来看看xasset是怎么处理资源卸载这一块逻辑的。

同样看Demo的卸载资源选项

```c#
private IEnumerator UnloadAssets ()
{
	foreach (var image in _gos) 
    {
		DestroyImmediate (image);
	}
	_gos.Clear ();
    
	foreach (var request in _requests) 
    {
        //减少引用计数
		request.Release ();
	}

	_requests.Clear ();
	yield return null;
    //卸载所有未被引用的资产
	Assets.RemoveUnusedAssets ();
}
```

来看看底层源码实现

```c#
//里面具体的卸载资源逻辑会因为资源类型不同而不同
public static void RemoveUnusedAssets()
{
    //先准备移除无用资产
    foreach (var item in _assets)
    {
        if (item.Value.IsUnused())
        {
            _unusedAssets.Add(item.Value);  
        }
    } 
    foreach (var request in _unusedAssets)
    {
        _assets.Remove(request.url);
    }
    //再准备移除无用AB
    foreach (var item in _bundles)
    {
        if (item.Value.IsUnused())
        {
            _unusedBundles.Add(item.Value);
        }
    } 
    foreach (var request in _unusedBundles)
    {
        _bundles.Remove(request.url);
    }
}
```

## 打包

对于打包这一块的支持，xasset做的也非常到位，自动分析依赖，冗余

xasset的打包方法是对文件夹进行Applay Rule

具体分类

- Text：此文件夹下的每个文本文件都会各自打成一个AB包
- Prefab：此文件夹下的每个Prefab都会各自打成一个AB包
- Png：此文件夹下的每个图片都会各自打成一个AB包
- Material：此文件夹下的每个Material都会各自打成一个AB包
- Controller：此文件夹下的每个Controller都会各自打成一个AB包
- Asset：此文件夹下的每个Asset都会各自打成一个AB包
- Scene：此文件夹下的每个场景都会各自打成一个AB包
- Directory：此文件夹下的每个文件夹都会各自打成一个AB包

以Demo为例，我们**每个Scene一个AB，每个UI下的子文件夹一个AB**

![image-20200712173410551](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200712173410551.png)

![image-20200712173346454](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200712173346454.png)

![image-20200711233323572](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200711233323572.png)

然后为了打包，我们需要填写起始Scene，注意不包含需要热更的Scene

![image-20200712173454019](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200712173454019.png)

效果应该如下

![image-20200712173627053](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200712173627053.png)

最后我们Build Bundles

![image-20200711233641134](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200711233641134.png)



会生成在**项目目录/DLC/目标平台下**

![image-20200711233558636](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200711233558636.png)

## 上传文件服务器

上传AB文件到服务器时要注意，需要把整个DLC文件夹都上传到服务器

xasset自带了HFS工具，这是一个本地的资源服务器，我们可以用它做实验（不过俺打不开，重新下了一个）

![image-20200711233838884](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200711233838884.png)

![image-20200711234707645](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200711234707645.png)

![image-20200711234722587](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200711234722587.png)

把这个网址复制到Unity，每个人可能都不一样

![image-20200711234841606](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200711234841606.png)

![image-20200711234925114](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200711234925114.png)

## 特性

### VFS

Virtual File System（虚拟文件系统），通过Virtual File和Virtual Disk来实现一套I/O方案，用自定义的格式对所有资源文件进行打包防止资源被ABE或AS之类的工具轻易提取，除了安全性得到提升外，它在测试的Android设备上的IO性能也有客观的提升，参考

![image-20200712171833993](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200712171833993.png)

### 惰性GC

之所以叫惰性GC，是因为和上一个版本相比，上一个版本是每帧都会检查和清理未使用的资源，这个版本底层只会在切换场景或者主动调用Assets.RemoveUnusedAssets();的时候才会清理未使用的资源，这样用户可以按需调整资源回收的频率，在没有内存压力的时候，不回收可以获得更好的性能。

## 架构流程图

![xasset架构图 (1)](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/xasset架构图 (1).png)
