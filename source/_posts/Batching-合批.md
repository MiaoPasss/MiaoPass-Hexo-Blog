---
title: Batching 合批
date: 2026-06-04 13:25:14
categories: 
- Game Engine
- Unity
tags:
- Rendering pipeline
---

# Batching
静态合批: 将相同材质并且始终不动的的Mesh合并成为一个大Mesh，然后由CPU合并为一个批次发送给GPU处理,从而减少DrawCall带来的消耗。
动态合批: 相同材质并且会动的物体Mesh合并成为一个大Mesh
SRP Batcher: 将多个网格合并成单个批次进行渲染，从而提高性能。与其他合批不同，SRP Batcher将未改变属性的Mesh缓存起来，从而减少消耗
GPU Instancing: 使用一个DrawCall渲染多个相同材质的网格对象（如场景中大量重复的物体如树木和草地等），从而减少CPU和GPU的开销

## Static Batching
针对静态物体（勾选 Static）。在构建时，Unity 把所有使用相同材质的静态 mesh 合并成一个大 mesh，存到内存里。运行时一次 Draw Call 画完。

| 特性 | 描述 |
| :--- | :--- |
| **核心思想** | 在**编辑器时**，将标记为 `Static` 且使用**相同材质**的游戏对象的网格（Mesh）合并成一个或多个巨大的网格。 |
| **工作原理** | 1. 你在编辑器中将多个物体勾选为 `Static`。<br>2. 在构建或进入播放模式时，Unity 找到使用相同材质的静态物体，将它们的顶点和索引数据合并成一个新的、巨大的网格。<br>3. 运行时，渲染这个大网格只需要**一次 Draw Call**。 |
| **主要优点** | ✅ **大幅减少 Draw Call**，性能提升非常显著。 |
| **主要缺点** | ❌ **内存和显存开销大**：合并后的网格需要额外存储，且无法进行视锥体剔除（Culling）的细粒度操作（整个大网格要么全被剔除，要么全不被剔除）。<br>❌ **不能用于非静态物体**：物体不能移动、旋转、缩放。 |
| **适用场景** | 场景中静止不动的所有物体，如建筑、地面、大块的岩石、静态的装饰物等。 |

```
场景里100棵树（同材质，Static）：
  构建时合并 → 1个大mesh
  运行时：SetPass × 1，Draw Call × 1
```
代价是内存，合并后的大 mesh 常驻内存，物体也不能移动。

## Dynamic Batching
与静态合批不同，动态合批是的物体是可以运动的，针对动态小物体。每帧 CPU 实时把符合条件的 mesh 合并，然后一次提交。
条件很苛刻：顶点数 < 300，不能使用lightmap，不能使用多通道的shader，不能是缩放比不同的物体。
Unity 的 Dynamic Batching（动态批处理）是自动触发的。只要启用了该功能且物体满足特定条件（如使用相同材质、顶点数较少），Unity 引擎就会在运行时自动合并 Draw Call，无需手动干预。

| 特性 | 描述 |
| :--- | :--- |
| **核心思想** | 在**每一帧运行时**，由 **CPU** 自动寻找符合条件的小网格，并将它们合并成一个临时的网格进行绘制。 |
| **工作原理** | 1. CPU 寻找使用**相同材质**且**顶点数极少**（通常<300）的移动物体。<br>2. CPU 将这些物体的顶点从局部空间变换到世界空间。<br>3. CPU 将变换后的顶点数据拷贝到一个大的临时缓冲区中。<br>4. 用一次 Draw Call 将这个临时缓冲区提交给 GPU。 |
| **主要优点** | ✅ 对移动的小物体能减少一些 Draw Call。 |
| **主要缺点** | ❌ **巨大的 CPU 开销**：每一帧都在CPU上做顶点变换和拷贝，往往得不偿失。<br>❌ **限制非常严格**：对顶点数、顶点属性等有诸多要求。<br>❌ **在 SRP (URP/HDRP) 中已废弃/不推荐**：因为它与更高效的 SRP Batcher 工作机制冲突。 |
| **适用场景** | 在旧的内置渲染管线（Built-in RP）中，用于处理大量简单粒子、子弹等小物体。**在现代项目中几乎不再使用**。 |

```
每帧：
  CPU 检查哪些动态物体同材质 → 临时合并 mesh → 一次 Draw Call
```
代价是 CPU 每帧都要做合并，物体多了反而更慢。

## SRP Batcher 
Batches draw calls by grouping objects that use the same shader variant, reducing SetPass calls
Shader properties stored in GPU buffers instead of individual material constants

创建大型 GPU 缓冲区: 在每一帧开始时，SRP Batcher 会在 GPU 上开辟一个巨大的、持久化的缓冲区 (在现代图形API中通常是 Constant Buffer 或 UBO)。
打包所有数据: 它会遍历所有使用相同 Shader Variant（着色器变体）的物体，把它们的材质属性（颜色、向量、浮点数等）全部打包塞进这个大缓冲区里。

现在，当 CPU 指示 GPU 绘制时，CPU 对 GPU 说：“准备画一批物体，它们都用 Shader A。它们的材质数据已经全部放在那个大缓冲区里了。” (这对应一个非常快速的、几乎可以忽略不计的 SetPass 准备工作)
然后 CPU 快速地连续下达指令：
    “画第一个物体，它的数据在缓冲区偏移量0的位置。” (一个 Draw call)
    “画第二个物体，它的数据在缓冲区偏移量64的位置。” (一个 Draw call)
    “画第三个物体，它的数据在缓冲区偏移量128的位置。” (一个 Draw call)

SetPass call: 只有在切换 Shader Variant 时才会发生一次。对于所有使用相同 Shader Variant 的物体，无论它们有多少个，都属于同一个“SRP Batch”，共享一次 SetPass 设置。因此 SetPass call 的数量大幅降低。
Draw call: 每个物体最终还是需要 GPU 单独绘制一次。CPU 依然需要为每个物体都发送一个 Draw 指令。因此 Draw call 的数量没有改变。

| 特性 | 描述 |
| :--- | :--- |
| **核心思想** | 大幅降低因材质不同而产生的 **SetPass Call** 开销，但**不减少 Draw Call**。 |
| **工作原理** | 1. 找到所有使用**相同 Shader Variant**（着色器变体）的物体。<br>2. 将这些物体所有不同的材质属性（颜色、浮点数等）打包到一个巨大的、持久化的 GPU 缓冲区 (Constant Buffer / Uniform Buffer Object) 中。<br>3. 每次绘制时，CPU 不再上传完整的材质数据，而是只告诉 GPU 去这个大缓冲区的哪个**偏移量**位置读取数据。<br>4. 这使得切换材质的成本变得极低。 |
| **主要优点** | ✅ **极大提升 CPU 性能**，因为它解决了最耗时的 SetPass 瓶颈。<br>✅ 兼容性好，限制较少，适用于大量不同材质但相同Shader的物体。 |
| **主要缺点** | ❌ **不减少 Draw Call**。<br>❌ 需要 Shader 支持，所有材质属性必须定义在 `CBUFFER` 块中。URP 和 HDRP 的标准 Shader 已经默认支持。 |
| **适用场景** | URP/HDRP 项目中的**首选批处理方案**。适用于场景中几乎所有使用标准 Shader 的物体，极大地提升了异质化物体的渲染性能。 |

使用条件：
* 不支持内置渲染管线，支持通用(URP)/高清(HDRP)/自定义(SRP)渲染管线
* 不使用 MaterialPropertyBlocks
* Materials must use SRP-compatible shaders

在标准的渲染流程下，CPU需要收集所有场景物体的参数，场景中的材质越多CPU提交给GPU的数据就越多。
而在SRP中流程下GPU拥有数据管理的 "生命权" ,管理大量不同材质但Shader变动较小的的内容。
让数据在GPU中持久存在,从而减少消耗(SetPass Call)。

### SRP Batcher vs. Dynamic Batching

Both can be used together in SRP, but SRP Batcher takes precedence when enabled. For optimal performance in modern Unity projects, prefer SRP Batcher with proper shader setup.

Both Dynamic Batching and SRP Batcher are triggered automatically in Unity when their conditions are met:
* Unity automatically detects and performs Dynamic Batching on compatible objects at runtime
* SRP batcher automatically batches during render loop when objects use compatible shader variants enabled in the render pipeline settings (Window → Rendering → Render Pipeline → Universal Render Pipeline → General → SRP Batcher)

## GPU Instancing
针对大量相同 mesh + 相同材质的物体（但允许不同的属性，比如颜色、位置）。CPU 只提交一次 mesh 数据，附带一个"实例属性数组"，GPU 自己循环绘制 N 次。
在 Unity 中，需要在材质属性中勾选 "Enable GPU Instancing"。

```cpp
// shader 里声明每个实例可以有不同颜色
UNITY_INSTANCING_BUFFER_START(Props)
    UNITY_DEFINE_INSTANCED_PROP(float4, _Color)
UNITY_INSTANCING_BUFFER_END(Props)
```

| 特性 | 描述 |
| :--- | :--- |
| **核心思想** | 让 **GPU** 自己去复制绘制。一次 Draw Call 告诉 GPU 将**完全相同的网格和材质**绘制 N 次。 |
| **工作原理** | 1. CPU 准备一个包含 N 个实例所有不同属性（如位置、旋转、颜色）的额外数据缓冲区。<br>2. CPU 发起**一次 Draw Call**，并把网格、材质和这个属性缓冲区一并交给 GPU。<br>3. GPU 收到指令后，自己循环 N 次，每次绘制时从属性缓冲区中取出一个实例的数据应用，完成绘制。 |
| **主要优点** | ✅ **极大幅度地减少 Draw Call**，能以极高性能绘制成千上万个相同的物体。<br>✅ 每个实例可以有自己独特的位置、旋转、缩放和颜色（通过 `MaterialPropertyBlock`）。 |
| **主要缺点** | ❌ **要求网格和材质完全相同**。 |
| **适用场景** | 场景中需要大量重复出现的物体，如草、树木、石子、人群、军队、模块化建筑的窗户等。非常适合程序化生成的内容。 |

使用条件：
* 材质需要支持GPU Instancing，例如默认标准材质就有
* Tranform信息需要有所不同（完全重合了渲染出来也没有意义）
* 未使用SRP Batcher，如有会优先使用SRP Batcher。(在URP渲染管线中是默认开启的)
* 粒子对像不能合批

此时的CPU依然只提交1次Drawcall
```
100棵树（同mesh，同材质，不同位置/颜色）：
  CPU：提交1次mesh + 100个transform数组
  GPU：自己 instance 100次
  结果：SetPass × 1，Draw Call × 1
```