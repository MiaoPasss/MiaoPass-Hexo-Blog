---
title: Unity粒子特效
date: 2026-04-13 13:23:57
categories: 
- Game Engine
- Unity
---

Unity 的粒子特效主要基于内置的 `Particle System` 组件。它适合创建火焰、烟雾、爆炸、雪花、魔法光效等视觉效果。
粒子系统通过一组不断生成、模拟、渲染的“粒子”来表现复杂的视觉效果，而不需要为每一个物体写单独的网格。

# 粒子系统的核心概念

- **Emitter（发射器）**：粒子从何处发射，通常由发射形状（Shape）决定。
- **Lifetime（寿命）**：每个粒子存在的时间。
- **Velocity（速度）**：粒子运动方向与速度。
- **Size / Color over Lifetime**：粒子在生命周期中大小和颜色的变化。
- **Emission Rate（发射率）**：每秒生成粒子数量。
- **Renderer**：粒子最终如何绘制，常见为 Billboard、Mesh、Stretch Billboard。

# Unity 粒子系统常用模块

- `Emission`：控制发射速率、爆发次数。
- `Shape`：控制粒子发射区域，如 Sphere、Cone、Box 等。
- `Velocity over Lifetime`：设置粒子在生命周期内的速度变化。
- `Color over Lifetime`：设置颜色渐变。
- `Size over Lifetime`：控制大小随时间变化。
- `Rotation over Lifetime`：控制粒子旋转。
- `Noise`：添加噪声让粒子运动更自然。
- `Collision`：让粒子与世界碰撞。

# 混合代码与粒子系统使用方式

粒子系统通常由编辑器配置，但也可以通过脚本动态控制。下面是一个常见的代码用例：创建、配置并触发粒子效果。

```cs
using UnityEngine;

public class ParticleEffectSpawner : MonoBehaviour
{
    // `ParticleSystem` 组件可直接挂在预制体上，脚本通过 `Instantiate` 生成实例。
    public ParticleSystem particlePrefab;

    void Start()
    {
        if (particlePrefab == null)
        {
            Debug.LogWarning("请在 Inspector 中指定 particlePrefab");
            return;
        }
    }

    public void SpawnEffect(Vector3 position)
    {
        var instance = Instantiate(particlePrefab, position, Quaternion.identity);

        // `main`、`emission`、`shape` 等模块都可以在运行时通过脚本访问。
        var main = instance.main;
        main.startLifetime = 1.5f;
        main.startSpeed = 4f;
        main.startSize = 0.8f;
        main.startColor = Color.cyan;

        var emission = instance.emission;
        emission.rateOverTime = 100f;

        var shape = instance.shape;
        shape.shapeType = ParticleSystemShapeType.Cone;
        shape.angle = 20f;
        shape.radius = 0.5f;

        instance.Play();

        // `Destroy` 用于清理粒子对象，避免特效结束后残留内存。
        Destroy(instance.gameObject, main.startLifetime.constant + 0.5f);
    }
}
```

# 运行时参数控制示例

通过脚本修改粒子参数，可以实现不同风格的特效，例如爆炸效果：

```csharp
public void Explode(Vector3 position)
{
    var instance = Instantiate(particlePrefab, position, Quaternion.identity);
    var main = instance.main;
    main.startLifetime = 0.8f;
    main.startSpeed = 10f;
    main.startSize = 1.5f;
    main.startColor = new Color(1f, 0.5f, 0.1f);

    var emission = instance.emission;
    emission.rateOverTime = 0f;
    emission.SetBurst(0, new ParticleSystem.Burst(0f, 50, 80));

    var shape = instance.shape;
    shape.shapeType = ParticleSystemShapeType.Sphere;
    shape.radius = 0.3f;

    instance.Play();
    Destroy(instance.gameObject, main.startLifetime.constant + 0.5f);
}
```

# 与 Shader 的区别

Unity 粒子特效通常由 `Particle System` 来驱动整体行为，而 `Shader` 决定粒子的最终视觉表现。
两者搭配使用，既可以快速构建特效逻辑，也能在渲染上实现更加细腻的视觉结果。

## 粒子系统和 Shader 的职责不同

- `Particle System`：负责粒子的生成、生命周期、位置、速度、发射、碰撞、颜色和大小变化等整体行为。
- `Shader`：负责单个粒子的最终渲染效果，如纹理、颜色混合、透明度、光照、动画遮罩等。

粒子系统通常使用内置或自定义材质，材质中可以包含 Shader。也就是说，粒子系统负责“做出粒子”，Shader 负责“怎么画粒子”。

## 使用场景差异

- 粒子系统适合快速搭建常见特效、控制复杂行为、在编辑器中调整参数。
- Shader 适合实现特效的渲染细节，如渐隐、闪烁、扫描、光晕与特殊混合模式。

例如：你可以让 `Particle System` 负责生成烟雾粒子，而使用 `Unlit` 或 `Additive` Shader 实现在透明面上绘制柔和、发光的烟雾纹理。

## 性能与复杂度

- `Particle System` 自身封装了大量行为，适合面向特效设计师快速迭代。
- `Shader` 则在 GPU 上运行、效率更高，但需要编写渲染代码，对每个像素/顶点进行控制。

如果希望增强粒子表现，可以结合 `Shader Graph` 或自定义 Shader，使粒子面具备“流动光晕”、“波动边缘”、“复杂透明度曲线”等效果。

## 结合 Shader 的粒子效果

一个常见做法是：

1. 在 `Particle System` 中设置材质为 `Particle/Additive` 或自定义粒子材质。
2. 使用纹理图集、`Color over Lifetime`、`Size over Lifetime` 调整粒子表现。
3. 在 Shader 中实现附加效果，例如渐隐、扭曲、噪声贴图 UV 变换。

# 自定义 Shader 示例

下面是一个简单的粒子自定义 Shader，它对粒子透明度和 UV 进行了处理：

```shader
Shader "Custom/ParticleSoft"
{
    Properties
    {
        _MainTex ("Texture", 2D) = "white" {}
        _Color ("Tint", Color) = (1,1,1,1)
    }
    SubShader
    {
        Tags { "Queue"="Transparent" "RenderType"="Transparent" }
        Blend SrcAlpha One
        Cull Off
        Lighting Off
        ZWrite Off

        Pass
        {
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "UnityCG.cginc"

            sampler2D _MainTex;
            float4 _Color;

            struct appdata
            {
                float4 vertex : POSITION;
                float2 uv : TEXCOORD0;
                float4 color : COLOR;
            };

            struct v2f
            {
                float4 pos : SV_POSITION;
                float2 uv : TEXCOORD0;
                float4 color : COLOR;
            };

            v2f vert(appdata v)
            {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                o.uv = v.uv;
                o.color = v.color * _Color;
                return o;
            }

            fixed4 frag(v2f i) : SV_Target
            {
                fixed4 tex = tex2D(_MainTex, i.uv);
                tex.rgb *= i.color.rgb;
                tex.a *= i.color.a;
                return tex;
            }
            ENDCG
        }
    }
}
```

这个 Shader 只负责将粒子贴图按颜色和透明度渲染出来，而粒子系统仍然负责产生位置、速度和生命周期。

## 实战建议

- 先用编辑器调整粒子系统参数，确认效果后再用脚本控制动画和触发时机。
- 将复杂行为拆成多个粒子系统，如火焰、烟雾、火花分别使用不同系统组合。
- 如果粒子数量大或效果复杂，可考虑粒子 Shader 优化，例如使用 `Soft Particles`、`GPU Instancing` 或 `Shader Graph`。
- 当粒子需要与场景光照、阴影或深度融合时，配合自定义 Shader 和深度测试设置提高表现力。