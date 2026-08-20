---
title: Unity热更新
date: 2026-08-20 18:13:46
categories: 
- Game Engine
- Unity
tags: 
- Hotfix
---

Unity热更新可以通过Lua框架（xLua, toLua等）或者HybridCLR实现。

xLua 下载的是 Lua 脚本，由 Lua VM 执行。
HybridCLR 下载的是 C# DLL，由 HybridCLR 在 IL2CPP 中解释执行。

# 主要流程
如果你项目使用 Addressables，那么 catalog.json、AB 包、New Build / Update Build 的流程，对两者都一样。
Addressables 不关心文件是 Lua 还是 DLL；它只负责把指定地址的资源拿回来。
```
Player 内置启动器
  ↓
检查远端 catalog.hash / catalog.json
  ↓
下载变化的 AssetBundle
  ↓
从 AB 取出热更新文件
  ↓
加载并运行热更新逻辑
```

首次发布Android平台远端资源时，可以这样做：
```
Addressables → New Build
  ↓
ServerData/Android/
  ↓
将其中内容部署到 Cloudflare/CDN 的 Android 资源目录
```

例如：
```
本地：
ServerData/Android/
  catalog_xxx.json
  catalog_xxx.hash
  hotcode_a1.bundle
  ui_b1.bundle

CDN：
https://assets.example.com/aa/Android/
  catalog_xxx.json
  catalog_xxx.hash
  hotcode_a1.bundle
  ui_b1.bundle
```

需要确保以下两点：
1. 指定远端资源构建。需要启用 Build Remote Catalog，并将需要热更的 Group 设置为：
```
Build Path = Remote.BuildPath
Load Path = Remote.LoadPath
```
仅在当前项目点击 New Build，因远端 Catalog 关闭、默认 Group 是 Local，未必会产生可部署的 ServerData/Android 完整热更新内容。

2. Remote.LoadPath 必须在构建前指向真实 CDN 地址，例如：
```
https://assets.example.com/aa/[BuildTarget]
```
不要先用 localhost 构建，再直接把文件传到 Cloudflare；Catalog 中会保留构建时写入的 LoadPath。

首次发布后，后续的操作不同：
```
Update Previous Build
  ↓
只产生新增/变化的 Bundle 与新 Catalog
  ↓
合并到服务器已有 Android 资源目录
```
不要清空服务器 Android 目录，也不要只部署本次 Update Build 的少量输出。新 Catalog 文件替换，旧 Bundle 保留。

# xLua：Lua 驱动 C#
xLua 的核心是把 Lua VM 放进 Player：
```
C# AOT Player
  ↓
LuaEnv
  ↓
下载 Main.lua、Login.lua、Battle.lua
  ↓
Lua 脚本解释执行
  ↓
通过绑定层调用 Unity/C#
```
游戏启动时，C# 通常创建一个全局 LuaEnv
所以它不是“把 Lua 脚本编译进原生 Player”；而是 Player 预先带着 Lua 解释器，后续下载 Lua 脚本，解释器负责运行这些新脚本。

Lua 与 C# 的交互通常需要配置生成：
* CSharpCallLua：让 C# 调用 Lua。
* LuaCallCSharp：让 Lua 高效调用 C#。
* Delegate、Interface、泛型、重载等都要处理绑定与代码生成。

## xLua 的两种“热更新”

* Lua 业务热更新
这是最常见、最稳的方式：
```
C#：框架、Unity 封装、网络、资源管理
Lua：UI 流程、活动、战斗规则、业务逻辑
```
后续下载新的 Lua 文件即可生效。
优点是包小、更新快；代价是业务层需要长期使用 Lua 开发，并维护 Lua/C# 边界。

* xLua Hotfix：Lua 覆盖 C# 方法
xLua 还有 Hotfix 注入机制。你在发包前标记/生成可热修复的 C# 方法，构建时 xLua 向这些方法插入跳板：
```
原 C# 方法
  ↓
预先注入 Hotfix 检查点
  ↓
运行时 Lua 可替换该方法实现
```
例如 Lua 可以通过 xlua.hotfix(...) 替换一个已预留的 C# 方法。
关键限制是：xLua 不能在 Player 发布后，任意修改一个当初没有注入 Hotfix 跳板的 C# 方法。
也就是说，它是“预埋可替换点”，不是发布后任意加载新 C# 代码。

# HybridCLR：运行新的 C# 逻辑
HybridCLR 则更接近“可加载新的 C# 程序集”，但同样受基础 Player 的 API、裁剪、泛型、桥接代码约束。

```
C# AOT Player
  ↓
HybridCLR Runtime
  ↓
下载 Game.HotUpdate.dll.bytes
  ↓
Assembly.Load
  ↓
解释执行新的 C# IL
```

热更新代码仍是 C#：
* 可以新增普通 C# 类、接口、业务系统。
* 可以使用熟悉的 C# 语法、async、LINQ、泛型、异常等。
* 可以直接引用已有 AOT 层 API。
* 主体逻辑通常无需改写成另一种语言。
但它不能改变已经编译进 Player 的原生 AOT 代码；它解决的是“新增/替换热更新程序集中的逻辑”。

# Catalog 与 AB包
假设热更入口地址为 `HotEntry`。

* xLua
```
catalog.json
  "HotEntry"
    → lua_main_abc.bundle

lua_main_abc.bundle
  → Main.lua
  → Login.lua
  → Battle.lua
```

运行时：
```cs
luaEnv.DoString(luaTextAsset.text, "Main.lua");
```

* HybridCLR
```
catalog.json
  "HotEntry"
    → hotcode_abc.bundle

hotcode_abc.bundle
  → Game.HotUpdate.dll.bytes
```

运行时：
```cs
Assembly.Load(hotDllTextAsset.bytes);
```

HybridCLR 可能还多一组
```
aotmetadata_abc.bundle
  → Game.Runtime.dll.bytes
  → System.dll.bytes
```
这些 AOT 元数据要在加载热更新 DLL 前处理。

## New Build / Update Build 对比
它们的发布方式一致。第一次发包：
```
xLua：
New Build → Lua AB + Catalog → 打入/上传

HybridCLR：
生成 HybridCLR 必要代码
→ 构建 Player
→ 打包热更新 DLL / AOT 元数据到 AB
→ New Build → Catalog + AB
```

后续只改热更逻辑：
```
xLua：
改 Lua → Update Previous Build → 上传新 Lua AB + Catalog

HybridCLR：
编译新 HotUpdate.dll
→ Update Previous Build
→ 上传新 DLL AB + Catalog
```

两者都不是二进制增量补丁：
* 修改一个 Lua 文件，通常重下它所在的整个 Lua Bundle。
* 修改一个 DLL，通常重下它所在的整个 DLL Bundle。
* Bundle 划分越细，下载越小；Bundle 越大，依赖和管理越简单。