---
title: Unity Instance ID Cache
date: 2026-03-24 11:49:36
categories: 
- Game Engine
- Unity
tags:
- Resource Management
---

At startup, the Instance ID cache is initialized with data for all Objects immediately required by the project (i.e., referenced in built Scenes), as well as all Objects contained in the Resources folder. Additional entries are added to the cache when new assets are imported at runtime or when Objects are loaded from AssetBundles. ( Note: An example of an Asset created at runtime would be a Texture2D Object created in script, like so: *var myTexture = new Texture2D(1024, 768);* )

* File GUID（资源UID）
编辑器层面，存在 .meta 文件里，唯一标识磁盘上的一个资源文件，打包后不直接使用

* Local ID
一个资源文件内部，每个 Object 的编号
比如一个 FBX 里有 Mesh、Material、动画，各有自己的 Local ID

* Instance ID（实例ID）
运行时层面，Unity 内存中每个 Object 的唯一标识，本质是一个整数，运行时动态分配
是 File GUID + Local ID 的运行时映射

```
磁盘：  File GUID + Local ID  →  唯一定位一个资源
运行时：Instance ID           →  指向内存中的 Object

Instance ID Cache = 这个映射表
{ Instance ID → (File GUID, Local ID, 内存地址或null) }
```

* Instance ID Cache 的作用

```
启动时预填充：
- 所有场景直接引用的资源
- Resources 文件夹下的所有资源
→ 这些 Object 的 Instance ID 提前注册，但不一定加载进内存

运行时追加：
- 从 AB 包加载新资源时
- 代码里 new Texture2D() 创建时
→ 新 Instance ID 动态加入缓存
```

当你持有一个对象引用，Unity 内部就是拿着 Instance ID 去缓存里查：
1. 查到且有内存地址 → 直接返回
2. 查到但没加载 → 触发加载
3. 查不到 → null（Missing）

总结：Instance ID Cache 是运行时的"对象注册表"，把内存中每个 Object 的整数 ID 映射到它的资源来源（GUID+LocalID）和内存位置。
资源 UID（File GUID）是编辑器/磁盘层面的概念，Instance ID 是运行时层面的概念，两者通过这个缓存关联起来。