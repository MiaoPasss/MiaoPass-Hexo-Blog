---
title: Drawcall和SetPassCall和Batching
date: 2026-03-27 18:28:12
categories: 
- Game Engine
- Unity
tags:
- Rendering pipeline
- Shader
---

DrawCall 是 CPU 向 GPU 发送的一次渲染指令，包含了 GPU 绘制所需的全部信息：几何数据、纹理、Shader、Buffer 等。
每次调用 DrawMesh、DrawRenderer 等，就是一个 Draw Call。GPU 收到命令后开始渲染对应的几何体。

```
CPU 准备渲染数据
    ↓
设置渲染状态（材质、Shader、纹理绑定）← SetPass Call
    ↓
提交 DrawCall（告诉 GPU 画什么、怎么画）
    ↓
GPU 执行光栅化、着色、输出像素
```
关键点：DrawCall 本身消耗资源，但准备阶段（SetPass Call）通常比 DrawCall 本身更耗性能。
开启批处理后，Unity 可以用一次 SetPass Call 复用同一材质状态，然后提交多个 DrawCall。所以降低 SetPass Call 往往比降低 DrawCall 更重要。

# DrawCall（绘制调用）
CPU准备数据并通过调用图形API接口命令GPU对指定物体进行渲染一次的操作称为一次DrawCall。
当收到一个DrawCall时，GPU会按照指令，根据渲染状态和输入的顶点信息，指定一个网格（Mesh）进行渲染，绘制材质。
GPU 渲染时，同一批次的对象必须使用相同的材质和纹理。一旦纹理不同，就必须切换渲染状态，触发新的 DrawCall。
```
Image A（纹理1）→ DrawCall 1
Image B（纹理2）→ DrawCall 2（纹理不同，无法合批）
Image C（纹理1）→ 可以与 A 合批，不增加 DrawCall
```

为了CPU和GPU可以进行并行工作，需要一个命令缓冲区，由CPU向其中添加命令，然后由GPU从中读取命令，这样就实现了通过CPU准备数据，通知GPU进行渲染。

# SetPass Call
在 Draw Call 之前，如果需要切换渲染状态（shader、材质属性、纹理等），CPU 就要先做一次 SetPass Call 来通知 GPU 切换 Pass。
SetPass Call 才是真正的性能瓶颈，因为它涉及 shader 编译、状态切换等开销。

例如场景里有 5 个物体：
```
物体A - 材质1 (Shader: Standard, 纹理: tex_a)
物体B - 材质1 (同上)
物体C - 材质1 (同上)
物体D - 材质2 (Shader: Standard, 纹理: tex_b)
物体E - 材质3 (Shader: Unlit,   纹理: tex_c)
```

渲染顺序：
```
SetPass Call → 切换到材质1
  Draw Call → 绘制物体A
  Draw Call → 绘制物体B
  Draw Call → 绘制物体C
SetPass Call → 切换到材质2
  Draw Call → 绘制物体D
SetPass Call → 切换到材质3
  Draw Call → 绘制物体E
```
结果：5 个 Draw Call，3 个 SetPass Call。

## Pass 是什么
一个 Pass 就是 shader 里的一次完整渲染流程，包含一套完整的顶点着色器 + 片元着色器，以及对应的渲染状态（混合模式、深度测试、剔除等）。
一个材质可以有多个 Pass，每个 Pass 对同一个物体渲染一遍。

* 具体例子：描边效果
描边通常需要两个 Pass：第一个 Pass 画描边（把模型沿法线方向放大，只渲染背面），第二个 Pass 画正常颜色。

```hlsl
Shader "Custom/Outline"
{
    Properties
    {
        _Color ("Color", Color) = (1,1,1,1)
        _OutlineColor ("Outline Color", Color) = (0,0,0,1)
        _OutlineWidth ("Outline Width", Float) = 0.02
    }

    SubShader
    {
        // ---- Pass 0：描边 ----
        Pass
        {
            Name "OUTLINE"
            Cull Front  // 只渲染背面，正面被裁掉

            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            float _OutlineWidth;
            float4 _OutlineColor;

            struct appdata { float4 vertex : POSITION; float3 normal : NORMAL; };
            struct v2f    { float4 pos : SV_POSITION; };

            v2f vert(appdata v)
            {
                v2f o;
                // 沿法线方向把顶点往外推
                float3 expanded = v.vertex.xyz + v.normal * _OutlineWidth;
                o.pos = UnityObjectToClipPos(float4(expanded, 1));
                return o;
            }

            fixed4 frag(v2f i) : SV_Target
            {
                return _OutlineColor;  // 纯色描边
            }
            ENDCG
        }

        // ---- Pass 1：正常渲染 ----
        Pass
        {
            Name "BASE"
            Cull Back   // 只渲染正面

            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            float4 _Color;

            struct appdata { float4 vertex : POSITION; };
            struct v2f    { float4 pos : SV_POSITION; };

            v2f vert(appdata v)
            {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                return o;
            }

            fixed4 frag(v2f i) : SV_Target
            {
                return _Color;
            }
            ENDCG
        }
    }
}
```

渲染这一个物体时，GPU 会走两遍：
```
SetPass Call → 激活 Pass 0 (OUTLINE)，设置 Cull Front 状态
  Draw Call  → 用 Pass 0 的 shader 绘制物体（画出描边）

SetPass Call → 激活 Pass 1 (BASE)，设置 Cull Back 状态
  Draw Call  → 用 Pass 1 的 shader 绘制物体（画出本体）
```
所以 1 个物体、1 个材质，产生了 2 个 SetPass Call + 2 个 Draw Call。

# Batching
静态合批: 将相同材质并且始终不动的的Mesh合并成为一个大Mesh，然后由CPU合并为一个批次发送给GPU处理,从而减少DrawCall带来的消耗。
动态合批: 相同材质并且会动的物体Mesh合并成为一个大Mesh
SRP Batcher: 将多个网格合并成单个批次进行渲染，从而提高性能。与其他合批不同，SRP Batcher将未改变属性的Mesh缓存起来，从而减少消耗
GPU Instancing: 使用一个DrawCall渲染多个相同材质的网格对象（如场景中大量重复的物体如树木和草地等），从而减少CPU和GPU的开销

## Static Batching
针对静态物体（勾选 Static）。在构建时，Unity 把所有使用相同材质的静态 mesh 合并成一个大 mesh，存到内存里。运行时一次 Draw Call 画完。

```
场景里100棵树（同材质，Static）：
  构建时合并 → 1个大mesh
  运行时：SetPass × 1，Draw Call × 1
```
代价是内存，合并后的大 mesh 常驻内存，物体也不能移动。

## Dynamic Batching
与静态合批不同，动态合批是的物体是可以运动的，针对动态小物体。每帧 CPU 实时把符合条件的 mesh 合并，然后一次提交。
条件很苛刻：顶点数 < 900，不能使用lightmap，不能使用多通道的shader，不能是缩放比不同的物体。
Unity 的 Dynamic Batching（动态批处理）是自动触发的。只要启用了该功能且物体满足特定条件（如使用相同材质、顶点数较少），Unity 引擎就会在运行时自动合并 Draw Call，无需手动干预。
```
每帧：
  CPU 检查哪些动态物体同材质 → 临时合并 mesh → 一次 Draw Call
```
代价是 CPU 每帧都要做合并，物体多了反而更慢，现在基本被 GPU Instancing 取代。

## SRP Batcher 
Batches draw calls by grouping objects that use the same shader variant, reducing SetPass calls
Shader properties stored in GPU buffers instead of individual material constants

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

```hlsl
// shader 里声明每个实例可以有不同颜色
UNITY_INSTANCING_BUFFER_START(Props)
    UNITY_DEFINE_INSTANCED_PROP(float4, _Color)
UNITY_INSTANCING_BUFFER_END(Props)
```

使用条件：
* 材质需要支持GPU Instancing,例如默认标准材质就有
* Tranform信息需要有所不同,(完全重合了渲染出来也没有意义)
* 未使用SRP Batcher，如有会优先使用SRP Batcher。(在URP渲染管线中是默认开启的)
* 粒子对像不能合批
* 使用MaterialPropertyBlocks的游戏不能合批
* Shader必须是使用compatible的

此时的CPU依然只提交1次Drawcall
```
100棵树（同mesh，同材质，不同位置/颜色）：
  CPU：提交1次mesh + 100个transform数组
  GPU：自己 instance 100次
  结果：SetPass × 1，Draw Call × 1
```