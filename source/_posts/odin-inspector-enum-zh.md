---
title: Odin在Inspector面板显示枚举中文名
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

Odin这个神器插件想必现在已经家喻户晓了，如果还不知道的建议重。。。重新去百度一下Odin插件。

简而言之使用Odin之后，能应付90%的编辑器拓展工作，并节省至少70%的体力，摸鱼力UPUP。

但是封装这么到位的插件也是有短肋的，他对于用户来说就是一个黑盒，想要拓展功能往往比较麻烦，有些极端情况甚至需要修改源代码（有UE4内味了，味大了），不过总体而言还是利大于弊的，大家酌情选择使用。

那么这篇文章就是带领大家解决一个比较常见的痛点，Odin无法在Inspector面板显示出枚举类型的中文名，比如

![image-20200804235455066](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200804235455066.png)

他就只能这样

![image-20200804235538862](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200804235538862.png)

虽然说那个下拉窗口里是中文，方便我们选择，但是在Inspector面板显示英文的话仍然不够直观，所以我们尽可能去实现这一个功能，来进一步提高生产力



**别来跟我提Unity原生的枚举中文显示解决方案嗷，太丑了点**



### 正文

首先要去扒拉Odin的源码，我这边已经扒拉好了

这就是绘制枚举类型的核心函数，第一个Label代表我们的字段名，第二个Label就是我们具体枚举的显示文字了

```C#
    /// <summary>Draws an enum selector field using the enum selector.</summary>
    public static T DrawEnumField(
      GUIContent label,
      GUIContent contentLabel,
      T value,
      GUIStyle style = null)
    {
      int controlId;
      Rect valueRect;
      SirenixEditorGUI.GetFeatureRichControlRect(label, out controlId, out bool _, out valueRect);
      Action<EnumSelector<T>> bindSelector;
      Func<IEnumerable<T>> resultGetter;
      if (OdinSelector<T>.DrawSelectorButton<EnumSelector<T>>(valueRect, contentLabel, style ?? EditorStyles.popup, controlId, out bindSelector, out resultGetter))
      {
        EnumSelector<T> enumSelector = new EnumSelector<T>();
        if (!EditorGUI.showMixedValue)
          enumSelector.SetSelection(value);
        OdinEditorWindow odinEditorWindow = enumSelector.ShowInPopup(new Vector2(valueRect.xMin, valueRect.yMax));
        if (EnumSelector<T>.isFlagEnum)
          odinEditorWindow.OnClose += new Action(enumSelector.SelectionTree.Selection.ConfirmSelection);
        bindSelector(enumSelector);
      }
      if (resultGetter != null)
        value = resultGetter().FirstOrDefault<T>();
      return value;
    }
```

目标函数找到了，剩下的就很简单了，Odin提供给我们自定义绘制字段值的功能，然后我们再只需要用一下反射即可

还是上面那个例子

```C#
    [CustomValueDrawer("TestDrawEnumByChinese")]
    [LabelText("测试中文枚举")]
    public enum TestEnum
    {
        [LabelText("测试1")]
        Test1,

        [LabelText("测试2")]
        Test2
    }
```



```C#
        private static TestEnum TestDrawEnumByChinese(TestEnum value, GUIContent label)
        {
            Type targetEnumType = typeof (TestEnum);
            LabelTextAttribute targetEnumLabelTextAttribute = targetEnumType.GetAttribute<LabelTextAttribute>();

            FieldInfo[] fieldInfos = targetEnumType.GetFields();

            string[] targetItemNames = Enum.GetNames(targetEnumType);

            string finalShowName = "";
            for (int i = 0; i < targetItemNames.Length; i++)
            {
                if (targetItemNames[i] == value.ToString())
                {
                    finalShowName = value.ToString();
                    break;
                }
            }

            for (int i = 0; i < fieldInfos.Length; i++)
            {
                if (fieldInfos[i].Name == finalShowName)
                {
                    finalShowName = fieldInfos[i].GetAttribute<LabelTextAttribute>().Text;
                    break;
                }
            }

            return EnumSelector<TestEnum>.DrawEnumField(new GUIContent(targetEnumLabelTextAttribute.Text), new GUIContent(finalShowName), value);
        }
```

然后我们随便找个Mono脚本声明一个TestEnum字段即可

然后就可以在Inspector面板看到结果啦

![image-20200805000332567](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/typoraImages/image-20200805000332567.png)

### 优化及注意点

当然了，由于我们用到了反射，并且Odin会频繁调用这个自定义的绘制函数，所以我们肯定还是要做一下优化的，主要有以下几点

- 单独做一个静态类用来定义项目中所有需要中文显示枚举名的自定义绘制函数（都差不太多），方便管理
- 适配标注System.Flag属性的枚举
- Cache住反射信息

