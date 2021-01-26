---
title: ET&&FGUI接入URP流程
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

### 前言

前几天技能系统发布了新版本，休息了以下，接下来要开始开发[Moba](https://gitee.com/NKG_admin/NKGMobaBasedOnET)项目的特效部分了，准备使用Shader Graph（简称SG）技术栈，但是SG只在SRP中才可以正常工作，所以要做一些适配。

### 环境

[DCET](https://github.com/DukeChiang/DCET)中的FGUI模块（Model层与Hotfix层）

ET 5.0

Unity 2019.4.8 LTS

URP 7.3.1

### 步骤

由于URP版本中Camera的特殊性(使用Render Type和CameraStack控制渲染)，需要先对相机模块进行更改

首先Unity.Model和ThirdParty的Assembly Definition需要索引Unity.RenderPipeline.Universal.Runtime

![image-20201005210256063](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20201005210256.png)

我们要确保整个项目只有一个Render Type为Base的Camera，否则渲染将不会如我们所愿的那样（指Build-in管线中相机渲染结果的自动叠加）

首先更改FGUI源码中的StageCamera.cs文件

```cs
        /// <summary>
        /// 
        /// </summary>
        /// <param name="name"></param>
        /// <param name="cullingMask"></param>
        /// <returns></returns>
        public static Camera CreateCamera(string name, int cullingMask)
        {
            GameObject cameraObject = new GameObject(name);
            Camera camera = cameraObject.AddComponent<Camera>();
            camera.depth = 1;
            camera.cullingMask = cullingMask;
            camera.clearFlags = CameraClearFlags.Depth;
            camera.orthographic = true;
            camera.orthographicSize = DefaultCameraSize;
            camera.nearClipPlane = -30;
            camera.farClipPlane = 30;
#if UNITY_5_4_OR_NEWER
            camera.stereoTargetEye = StereoTargetEyeMask.None;
#endif
#if UNITY_5_6_OR_NEWER
            camera.allowHDR = false;
            camera.allowMSAA = false;
#endif
            #region 适配URP
            UniversalAdditionalCameraData universalAdditionalCameraData = camera.GetUniversalAdditionalCameraData();
            universalAdditionalCameraData.renderType = CameraRenderType.Overlay;
            Camera.main.GetUniversalAdditionalCameraData().cameraStack.Add(camera);
            DontDestroyOnLoad(cameraObject);
            #endregion
            cameraObject.AddComponent<StageCamera>();
            return camera;
        }
```

创建MainCamera，放置在Init场景

![image-20201005205859826](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20201005205859.png)

使用Init脚本索引MainCamera

![image-20201005205924525](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/20201005205924.png)

然后在Init代码中设置

```cs
DontDestroyOnLoad(MainCamera);
```

即可正常运行

### 番外

如果发现项目中部分Shader失效，是因为在URP中已经不适配了，需要自己重写
