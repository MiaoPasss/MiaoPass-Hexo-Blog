---
title: AssetBundle 与 Addressables
date: 2026-03-24 16:58:01
categories: 
- Game Engine
- Unity
tags:
- Resource Management
---

# AssetBundle
AssetBundle 本质是一个压缩的二进制文件，内部结构如下：
```
AssetBundle 文件
├── Header（包头：版本、压缩类型、元数据）
├── Manifest（资源清单：资源名、依赖关系、GUID）
└── Data Segment（序列化的资源数据）
```
支持两种压缩格式：
* LZMA：压缩率高，但加载时需整包解压，内存占用大
* LZ4：块压缩，按需解压，加载更快，推荐使用

## 依赖关系
AB 包之间存在依赖链，必须按顺序加载：
```
bundle_character.ab
    └── 依赖 bundle_texture.ab
            └── 依赖 bundle_shader.ab
```
如果 bundle_texture.ab 未加载，bundle_character.ab 中的资源会出现材质丢失（粉红色）。

## 打包脚本（Editor）
```cs
// Editor/AssetBundleBuilder.cs
using UnityEditor;
using System.IO;

public class AssetBundleBuilder
{
    [MenuItem("Tools/Build AssetBundles")]
    public static void BuildAllAB()
    {
        string outputPath = "Assets/StreamingAssets/AssetBundles";
        if (!Directory.Exists(outputPath))
            Directory.CreateDirectory(outputPath);

        // BuildAssetBundleOptions.ChunkBasedCompression 使用 LZ4 压缩
        BuildPipeline.BuildAssetBundles(
            outputPath,
            BuildAssetBundleOptions.ChunkBasedCompression,
            BuildTarget.StandaloneWindows64
        );

        AssetDatabase.Refresh();
        UnityEngine.Debug.Log("AssetBundle 构建完成");
    }
}
```

## 运行时加载管理器
```cs Runtime/AssetBundleManager.cs
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class AssetBundleManager : MonoBehaviour
{
    // 缓存已加载的 AB 包，避免重复加载
    private Dictionary<string, AssetBundle> _loadedBundles = new();
    
    private string _basePath;

    void Awake()
    {
        _basePath = Application.streamingAssetsPath + "/AssetBundles/";
    }

    // 同步加载 AB 包（会阻塞主线程，小包可用）
    public AssetBundle LoadBundle(string bundleName)
    {
        if (_loadedBundles.TryGetValue(bundleName, out var cached))
            return cached;

        // 先加载依赖
        LoadDependencies(bundleName);

        var bundle = AssetBundle.LoadFromFile(_basePath + bundleName);
        if (bundle == null)
        {
            Debug.LogError($"加载 AB 包失败: {bundleName}");
            return null;
        }

        _loadedBundles[bundleName] = bundle;
        return bundle;
    }

    // 加载依赖包（通过 Manifest 自动解析）
    private void LoadDependencies(string bundleName)
    {
        // Manifest 包名与输出目录同名
        var manifestBundle = AssetBundle.LoadFromFile(_basePath + "AssetBundles");
        if (manifestBundle == null) return;

        var manifest = manifestBundle.LoadAsset<AssetBundleManifest>("AssetBundleManifest");
        string[] deps = manifest.GetAllDependencies(bundleName);

        foreach (var dep in deps)
        {
            if (!_loadedBundles.ContainsKey(dep))
            {
                var depBundle = AssetBundle.LoadFromFile(_basePath + dep);
                if (depBundle != null)
                    _loadedBundles[dep] = depBundle;
            }
        }

        manifestBundle.Unload(false); // 卸载 Manifest 包，保留已加载资源
    }

    // 异步加载 AB 包（推荐，不阻塞主线程）
    public IEnumerator LoadBundleAsync(string bundleName, System.Action<AssetBundle> onComplete)
    {
        if (_loadedBundles.TryGetValue(bundleName, out var cached))
        {
            onComplete?.Invoke(cached);
            yield break;
        }

        var request = AssetBundle.LoadFromFileAsync(_basePath + bundleName);
        yield return request;

        if (request.assetBundle == null)
        {
            Debug.LogError($"异步加载 AB 包失败: {bundleName}");
            yield break;
        }

        _loadedBundles[bundleName] = request.assetBundle;
        onComplete?.Invoke(request.assetBundle);
    }

    // 从已加载的包中加载具体资源
    public T LoadAsset<T>(string bundleName, string assetName) where T : Object
    {
        var bundle = LoadBundle(bundleName);
        return bundle?.LoadAsset<T>(assetName);
    }

    // 卸载 AB 包
    // unloadAllObjects=true：同时卸载从该包加载的所有资源实例（防内存泄漏）
    // unloadAllObjects=false：只卸载包本身，已加载的资源继续存在
    public void UnloadBundle(string bundleName, bool unloadAllObjects = false)
    {
        if (_loadedBundles.TryGetValue(bundleName, out var bundle))
        {
            bundle.Unload(unloadAllObjects);
            _loadedBundles.Remove(bundleName);
        }
    }

    // 卸载所有包
    public void UnloadAll(bool unloadAllObjects = false)
    {
        foreach (var bundle in _loadedBundles.Values)
            bundle.Unload(unloadAllObjects);
        _loadedBundles.Clear();
    }
}
```

## 使用示例
```cs
// 使用 AssetBundleManager 加载预制体并实例化
public class GameLoader : MonoBehaviour
{
    [SerializeField] private AssetBundleManager _abManager;

    IEnumerator Start()
    {
        // 异步加载包
        AssetBundle bundle = null;
        yield return _abManager.LoadBundleAsync("characters", b => bundle = b);

        if (bundle != null)
        {
            // 从包中加载预制体
            var prefab = bundle.LoadAsset<GameObject>("Hero");
            Instantiate(prefab, Vector3.zero, Quaternion.identity);

            // 资源实例化后可以卸载包（unloadAllObjects=false 保留实例）
            _abManager.UnloadBundle("characters", false);
        }
    }
}
```
## 关于 catalog.json

AB 是实际的数据包，可以包含：
* Prefab
* Texture、Sprite
* Scene
* Audio
* ScriptableObject
* 配置表
* 热更新 DLL 的 .bytes
* AOT 元数据 DLL 的 .bytes
* AB 是平台相关的，Android、iOS、Windows 通常需要分别构建。

*catalog.json* 不保存资源内容，它保存“资源怎样定位”的信息，例如：
```
地址/Label/GUID
  → 使用哪个 Provider
  → 对应哪个 AssetBundle
  → Bundle 的远端 URL
  → Bundle Hash、CRC、大小
  → 依赖哪些其他 Bundle
  → 最终加载的资源类型
```

所以运行时链路是：
```
Addressables 地址
      ↓ 查询
catalog.json
      ↓ 找到 Bundle 地址和依赖
下载/读取 AssetBundle
      ↓
从 AB 中取出 TextAsset、Prefab 等实际资源
```

启用 Remote Catalog 后，启动流程通常是：
1. 加载 Player 内置的 Addressables 设置
2. 请求远端 catalog.hash
3. 与本地缓存 Hash 比较
4. Hash 不同则下载新的 catalog.json
5. 新 Catalog 指向新的或旧的 AB
6. 按需下载变化的 AB

带内容 Hash 的 AB 应当视为不可变文件，旧 AB 不要立即删除，因为新 Catalog仍可能引用未变化的旧 Bundle。

---

# Addressables
Addressables 是对 AssetBundle 的高层封装，核心组件：
```
Addressables 系统
├── Catalog（资源目录）
│     └── 地址(string) → 资源位置(本地/远程) 的映射表
├── ResourceLocator（资源定位器）
│     └── 根据 Catalog 找到资源的实际路径
├── ResourceProvider（资源提供者）
│     └── 负责实际的加载逻辑（本地文件/网络下载）
├── AsyncOperationHandle（异步操作句柄）
│     └── 管理加载状态、进度、回调
└── ResourceManager（资源管理器）
      └── 引用计数、内存管理
```

## 加载流程
```
Addressables.LoadAssetAsync("Hero")
    ↓
ResourceLocator 查询 Catalog
    ↓
找到资源位置（本地 StreamingAssets 或远程 CDN URL）
    ↓
ResourceProvider 加载对应的 AssetBundle
    ↓
自动加载所有依赖 Bundle
    ↓
从 Bundle 中提取目标资源
    ↓
引用计数 +1，返回 AsyncOperationHandle
```

## 引用计数机制
Addressables 内部维护引用计数，每次 LoadAssetAsync 计数 +1，每次 Release 计数 -1，归零时自动卸载 Bundle，这是它比原生 AB 包更安全的核心原因。

## 基础加载与释放
```cs
// Runtime/AddressablesLoader.cs
using System.Collections;
using UnityEngine;
using UnityEngine.AddressableAssets;
using UnityEngine.ResourceManagement.AsyncOperations;

public class AddressablesLoader : MonoBehaviour
{
    // 在 Inspector 中直接引用 Addressable 资源（推荐，类型安全）
    [SerializeField] private AssetReferenceGameObject _heroPrefabRef;

    private AsyncOperationHandle<GameObject> _loadHandle;

    void Start()
    {
        // 方式一：通过 AssetReference（Inspector 拖拽绑定）
        LoadViaAssetReference();

        // 方式二：通过地址字符串
        // LoadViaAddress("Hero");
    }

    // 通过 AssetReference 加载（推荐）
    void LoadViaAssetReference()
    {
        _loadHandle = _heroPrefabRef.LoadAssetAsync<GameObject>();
        _loadHandle.Completed += OnLoadComplete;
    }

    // 通过地址字符串加载
    void LoadViaAddress(string address)
    {
        _loadHandle = Addressables.LoadAssetAsync<GameObject>(address);
        _loadHandle.Completed += OnLoadComplete;
    }

    private void OnLoadComplete(AsyncOperationHandle<GameObject> handle)
    {
        if (handle.Status == AsyncOperationStatus.Succeeded)
        {
            Instantiate(handle.Result, Vector3.zero, Quaternion.identity);
            Debug.Log("资源加载成功");
        }
        else
        {
            Debug.LogError($"资源加载失败: {handle.OperationException}");
        }
    }

    void OnDestroy()
    {
        // 必须释放句柄，否则引用计数不归零，内存泄漏
        if (_loadHandle.IsValid())
            Addressables.Release(_loadHandle);
    }
}
```

## 异步实例化（更高效，跳过 LoadAsset 步骤）
```cs
public class AddressablesInstantiator : MonoBehaviour
{
    [SerializeField] private string _address;
    private AsyncOperationHandle<GameObject> _instantiateHandle;

    IEnumerator Start()
    {
        // InstantiateAsync 直接加载并实例化，一步到位
        _instantiateHandle = Addressables.InstantiateAsync(
            _address,
            Vector3.zero,
            Quaternion.identity
        );

        // 等待完成并显示进度
        while (!_instantiateHandle.IsDone)
        {
            Debug.Log($"加载进度: {_instantiateHandle.PercentComplete:P0}");
            yield return null;
        }

        if (_instantiateHandle.Status == AsyncOperationStatus.Succeeded)
            Debug.Log($"实例化成功: {_instantiateHandle.Result.name}");
    }

    void OnDestroy()
    {
        // InstantiateAsync 创建的对象用 ReleaseInstance 释放
        if (_instantiateHandle.IsValid())
            Addressables.ReleaseInstance(_instantiateHandle);
    }
}
```

## 批量加载（LoadAssetsAsync）
```cs
public class AddressablesBatchLoader : MonoBehaviour
{
    // 通过 Label 批量加载同类资源
    [SerializeField] private string _label = "Enemies";

    private AsyncOperationHandle<IList<GameObject>> _batchHandle;

    IEnumerator Start()
    {
        _batchHandle = Addressables.LoadAssetsAsync<GameObject>(
            _label,
            obj => Debug.Log($"单个资源加载完成: {obj.name}") // 每个资源加载完的回调
        );

        yield return _batchHandle;

        if (_batchHandle.Status == AsyncOperationStatus.Succeeded)
        {
            foreach (var prefab in _batchHandle.Result)
                Instantiate(prefab);
        }
    }

    void OnDestroy()
    {
        if (_batchHandle.IsValid())
            Addressables.Release(_batchHandle);
    }
}
```

## 场景加载
```cs
using UnityEngine.ResourceManagement.ResourceProviders;
using UnityEngine.SceneManagement;

public class AddressablesSceneLoader : MonoBehaviour
{
    [SerializeField] private string _sceneAddress = "Level_01";
    private AsyncOperationHandle<SceneInstance> _sceneHandle;

    public IEnumerator LoadScene()
    {
        _sceneHandle = Addressables.LoadSceneAsync(
            _sceneAddress,
            LoadSceneMode.Additive, // 叠加模式，不卸载当前场景
            activateOnLoad: true
        );

        while (!_sceneHandle.IsDone)
        {
            Debug.Log($"场景加载进度: {_sceneHandle.PercentComplete:P0}");
            yield return null;
        }

        Debug.Log("场景加载完成");
    }

    public IEnumerator UnloadScene()
    {
        if (_sceneHandle.IsValid())
        {
            var unloadHandle = Addressables.UnloadSceneAsync(_sceneHandle);
            yield return unloadHandle;
        }
    }
}
```

## 热更新检查（远程内容更新）
```cs
using System.Collections.Generic;
using UnityEngine.AddressableAssets.ResourceLocators;

public class AddressablesUpdater : MonoBehaviour
{
    IEnumerator Start()
    {
        // Step 1：检查 Catalog 是否有更新
        var checkHandle = Addressables.CheckForCatalogUpdates(autoReleaseHandle: false);
        yield return checkHandle;

        if (checkHandle.Status != AsyncOperationStatus.Succeeded)
        {
            Addressables.Release(checkHandle);
            yield break;
        }

        List<string> catalogsToUpdate = checkHandle.Result;
        Addressables.Release(checkHandle);

        if (catalogsToUpdate.Count == 0)
        {
            Debug.Log("无需更新");
            yield break;
        }

        // Step 2：更新 Catalog
        var updateHandle = Addressables.UpdateCatalogs(catalogsToUpdate, autoReleaseHandle: false);
        yield return updateHandle;

        List<IResourceLocator> updatedLocators = updateHandle.Result;
        Addressables.Release(updateHandle);

        // Step 3：下载更新的内容
        foreach (var locator in updatedLocators)
        {
            var sizeHandle = Addressables.GetDownloadSizeAsync(locator.Keys);
            yield return sizeHandle;

            long downloadSize = sizeHandle.Result;
            Addressables.Release(sizeHandle);

            if (downloadSize > 0)
            {
                Debug.Log($"需要下载: {downloadSize / 1024f / 1024f:F2} MB");

                var downloadHandle = Addressables.DownloadDependenciesAsync(locator.Keys);
                yield return downloadHandle;
                Addressables.Release(downloadHandle);
            }
        }

        Debug.Log("更新完成");
    }
}
```

## Create New Build
这里指 Addressables 的：
```
Build
  → New Build
  → Default Build Script
```

它构建的是 Addressables 内容，不等于构建 APK、AAB、IPA。New Build 的含义是“建立一个新的内容基线”：
* 根据当前 Group 和 Schema 重新生成 Catalog。
* 构建当前资源对应的 AssetBundle。
* 生成新的 addressables_content_state.bin。
* 不把某个已经发布的旧版本作为内容兼容约束。
* 可能利用本地构建缓存，但逻辑上仍然是新基线。

适合以下情况：
* 第一次发布。
* 发布新的 Player 大版本。
* 修改了 AOT 代码或原生插件。
* 升级 Unity、HybridCLR 或 Addressables。
* 修改 IL2CPP、裁剪、桥接代码。
* 大幅改变 Group、Bundle 布局、Build/Load Path。
* 原内容状态文件丢失或不可信。
* 更换平台。

每个正式 Player 基线都应该保存：
```
Android / Player 1.0 / addressables_content_state.bin
iOS     / Player 1.0 / addressables_content_state.bin
```

## Update a Previous Build
Update Build 获取已发布内容对应的addressables_content_state.bin，把它与当前资源状态比较，然后：
* 未变化的 Bundle 保持原名称和引用。
* 变化的 Bundle生成新版本。
* 生成新 Catalog。
* 新 Catalog 可以同时引用旧 Bundle 和新 Bundle。
* 它不会生成二进制差分补丁。
* Bundle 内只改了一个文件，也可能需要重新下载整个 Bundle。

例如初始版本：
```
catalog v1:
HotCode → hotcode_aaa.bundle
Lobby   → lobby_111.bundle
```
只修改热更新 DLL 后：
```
catalog v2:
HotCode → hotcode_bbb.bundle   新 Bundle
Lobby   → lobby_111.bundle     继续引用旧 Bundle
```
客户端只需要下载新 Catalog 和 hotcode_bbb.bundle。

Update Build 适合：
* 修改 HybridCLR 热更新 DLL。
* 修改远端 Prefab、图片、配置表。
* 修改活动和业务逻辑。
* 不需要修改 Player 原生/AOT 部分的内容更新。