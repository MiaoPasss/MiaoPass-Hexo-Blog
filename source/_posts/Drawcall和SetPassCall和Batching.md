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
每次调用 DrawMesh、DrawRenderer 等，就是一个 Draw Call。GPU 收到命令后开始渲染对应的几何体。
GPU 渲染时，同一批次的对象必须使用相同的材质和纹理。一旦纹理不同，就必须切换渲染状态，触发新的 DrawCall。
```
Image A（纹理1）→ DrawCall 1
Image B（纹理2）→ DrawCall 2（纹理不同，无法合批）
Image C（纹理1）→ 可以与 A 合批，不增加 DrawCall
```

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
Static Batching / GPU Instancing 能合并 Draw Call，但不能合并 SetPass Call 。
```
// GPU Instancing：同一材质的100个物体(DrawCall x 100)
SetPass Call × 1   ← 只切换一次状态
Draw Call × 1      ← 合并成一次绘制（instanced）
```
而 Dynamic Batching 同时减少两者，因为它把多个 mesh 合并成一个，用同一个材质一次画完。

# 总结
* Draw Call 多 → GPU 压力大
* SetPass Call 多 → CPU 压力大（状态切换开销）
* 先降 SetPass Call（合并材质、用图集），再降 Draw Call（Batching、Instancing）

Unity Stats 窗口里显示的 "SetPass Calls" 和 "Batches" 分别对应这两个指标，看这两个数比只看 Draw Call 更有参考价值。