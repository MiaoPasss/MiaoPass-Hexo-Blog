---
title: Sprite Atlas 图集
date: 2026-03-25 17:02:10
categories: 
- Game Engine
- Unity
tags:
- Resource management
---

# Sprite Atlas 图集
把多张小纹理合并成一张大纹理，GPU 渲染时所有引用该图集的 Sprite 共享同一张纹理，满足合批条件，从而将多个 DrawCall 合并为一个。
```
未使用图集：
  Sprite A（texture_a.png）→ DrawCall 1
  Sprite B（texture_b.png）→ DrawCall 2
  Sprite C（texture_c.png）→ DrawCall 3

使用图集后：
  Sprite A ─┐
  Sprite B ─┼─ atlas.png → DrawCall 1（合批）
  Sprite C ─┘
```

除了减少 DrawCall，图集还有两个底层优化：
* GPU 处理 2 的幂次方尺寸的纹理效率更高，图集会自动将小图打包成符合规范的大图
* 多张小图的磁盘占用通常大于一张合并后的大图（压缩算法在大块连续数据上效率更高）

Unity中的图集有2种类型
* Master（主图集）：标准图集，包含实际资源
* Variant（变体图集）：引用主图集内容，通过 Scale 缩放生成不同分辨率版本，适合高清/低清资源切换场景

# 代码用例

## 运行时加载图集（Resources）
```cs
using UnityEngine;
using UnityEngine.UI;
using UnityEngine.U2D;

public class AtlasLoader : MonoBehaviour
{
    [SerializeField] private Image[] _images;

    void Start()
    {
        // 从 Resources 文件夹加载图集
        SpriteAtlas atlas = Resources.Load<SpriteAtlas>("UI/MainAtlas");
        if (atlas == null)
        {
            Debug.LogError("图集加载失败，检查路径是否正确");
            return;
        }

        // GetSprite 传入的是打包时的文件名（不含扩展名）
        // Single 模式：直接传文件名，如 "icon_sword"
        // Multiple 模式（Sprite Sheet）：传 "文件名_索引"，如 "icon_sword_0"
        _images[0].sprite = atlas.GetSprite("icon_sword");
        _images[1].sprite = atlas.GetSprite("icon_shield");
        _images[2].sprite = atlas.GetSprite("icon_potion");
    }
}
```

## 从 AssetBundle 加载图集
```cs
using UnityEngine;
using UnityEngine.UI;
using UnityEngine.U2D;

public class AtlasFromAB : MonoBehaviour
{
    [SerializeField] private Image _targetImage;

    void Start()
    {
        string bundlePath = Application.streamingAssetsPath + "/AssetBundles/ui_atlas";
        AssetBundle bundle = AssetBundle.LoadFromFile(bundlePath);

        if (bundle == null)
        {
            Debug.LogError("AB 包加载失败");
            return;
        }

        SpriteAtlas atlas = bundle.LoadAsset<SpriteAtlas>("MainAtlas");
        _targetImage.sprite = atlas.GetSprite("icon_sword");

        // 图集资源已提取，可以卸载包体（false = 保留已加载资源）
        bundle.Unload(false);
    }
}
```

## 后期绑定（Late Binding）
当图集未勾选 Include in Build 时，必须手动监听 atlasRequested 事件，否则引用该图集的 Sprite 会显示为空白。
```cs
using UnityEngine;
using UnityEngine.U2D;
using System;

// 挂载到场景中持久存在的对象上（如 GameManager）
public class AtlasLateBinding : MonoBehaviour
{
    void OnEnable()
    {
        // 注册图集请求事件
        SpriteAtlasManager.atlasRequested += OnAtlasRequested;
        // 可选：图集注册完成后的回调
        SpriteAtlasManager.atlasRegistered += OnAtlasRegistered;
    }

    void OnDisable()
    {
        SpriteAtlasManager.atlasRequested -= OnAtlasRequested;
        SpriteAtlasManager.atlasRegistered -= OnAtlasRegistered;
    }

    // Unity 找不到图集时触发，atlasName 是图集名称
    private void OnAtlasRequested(string atlasName, Action<SpriteAtlas> callback)
    {
        Debug.Log($"请求图集: {atlasName}");

        // 从 Resources 加载（也可以换成 AB 包加载）
        SpriteAtlas atlas = Resources.Load<SpriteAtlas>(atlasName);

        if (atlas != null)
            callback(atlas); // 把图集交还给 Unity
        else
            Debug.LogError($"找不到图集: {atlasName}");
    }

    private void OnAtlasRegistered(SpriteAtlas atlas)
    {
        Debug.Log($"图集已注册: {atlas.name}");
    }
}
```

## Editor 工具：自动打包图集
这个工具会遍历指定目录，按文件夹自动创建并更新图集，适合团队统一管理图集资源。
```cs
// 必须放在 Editor 文件夹下
using System;
using System.Collections.Generic;
using System.IO;
using UnityEditor;
using UnityEditor.U2D;
using UnityEngine;
using UnityEngine.U2D;
using Object = UnityEngine.Object;

public class SpriteAtlasBuilder
{
    // 需要打包的图片根目录（每个子文件夹对应一个图集）
    private static readonly string SourceRoot = Application.dataPath + "/Art/Sprites/";
    // 图集输出目录
    private static readonly string OutputPath = "Assets/Art/Atlases/";

    [MenuItem("Tools/Build Sprite Atlases")]
    public static void BuildAll()
    {
        if (!Directory.Exists(SourceRoot))
        {
            Debug.LogError($"源目录不存在: {SourceRoot}");
            return;
        }

        Directory.CreateDirectory(OutputPath.Replace("Assets/", Application.dataPath + "/").Replace("Assets", ""));

        var dirs = new DirectoryInfo(SourceRoot).GetDirectories();
        for (int i = 0; i < dirs.Length; i++)
        {
            var dir = dirs[i];
            EditorUtility.DisplayProgressBar("打包图集", $"处理: {dir.Name}", (float)i / dirs.Length);

            string atlasPath = OutputPath + dir.Name + ".spriteatlas";
            SpriteAtlas atlas = AssetDatabase.LoadAssetAtPath<SpriteAtlas>(atlasPath)
                                ?? CreateAtlas(atlasPath);

            UpdateAtlasContent(atlas, SourceRoot + dir.Name);
        }

        EditorUtility.ClearProgressBar();
        AssetDatabase.SaveAssets();
        AssetDatabase.Refresh();
        Debug.Log("图集打包完成");
    }

    private static SpriteAtlas CreateAtlas(string savePath)
    {
        var atlas = new SpriteAtlas();

        // 基础打包设置
        atlas.SetPackingSettings(new SpriteAtlasPackingSettings
        {
            blockOffset = 1,
            enableRotation = false,    // 禁止旋转，避免 UI 显示异常
            enableTightPacking = false, // 禁止紧密排列，避免图片边缘互相干扰
            padding = 4,
        });

        // 纹理设置
        atlas.SetTextureSettings(new SpriteAtlasTextureSettings
        {
            readable = false,
            generateMipMaps = false,
            sRGB = true,
            filterMode = FilterMode.Bilinear,
        });

        // Android 平台压缩格式
        var androidSettings = atlas.GetPlatformSettings("Android");
        androidSettings.overridden = true;
        androidSettings.maxTextureSize = 2048;
        androidSettings.format = TextureImporterFormat.ASTC_6x6;
        atlas.SetPlatformSettings(androidSettings);

        // iOS 平台压缩格式
        var iosSettings = atlas.GetPlatformSettings("iPhone");
        iosSettings.overridden = true;
        iosSettings.maxTextureSize = 2048;
        iosSettings.format = TextureImporterFormat.PVRTC_RGB4;
        atlas.SetPlatformSettings(iosSettings);

        AssetDatabase.CreateAsset(atlas, savePath);
        return atlas;
    }

    private static void UpdateAtlasContent(SpriteAtlas atlas, string folderPath)
    {
        var existing = new HashSet<Object>(atlas.GetPackables());
        var toAdd = new List<Object>();

        foreach (var file in Directory.GetFiles(folderPath, "*", SearchOption.AllDirectories))
        {
            if (!file.EndsWith(".png") && !file.EndsWith(".jpg")) continue;

            // 转换为相对于 Assets 的路径
            string relativePath = "Assets" + file.Replace(Application.dataPath, "").Replace("\\", "/");
            var obj = AssetDatabase.LoadAssetAtPath<Object>(relativePath);

            if (obj != null && !existing.Contains(obj))
                toAdd.Add(obj);
        }

        if (toAdd.Count > 0)
            atlas.Add(toAdd.ToArray());
    }
}
```

## 运行时动态检查 DrawCall（调试用）
```cs
#if UNITY_EDITOR
using UnityEngine;
using UnityEngine.Profiling;

// 挂到场景中，运行时在 Console 输出当前帧的渲染统计
public class DrawCallMonitor : MonoBehaviour
{
    void Update()
    {
        // 需要在 Editor 的 Stats 面板查看，或通过 Profiler API
        // UnityStats 仅在编辑器下可用
        if (Time.frameCount % 60 == 0) // 每 60 帧输出一次
        {
            Debug.Log($"[渲染统计] 帧: {Time.frameCount} | " +
                      $"已分配内存: {Profiler.GetTotalAllocatedMemoryLong() / 1024 / 1024} MB");
        }
    }
}
#endif
```

# 注意事项与最佳实践

## 内存陷阱
图集中只要有一个 Sprite 被引用，整张图集纹理都会加载进内存。所以图集的划分策略很关键：
* 按功能模块划分（登录界面、大厅界面、战斗界面各一个图集）
* 避免把低频资源和高频资源放在同一图集
* 单个图集不要超过 2048×2048，超出部分 Unity 会自动忽略 MaxTextureSize 设置

## 合批失效的常见原因：
* 两个 Sprite 来自不同图集
* 中间插入了使用不同材质的对象（打断批次）
* 开启了 Tight Packing 导致 UV 计算异常

# Sprite Atlas 和 Drawcall
Atlas 本身不减少 Draw Call，它是让 batching 成为可能的前提条件。

## Sprite Atlas 
图集解决的是纹理问题，把多张小图合并成一张大图。
原来每个 sprite 用不同纹理 → 不同材质 → 每次渲染都要 SetPass Call 切换。合并后所有 sprite 共享同一张纹理 → 同一个材质 → 为后续 batching 创造条件。

```
没有图集：
  图标A (tex_a) → SetPass → Draw
  图标B (tex_b) → SetPass → Draw   ← 纹理不同，必须切换
  图标C (tex_c) → SetPass → Draw

有图集：
  图标A (atlas) → SetPass → Draw
  图标B (atlas) →           Draw   ← 同一纹理，可以 batch
  图标C (atlas) →           Draw
```

## Batching
Static Batching / GPU Instancing 能合并 Draw Call，但不能合并 SetPass Call 。
```
// GPU Instancing：同一材质的100个物体(DrawCall x 100)
SetPass Call × 1   ← 只切换一次状态
Draw Call × 1      ← 合并成一次绘制（instanced）
```
而 Dynamic Batching 同时减少两者，因为它把多个 mesh 合并成一个，用同一个材质一次画完。

* Draw Call 多 → GPU 压力大
* SetPass Call 多 → CPU 压力大（状态切换开销）
* 先降 SetPass Call（合并材质、用图集），再降 Draw Call（Batching、Instancing）

Unity Stats 窗口里显示的 "SetPass Calls" 和 "Batches" 分别对应这两个指标，看这两个数比只看 Draw Call 更有参考价值。

## Sprite Atlas 如何减少 Draw Call
Sprite Atlas 确实减少 Draw Call，但不是靠自己，而是靠触发 Dynamic Batching。

Draw Call 能被合并的条件是：多个物体使用完全相同的材质（同 shader + 同纹理 + 同参数）。

没有 Atlas 时：
```
图标A → 材质A (tex_a)  ← 不同纹理 = 不同材质
图标B → 材质B (tex_b)  ← 无法 batch
图标C → 材质C (tex_c)  ← 3个 Draw Call
```

有了 Atlas，所有图标共享同一张纹理，材质变成一样的，Dynamic Batching 才能介入把它们合并：
```
图标A → 材质X (atlas)  ← 同一材质
图标B → 材质X (atlas)  ← Dynamic Batching 合并
图标C → 材质X (atlas)  ← 1个 Draw Call
```

所以准确说法是：Atlas 统一了纹理，使 batching 成为可能，batching 才是真正减少 Draw Call 的那一步。 两者缺一不可。

即使不考虑 Draw Call，Atlas 本身也有价值：
* 减少纹理切换的 SetPass Call，这个开销比 Draw Call 更重
* 减少内存碎片，GPU 显存里一张大图比十张小图更高效
* 减少文件 IO，加载一张图比加载十张快

# 总结
Atlas 的核心价值是"统一材质"，Draw Call 的减少是这个统一带来的副产品，真正执行合并的是 Dynamic Batching。