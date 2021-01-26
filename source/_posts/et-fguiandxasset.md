---
title: ET&&FGUI接入xasset流程
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

## 前言

技能系统暂时告一段落，现在要花点时间规范一下客户端这边的资源管理以及一些流程优化，这里选择轻量高效资源管理框架xasset：https://github.com/xasset/xasset

版本为：https://github.com/xasset/xasset/commit/3d4983cd24ff92a63156c8078caf34b20d2d4c02

代码量很少，一天就能看个差不多，但是质量很高，如果只追求使用的话，是可以开箱即用的。

另外我对xasset底层源码做了一些修改，主要是为了迎合我们ET传统艺能await/async样式代码，所以推荐大家直接使用我项目（下文的Moba项目）中的xasset源码

想要入门此资源管理框架的可以查看：https://www.lfzxb.top/xasset_base/

以及视频教程：https://www.bilibili.com/video/BV15A411v7nT

为了方便大家参考，可以去我对Moba项目：https://gitee.com/NKG_admin/NKGMobaBasedOnET，来查看更加具体的接入代码，目前全部功能测试通过（资源加载，普通热更新，VFS热更新，FGUI适配）

## 流程预演

为了流畅接入一个框架，可以先把大体流程先构思一下

xasset使用主要分为3块

- 打包工具配置（直接拿过来改几个路径即可使用（因为要打包到我们资源服务器指定的目录下面））
- 本地资源服务器（使用ET自带的Web资源服务器即可）
- 运行时类库（非常简单的接口使用，完全不需要关心资源管理）

### 打包工具

xasset打包流程分ApplyRule（配置打包规则），BuildRule（自动分析依赖关系，优化资源冗余，解决资源冲突），BuildBundle（出包）三步走，具体内容可参照上文链接

### 本地资源服务器

这一块是ET的内容，事实上我们只需要修改代码，把资源打到文件资源服务器指定的目录就行了

### 运行时类库

xasset运行时接入相对于前面两块内容较为复杂，主要包括资源热更新模块接入，API封装，FGUI资源加载适配

## 正式开始

### xasset导入

首先导入xasset到ET，主要有Editor和Runtime这两部分内容

首先是Editor部分，把Assets/XAsset/Editor文件夹放到我们ET中的Editor文件夹

![image-20201016123426783](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20201016123426.png)

Assets/XAsset/Runtime文件夹放到我们ET中的ThirdParty，注意移除UI文件夹，因为他是和xasset的官方Demo耦合的

![image-20201016123338059](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20201016123338.png)

会有一些Updater脚本的报错，但是不要怕，我们接下来解决他

它里面的报错主要是引用的Message Mono类（一个用于显示对话框的类）找不到导致的，所以我们把这部分内容改成用Debug输出或者直接删掉就行了

这里提供一个我的修改版本的

```cs
//
// Updater.cs
//
// Author:
//       fjy <jiyuan.feng@live.com>
//
// Copyright (c) 2020 fjy
//
// Permission is hereby granted, free of charge, to any person obtaining a copy
// of this software and associated documentation files (the "Software"), to deal
// in the Software without restriction, including without limitation the rights
// to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
// copies of the Software, and to permit persons to whom the Software is
// furnished to do so, subject to the following conditions:
//
// The above copyright notice and this permission notice shall be included in
// all copies or substantial portions of the Software.
//
// THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
// IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
// FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
// AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
// LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
// OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
// THE SOFTWARE.

using System;
using System.Collections;
using System.Collections.Generic;
using System.IO;
using UnityEngine;
using UnityEngine.Networking;

namespace libx
{
    public enum Step
    {
        Wait,
        Copy,
        Coping,
        Versions,
        Prepared,
        Download,
        Completed,
    }

    [RequireComponent(typeof (Downloader))]
    public class Updater: MonoBehaviour
    {
        public Step Step;

        public Action ResPreparedCompleted;

        public float UpdateProgress;
        
        public bool DevelopmentMode;
        
        public bool EnableVFS = true;

        [SerializeField]
        private string baseURL = "http://127.0.0.1:7888/DLC/";
        private Downloader _downloader;
        private string _platform;
        private string _savePath;
        private List<VFile> _versions = new List<VFile>();

        public void OnMessage(string msg)
        {
            Debug.Log(msg);
        }

        public void OnProgress(float progress)
        {
            UpdateProgress = progress;
        }

        private void Awake()
        {
            _downloader = gameObject.GetComponent<Downloader>();
            _downloader.onUpdate = OnUpdate;
            _downloader.onFinished = OnComplete;

            _savePath = string.Format("{0}/DLC/", Application.persistentDataPath);
            _platform = GetPlatformForAssetBundles(Application.platform);

            this.Step = Step.Wait;

            Assets.updatePath = _savePath;
        }

        private void OnUpdate(long progress, long size, float speed)
        {
            OnMessage(string.Format("下载中...{0}/{1}, 速度：{2}",
                Downloader.GetDisplaySize(progress),
                Downloader.GetDisplaySize(size),
                Downloader.GetDisplaySpeed(speed)));

            OnProgress(progress * 1f / size);
        }

        private IEnumerator _checking;

        public void StartUpdate()
        {
            Debug.Log("StartUpdate.Development:" + this.DevelopmentMode);
#if UNITY_EDITOR
            if (this.DevelopmentMode)
            {
                Assets.runtimeMode = false;
                StartCoroutine(LoadGameScene());
                return;
            }
#endif

            if (_checking != null)
            {
                StopCoroutine(_checking);
            }

            _checking = Checking();

            StartCoroutine(_checking);
        }

        private void AddDownload(VFile item)
        {
            _downloader.AddDownload(GetDownloadURL(item.name), item.name, _savePath + item.name, item.hash, item.len);
        }

        private void PrepareDownloads()
        {
            if (this.EnableVFS)
            {
                var path = string.Format("{0}{1}", _savePath, Versions.Dataname);
                if (!File.Exists(path))
                {
                    AddDownload(_versions[0]);
                    return;
                }

                Versions.LoadDisk(path);
            }

            for (var i = 1; i < _versions.Count; i++)
            {
                var item = _versions[i];
                if (Versions.IsNew(string.Format("{0}{1}", _savePath, item.name), item.len, item.hash))
                {
                    AddDownload(item);
                }
            }
        }

        private static string GetPlatformForAssetBundles(RuntimePlatform target)
        {
            // ReSharper disable once SwitchStatementMissingSomeCases
            switch (target)
            {
                case RuntimePlatform.Android:
                    return "Android";
                case RuntimePlatform.IPhonePlayer:
                    return "iOS";
                case RuntimePlatform.WebGLPlayer:
                    return "WebGL";
                case RuntimePlatform.WindowsPlayer:
                case RuntimePlatform.WindowsEditor:
                    return "Windows";
                case RuntimePlatform.OSXEditor:
                case RuntimePlatform.OSXPlayer:
                    return "iOS"; // OSX
                default:
                    return null;
            }
        }

        private string GetDownloadURL(string filename)
        {
            return string.Format("{0}{1}/{2}", baseURL, _platform, filename);
        }

        private IEnumerator Checking()
        {
            if (!Directory.Exists(_savePath))
            {
                Directory.CreateDirectory(_savePath);
            }

            this.Step = Step.Copy;

            if (this.Step == Step.Copy)
            {
                yield return RequestCopy();
            }

            if (this.Step == Step.Coping)
            {
                var path = _savePath + Versions.Filename + ".tmp";
                var versions = Versions.LoadVersions(path);
                var basePath = GetStreamingAssetsPath() + "/";
                yield return UpdateCopy(versions, basePath);
                this.Step = Step.Versions;
            }

            if (this.Step == Step.Versions)
            {
                yield return RequestVersions();
            }

            if (this.Step == Step.Prepared)
            {
                OnMessage("正在检查版本信息...");
                var totalSize = _downloader.size;
                if (totalSize > 0)
                {
                    Debug.Log($"发现内容更新，总计需要下载 {Downloader.GetDisplaySize(totalSize)} 内容");
                    _downloader.StartDownload();
                    this.Step = Step.Download;
                }
                else
                {
                    OnComplete();
                }
            }
        }

        private IEnumerator RequestVersions()
        {
            OnMessage("正在获取版本信息...");
            if (Application.internetReachability == NetworkReachability.NotReachable)
            {
                Debug.LogError("请检查网络连接状态");
                yield break;
            }

            var request = UnityWebRequest.Get(GetDownloadURL(Versions.Filename));
            request.downloadHandler = new DownloadHandlerFile(_savePath + Versions.Filename);
            yield return request.SendWebRequest();
            var error = request.error;
            request.Dispose();
            if (!string.IsNullOrEmpty(error))
            {
                Debug.LogError($"获取服务器版本失败：{error}");
                yield break;
            }

            try
            {
                _versions = Versions.LoadVersions(_savePath + Versions.Filename, true);
                if (_versions.Count > 0)
                {
                    PrepareDownloads();
                    this.Step = Step.Prepared;
                }
                else
                {
                    OnComplete();
                }
            }
            catch (Exception e)
            {
                Debug.LogException(e);
                Debug.LogError("版本文件加载失败");
            }
        }

        private static string GetStreamingAssetsPath()
        {
            if (Application.platform == RuntimePlatform.Android)
            {
                return Application.streamingAssetsPath;
            }

            if (Application.platform == RuntimePlatform.WindowsPlayer ||
                Application.platform == RuntimePlatform.WindowsEditor)
            {
                return "file:///" + Application.streamingAssetsPath;
            }

            return "file://" + Application.streamingAssetsPath;
        }

        private IEnumerator RequestCopy()
        {
            var v1 = Versions.LoadVersion(_savePath + Versions.Filename);
            var basePath = GetStreamingAssetsPath() + "/";
            var request = UnityWebRequest.Get(basePath + Versions.Filename);
            var path = _savePath + Versions.Filename + ".tmp";
            request.downloadHandler = new DownloadHandlerFile(path);
            yield return request.SendWebRequest();
            if (string.IsNullOrEmpty(request.error))
            {
                var v2 = Versions.LoadVersion(path);
                if (v2 > v1)
                {
                    Debug.Log("将资源解压到本地");
                    this.Step = Step.Coping;
                }
                else
                {
                    Versions.LoadVersions(path);
                    this.Step = Step.Versions;
                }
            }
            else
            {
                this.Step = Step.Versions;
            }

            request.Dispose();
        }

        private IEnumerator UpdateCopy(IList<VFile> versions, string basePath)
        {
            var version = versions[0];
            if (version.name.Equals(Versions.Dataname))
            {
                var request = UnityWebRequest.Get(basePath + version.name);
                request.downloadHandler = new DownloadHandlerFile(_savePath + version.name);
                var req = request.SendWebRequest();
                while (!req.isDone)
                {
                    OnMessage("正在复制文件");
                    OnProgress(req.progress);
                    yield return null;
                }

                request.Dispose();
            }
            else
            {
                for (var index = 0; index < versions.Count; index++)
                {
                    var item = versions[index];
                    var request = UnityWebRequest.Get(basePath + item.name);
                    request.downloadHandler = new DownloadHandlerFile(_savePath + item.name);
                    yield return request.SendWebRequest();
                    request.Dispose();
                    OnMessage(string.Format("正在复制文件：{0}/{1}", index, versions.Count));
                    OnProgress(index * 1f / versions.Count);
                }
            }
        }

        private void OnComplete()
        {
            if (this.EnableVFS)
            {
                var dataPath = _savePath + Versions.Dataname;
                var downloads = _downloader.downloads;
                if (downloads.Count > 0 && File.Exists(dataPath))
                {
                    OnMessage("更新本地版本信息");
                    var files = new List<VFile>(downloads.Count);
                    foreach (var download in downloads)
                    {
                        files.Add(new VFile { name = download.name, hash = download.hash, len = download.len, });
                    }

                    var file = files[0];
                    if (!file.name.Equals(Versions.Dataname))
                    {
                        Versions.UpdateDisk(dataPath, files);
                    }
                }

                Versions.LoadDisk(dataPath);
            }

            OnProgress(1);
            OnMessage($"更新完成，版本号：{Versions.LoadVersion(_savePath + Versions.Filename)}");

            StartCoroutine(LoadGameScene());
        }

        private IEnumerator LoadGameScene()
        {
            OnMessage("正在初始化");
            var init = Assets.Initialize();
            yield return init;
            this.Step = Step.Completed;
            if (string.IsNullOrEmpty(init.error))
            {
                init.Release();
                OnProgress(0);
                OnMessage("加载游戏场景");
                ResPreparedCompleted?.Invoke();
            }
            else
            {
                init.Release();
                Debug.LogError($"初始化异常错误:{init.error},请联系技术支持");
            }
        }
    }
}
```

最后因为我们Model层会用到xasset，所以引用asmdef文件,Hotfix同理

![image-20201014161652452](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20201014161652.png)

### 替换ET资源管理模块

因为我们使用xasset全盘托管资源管理（资源加载，热更新），所以我们只需要对其进行封装即可

#### 移除所有打包模块

Editor下的打包模块相关代码都可以删除

#### ResourceComponent

其中有一些api需要对xasset源码进行拓展，可以参考Moba项目的xasset源码

```cs
using libx;
using UnityEngine;

namespace ETModel
{
    public class ResourcesComponent: Component
    {
        #region Assets

        /// <summary>
        /// 加载资源，path需要是全路径
        /// </summary>
        /// <param name="path"></param>
        /// <typeparam name="T"></typeparam>
        /// <returns></returns>
        public T LoadAsset<T>(string path) where T : UnityEngine.Object
        {
            AssetRequest assetRequest = Assets.LoadAsset(path, typeof (T));
            return (T) assetRequest.asset;
        }

        /// <summary>
        /// 异步加载资源，path需要是全路径
        /// </summary>
        /// <param name="path"></param>
        /// <typeparam name="T"></typeparam>
        /// <returns></returns>
        public ETTask<T> LoadAssetAsync<T>(string path) where T : UnityEngine.Object
        {
            ETTaskCompletionSource<T> tcs = new ETTaskCompletionSource<T>();
            AssetRequest assetRequest = Assets.LoadAssetAsync(path, typeof (T));
            assetRequest.completed = (arq) => { tcs.SetResult((T) arq.asset); };
            return tcs.Task;
        }

        /// <summary>
        /// 卸载资源，path需要是全路径
        /// </summary>
        /// <param name="path"></param>
        public void UnLoadAsset(string path)
        {
            Assets.UnloadAsset(path);
        }

        #endregion

        #region Scenes

        /// <summary>
        /// 加载场景，path需要是全路径
        /// </summary>
        /// <param name="path"></param>
        /// <returns></returns>
        public ETTask<SceneAssetRequest> LoadSceneAsync(string path)
        {
            ETTaskCompletionSource<SceneAssetRequest> tcs = new ETTaskCompletionSource<SceneAssetRequest>();
            SceneAssetRequest sceneAssetRequest = Assets.LoadSceneAsync(path, false);
            sceneAssetRequest.completed = (arq) =>
            {
                tcs.SetResult(sceneAssetRequest);
            };
            return tcs.Task;
        }

        /// <summary>
        /// 卸载场景，path需要是全路径
        /// </summary>
        /// <param name="path"></param>
        public void UnLoadScene(string path)
        {
            Assets.UnloadScene(path);
        }

        #endregion

        #region AssetBundles

        /// <summary>
        /// 加载AB,path需要为全路径
        /// </summary>
        /// <param name="path"></param>
        /// <returns></returns>
        public AssetBundle LoadAssetBundle(string path)
        {
            string assetBundleName;
            Assets.GetAssetBundleName(path, out assetBundleName);
            AssetRequest assetRequest = Assets.LoadBundle(assetBundleName);
            return (AssetBundle) assetRequest.asset;
        }

        /// <summary>
        /// 异步加载AB，path需要为全路径
        /// </summary>
        /// <param name="path"></param>
        /// <returns></returns>
        public ETTask<AssetBundle> LoadAssetBundleAsync(string path)
        {
            string assetBundleName;
            Assets.GetAssetBundleName(path, out assetBundleName);
            ETTaskCompletionSource<AssetBundle> tcs = new ETTaskCompletionSource<AssetBundle>();
            AssetRequest assetRequest = Assets.LoadBundleAsync(assetBundleName);
            assetRequest.completed = (arq) => { tcs.SetResult((AssetBundle) arq.asset); };
            return tcs.Task;
        }

        /// <summary>
        /// 卸载资源，path需要是全路径
        /// </summary>
        /// <param name="path"></param>
        public void UnLoadAssetBundle(string path)
        {
            string assetBundleName;
            Assets.GetAssetBundleName(path, out assetBundleName);
            Assets.UnloadBundle(assetBundleName);
        }

        #endregion
    }
}
```

#### BundleDownloaderComponent

```cs
using System;
using System.Collections.Generic;
using System.IO;
using System.Threading.Tasks;
using libx;
using UnityEngine;

namespace ETModel
{
    [ObjectSystem]
    public class UiBundleDownloaderComponentAwakeSystem: AwakeSystem<BundleDownloaderComponent>
    {
        public override void Awake(BundleDownloaderComponent self)
        {
            self.Updater = GameObject.FindObjectOfType<Updater>();
        }
    }
    
    [ObjectSystem]
    public class UiBundleDownloaderComponentSystem: UpdateSystem<BundleDownloaderComponent>
    {
        public override void Update(BundleDownloaderComponent self)
        {
            if (self.Updater.Step == Step.Completed)
            {
                self.Tcs.SetResult();
            }
        }
    }

    /// <summary>
    /// 封装XAsset Updater
    /// </summary>
    public class BundleDownloaderComponent: Component
    {
        public Updater Updater;

        public ETTaskCompletionSource Tcs;

        public ETTask StartUpdate()
        {
            Tcs = new ETTaskCompletionSource();
            Updater.ResPreparedCompleted = () =>
            {
                Tcs.SetResult();
            };
            Updater.StartUpdate();
            
            return Tcs.Task;
        }
    }
}
```

#### ABPathUtilities

因为xasset使用全路径对资源进行加载，所以我们要提供路径拓展

```cs
//------------------------------------------------------------
// Author: 烟雨迷离半世殇
// Mail: 1778139321@qq.com
// Data: 2020年10月14日 22:27:15
//------------------------------------------------------------

namespace ETModel
{
    /// <summary>
    /// AB实用函数集，主要是路径拼接
    /// </summary>
    public class ABPathUtilities
    {
        public static string GetTexturePath(string fileName)
        {
            return $"Assets/Bundles/Altas/{fileName}.prefab";
        }
        
        public static string GetFGUIDesPath(string fileName)
        {
            return $"Assets/Bundles/FUI/{fileName}.bytes";
        }
        
        public static string GetFGUIResPath(string fileName)
        {
            return $"Assets/Bundles/FUI/{fileName}.png";
        }
        
        public static string GetNormalConfigPath(string fileName)
        {
            return $"Assets/Bundles/Independent/{fileName}.prefab";
        }
        
        public static string GetSoundPath(string fileName)
        {
            return $"Assets/Bundles/Sounds/{fileName}.prefab";
        }
        
        public static string GetSkillConfigPath(string fileName)
        {
            return $"Assets/Bundles/SkillConfigs/{fileName}.prefab";
        }
        
        public static string GetUnitPath(string fileName)
        {
            return $"Assets/Bundles/Unit/{fileName}.prefab";
        }
        
        public static string GetScenePath(string fileName)
        {
            return $"Assets/Scenes/{fileName}.unity";
        }
    }
}
```

### 打包配置

#### BuildlScript

把脚本中对应路径进行修改即可

```cs
	public static class BuildScript
	{
        //打包AB的输出路径
		public static string ABOutPutPath  = c_RelativeDirPrefix + GetPlatformName(); 
		//前缀
		private const string c_RelativeDirPrefix = "../Release/";
		//Rules.asset保存路径
		private const string c_RulesDir = "Assets/Res/XAsset/Rules.asset";
		....
	}
```

#### Assets

按需求修改Manifest保存路径即可

```cs
public sealed class Assets: MonoBehaviour
{
	public static readonly string ManifestAsset = "Assets/Res/XAsset/Manifest.asset";
	...
}
```

### 适配FGUI

因为FGUI提供的API是对于AssetBundle而言的，而xasset设计理念是不关心AssetBundle，所以在ResourceComponent里暴露了xasset LoadBundle的API

然后FGUI提供的API还对AssetBundle进行了强制Unload(true);它与xasset的AB管理冲突，所以要把他去掉

```cs
		/// <summary>
		/// Add a UI package from two assetbundles with a optional main asset name.
		/// </summary>
		/// <param name="desc">A assetbunble contains description file.</param>
		/// <param name="res">A assetbundle contains resources.</param>
		/// <param name="mainAssetName">Main asset name. e.g. Basics_fui.bytes</param>
		/// <returns>UIPackage</returns>
		public static UIPackage AddPackage(AssetBundle desc, AssetBundle res, string mainAssetName)
		{
			byte[] source = null;
#if (UNITY_5 || UNITY_5_3_OR_NEWER)
			if (mainAssetName != null)
			{
				TextAsset ta = desc.LoadAsset<TextAsset>(mainAssetName);
				if (ta != null)
					source = ta.bytes;
			}
			else
			{
				string[] names = desc.GetAllAssetNames();
				string searchPattern = "_fui";
				foreach (string n in names)
				{
					if (n.IndexOf(searchPattern) != -1)
					{
						TextAsset ta = desc.LoadAsset<TextAsset>(n);
						if (ta != null)
						{
							source = ta.bytes;
							mainAssetName = Path.GetFileNameWithoutExtension(n);
							break;
						}
					}
				}
			}
#else
			if (mainAssetName != null)
			{
				TextAsset ta = (TextAsset)desc.Load(mainAssetName, typeof(TextAsset));
				if (ta != null)
					source = ta.bytes;
			}
			else
			{
				source = ((TextAsset)desc.mainAsset).bytes;
				mainAssetName = desc.mainAsset.name;
			}
#endif
			if (source == null)
				throw new Exception("FairyGUI: no package found in this bundle.");

			//TODO 此处因为会与XAsset的资源管理冲突，所以先去掉
			// if (desc != res)
			// 	desc.Unload(true);

			ByteBuffer buffer = new ByteBuffer(source);

			UIPackage pkg = new UIPackage();
			pkg._resBundle = res;
			pkg._fromBundle = true;
			int pos = mainAssetName.IndexOf("_fui");
			string assetNamePrefix;
			if (pos != -1)
				assetNamePrefix = mainAssetName.Substring(0, pos);
			else
				assetNamePrefix = mainAssetName;
			if (!pkg.LoadPackage(buffer, res.name, assetNamePrefix))
				return null;

			_packageInstById[pkg.id] = pkg;
			_packageInstByName[pkg.name] = pkg;
			_packageList.Add(pkg);

			return pkg;
		}
```

最后就是我们对UIPackage操作的封装了

```cs
using System.Collections.Generic;
using System.Threading.Tasks;
using FairyGUI;
using UnityEngine;

namespace ETModel
{
    /// <summary>
    /// 管理所有UI Package
    /// </summary>
    public class FUIPackageComponent: Component
    {
        public const string FUI_PACKAGE_DIR = "Assets/Bundles/FUI";

        private readonly Dictionary<string, UIPackage> packages = new Dictionary<string, UIPackage>();

        public async ETTask AddPackageAsync(string type)
        {
            if (this.packages.ContainsKey(type)) return;
            UIPackage uiPackage;
            if (Define.ResModeIsEditor)
            {
                uiPackage = UIPackage.AddPackage($"{FUI_PACKAGE_DIR}/{type}");
            }
            else
            {
                string uiBundleDesName = $"{type}_fui";
                string uiBundleResName = $"{type}_atlas0";
                ResourcesComponent resourcesComponent = Game.Scene.GetComponent<ResourcesComponent>();

                AssetBundle desAssetBundle = await resourcesComponent.LoadAssetBundleAsync(ABPathUtilities.GetFGUIDesPath(uiBundleDesName));
                AssetBundle resAssetBundle = await resourcesComponent.LoadAssetBundleAsync(ABPathUtilities.GetFGUIResPath(uiBundleResName));
                uiPackage = UIPackage.AddPackage(desAssetBundle, resAssetBundle);
            }

            packages.Add(type, uiPackage);
        }

        public void RemovePackage(string type)
        {
            UIPackage package;

            if (packages.TryGetValue(type, out package))
            {
                var p = UIPackage.GetByName(package.name);

                if (p != null)
                {
                    UIPackage.RemovePackage(package.name);
                }

                packages.Remove(package.name);
            }

            if (!Define.ResModeIsEditor)
            {
                string uiBundleDesName = $"{type}_fui";
                string uiBundleResName = $"{type}_atlas0";
                Game.Scene.GetComponent<ResourcesComponent>().UnLoadAssetBundle(ABPathUtilities.GetFGUIDesPath(uiBundleDesName));
                Game.Scene.GetComponent<ResourcesComponent>().UnLoadAssetBundle(ABPathUtilities.GetFGUIResPath(uiBundleResName));
            }
        }
    }
}
```

## 热更新流程演示

### 打包

xasset出包流程为

1. Apply Rule
2. Build Rule
3. Build Bundle
4. Build Player

根据我们ET的传统艺能，资源形式大多都是一个个prefab（但是这种做法不提倡嗷，要按正规项目那样分布）

这里以Unit为例，对Unit文件夹应用Prefab规则（各个规则代表的含义可以去前面链接里的文章查看）

![image-20201016124813817](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20201016124813.png)

对于FUI（我们的FGUI编辑器导出的文件）需要应用两次规则，因为有png和bytes两种文件

然后我们会得到一个如下所示的Rule.asset文件

![image-20201016125030207](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20201016125030.png)

其中的Scene In Build选项中需要包含我们随包发布的Scene（ET中的Init.scene）

然后我们Build Bundle，就可以出包了

### 运行

为Global添加Update Mono脚本

![image-20201016094914219](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20201016094920.png)

其中各个内容含义为：

- Step：当前热更新阶段
- Update Progess：当前热更新阶段进度
- Development Mode：是否开启编辑器资源模式，如果开启会使用AssetDatabase.load进行资源加载原始资源，如果关闭会模拟出包环境下的资源加载
- Enable VFS：是否开启VFS（对于VFS更加详细的内容，可以去上文链接中查看）
- Base URL：资源下载地址，这里我填写的是HFS的资源地址，如果我们使用ET资源文件服务器就是http://127.0.0.1:8080/

然后在脚本调用，即可进行热更新，其中对于热更新各个阶段的进度，都可对Updater的Step和UpdateProgress来取得

```cs
await bundleDownloaderComponent.StartUpdate();
```

### 资源加载

#### 同步资源加载

以我们加载Hotfix.dll.bytes为例

```cs
GameObject code = Game.Scene.GetComponent<ResourcesComponent>().LoadAsset<GameObject>(ABPathUtilities.GetNormalConfigPath("Code"));

byte[] assBytes = code.GetTargetObjectFromRC<TextAsset>("Hotfix.dll").bytes;
byte[] pdbBytes = code.GetTargetObjectFromRC<TextAsset>("Hotfix.pdb").bytes;
```

#### 异步资源加载

这里加载一在路径**Assets/Textures/TargetTextureName.png**中的贴图示例

```cs
await Game.Scene.GetComponent<ResourcesComponent>().LoadAssetAsync<Sprite>("Assets/Textures/TargetTextureName.png");
```

### 资源卸载

```cs
Game.Scene.GetComponent<ResourcesComponent>().UnLoadAsset("Assets/Textures/TargetTextureName.png");
```

### 资源内存释放

xasset采用惰性GC的资源内存管理方案，老版本是每帧都会检查和清理未使用的资源（称为灵敏GC），这个版本底层只会在切换场景或者主动调用Assets.RemoveUnusedAssets();的时候才会清理未使用的资源，这样用户可以按需调整资源回收的频率，在没有内存压力的时候，不回收可以获得更好的性能。
