---
title: Unity资源加载
date: 2026-03-20 16:49:19
categories: 
- Game Engine
- Unity
tags:
- Resource management
---

# Resources

Resources文件夹是一个只读的文件夹，通过Resources.Load()来读取对象。
因为这个文件夹下的所有资源都可以运行时来加载，所以Resources文件夹下的所有东西都会被无条件的打到发布包中。
建议这个文件夹下只放Prefab文件，因为Prefab会自动过滤掉对象上不需要的资源。

* 举个例子我把模型文件还有贴图文件都放在了Resources文件夹下，但是我有两张贴图是没有在模型上用的，那么此时这两张没用的贴图也会被打包到发布包中。
* 假如这里我用Prefab，那么Prefab会自动过滤到这两张不被用的贴图，这样发布包就会小一些了。
* 此时，若prefab引用的贴图在resources文件夹下，则贴图仍然会被打进包中，因为resources文件夹对资源无条件打包。
* 若prefab引用的贴图在AB包中，则实例化prefab之前，必须手动去把AB加载出来。
* 就算prefab不是在resources文件来而是在不同于它引用的贴图的AB包中，也一样要在实例化前把贴图所在的AB加载出来；但不用把贴图加载出来，prefab会自动去加载它。

## Resources.Load
Unity 打包时为每个资源分配 UID，同时建立一张"路径→UID"映射表存在配置文件里。
加载时先查表把路径转成 UID，再用 UID 找到实际数据。

```
Assets/Resources/ui/icon.png
         ↓ 打包
resources.assets（包含所有未被场景引用的Resources资源）
         ↓ 运行时
Resources.Load("ui/icon", typeof(Sprite))
```
特殊情况：如果 Resources 下的资源被场景直接引用，会打进 sharedassets\[level\].assets 而不是 resources.assets，不同目录下的 Resources 文件夹资源路径最好不重名。

卸载资源时用UnloadUnusedAssets（注意：UnloadAsset 对 Resources 无效）
```
// 加载
Texture2D tex = Resources.Load("ui/icon", typeof(Texture2D)) as Texture2D;

// 卸载
Resources.UnloadUnusedAssets(); // 正确方式
// Resources.UnloadAsset(tex); 只对 AB 包加载的资源有效

// 查找场景中所有同类型对象（包括未激活的）
Resources.FindObjectsOfTypeAll<GameObject>();
```

优点：
* 使用极简，一行代码加载
* 不需要管理 AB 包依赖

缺点：
* 所有 Resources 资源无论用没用都打进安装包，包体膨胀
* 无法热更，改资源必须重新发版
* 大量资源时启动慢（Unity 启动时要建立完整索引表）

# StreamingAssets
StreamingAssets文件夹也是一个只读的文件夹，其目录下的文件在打包时完全不处理，原样复制到目标平台，保留原始文件格式和路径结构。
* Resources文件夹下的资源会进行一次压缩，而且也会加密，不使用点特殊办法是拿不到原始资源的
* StreamingAssets文件夹下面的所有资源不会被加密，然后是原封不动的打包到发布包中，这样很容易就拿到里面的文件。所以StreamingAssets适合放一些二进制文件，而Resources更适合放一些GameObject和Object文件。
    * 比如 icon.png 是源文件，放进 Resources 后打包时变成 Texture2D Object，使用Resources.Load\<Texture2D\>("icon")可直接返回 Texture2D 对象
    * 放进 StreamingAssets 它还是原始 .png，Unity 不处理它。

## 各平台存储位置
```
Windows/Editor  → [项目]/Assets/StreamingAssets/
iOS             → [.ipa包]/Data/Raw/
Android         → [.apk包]/assets/  （压缩存储在zip内）
WebGL           → 无法直接访问
```

Android 特殊处理（存在 APK 压缩包内，不能直接用文件路径）：
```
// Android 必须用 UnityWebRequest
IEnumerator LoadVideoAndroid(string fileName)
{
    string path = Path.Combine(Application.streamingAssetsPath, fileName);
    using var req = UnityWebRequest.Get(path);
    yield return req.SendWebRequest();
    byte[] data = req.downloadHandler.data;
}

// 其他平台可以直接用文件路径
string videoPath = Path.Combine(Application.streamingAssetsPath, "intro.mp4");
videoPlayer.url = videoPath;
```

优点：
* 保留原始文件格式，适合需要文件路径的第三方库
* 视频播放器、SQLite 数据库等可以直接访问
* 不经过 Unity 资源管道，格式不受限

缺点：
* Android 上访问慢且必须用 UnityWebRequest
* 无法热更
* 不适合大量小文件（没有打包优化）

实际用例：
* 视频文件（需要 VideoPlayer 直接访问路径）
* SQLite 数据库初始文件（首次运行复制到 PersistentDataPath 再读写）
* 第三方 SDK 需要的配置文件
* AB 包的初始版本（首次运行写入缓存）

## PersistentDataPath
由于StreamingAssets文件夹里的内容只能读不能写，所以在需要进行写入操作的时候可以使用PersistentDataPath文件夹，此目录下的文件可读可写，但和StreamingAssets的不同点是：在Editor阶段没有，等到手机安装App后在目标设备上自动生成
* 可以将一些需要读写的文件先放在Streaming Assets，在App安装后通过代码Copy到PersistentDataPath文件夹，比如上面提到的数据库初始文件

# AssetBundle
资源在 Inspector 面板右下角设置包名和变体名，相同包名打进同一个 .ab 文件。
```
包名: characters/hero    变体名: hd
包名: characters/hero    变体名: sd
→ 生成 characters/hero.hd 和 characters/hero.sd 两个变体包
```

变体包（Variant）是同一资源的不同版本，用于针对不同设备/平台加载不同质量的内容，但代码层面引用名称保持一致。

```cs 根据设备性能决定加载哪个变体
string variant = SystemInfo.systemMemorySize > 4096 ? "hd" : "sd";

// 加载对应变体的 AB 包
string bundlePath = $"characters/hero.{variant}";
AssetBundle ab = AssetBundle.LoadFromFile(bundlePath);

// 资源名称不变，Unity 自动从当前加载的变体中取
GameObject hero = ab.LoadAsset<GameObject>("Hero");
Instantiate(hero);
```

运行游戏时加载方式：
```cs 本地加载和从服务器下载
// 本地加载
AssetBundle ab = AssetBundle.LoadFromFile(path);
GameObject prefab = ab.LoadAsset<GameObject>("Hero");

// 网络下载（带缓存）
IEnumerator LoadFromCDN(string url, uint version)
{
    using var req = UnityWebRequestAssetBundle.GetAssetBundle(url, version, 0);
    yield return req.SendWebRequest();
    AssetBundle ab = DownloadHandlerAssetBundle.GetContent(req);
    var prefab = ab.LoadAsset<GameObject>("Hero");
    Instantiate(prefab);
    ab.Unload(false);
}
```

优点：
* 支持热更新，资源可以独立于安装包更新
* 按需加载，减少内存占用
* 支持变体，同一资源针对不同平台/设备加载不同版本
* 可以精细控制加载和卸载时机

缺点：
* 需要管理依赖关系，漏加载依赖包会导致资源丢失，或者依赖未正确处理会产生资源冗余（同一资源打进多个包）
* 开发流程复杂，需要额外的打包和版本管理系统
* Unload 使用不当会导致内存泄漏或纹理变洋红

实际用例：
* 手游热更新（关卡、角色、皮肤）
* DLC 内容下载
* 大型游戏按需加载地图/场景
* 多语言资源包（不同语言加载不同 AB）

现代 Unity 项目的推荐方案是用 Addressables，它是对 AssetBundle 的封装，自动处理依赖、缓存和版本管理，大幅降低 AB 包的使用复杂度。