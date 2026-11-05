---
title: Spine 动画
date: 2026-11-05 11:41:04
categories: Animation
---

Spine 动画通常是在 Spine 编辑器里，把分层图片绑定到骨骼，再给骨骼和图片属性制作关键帧得到的。
Unity 通过 Spine 官方运行库播放它，它和 Unity 的 Animation / Animator 可以配合使用。

# 动画制作

以一个 2D 人物为例，用Spin制作动画通常是下面这个流程：
1. 准备分层图片
把头、身体、上臂、前臂、腿、武器等拆成独立图片。被其他部位遮住的部分通常也需要补画，避免转动时露出空洞。
2. 在 Spine 编辑器中建立骨骼
例如：身体 → 上臂 → 前臂 → 手。移动上臂时，前臂和手会跟着移动。
3. 把图片挂到骨骼上
简单部位可以直接挂一张图片；需要弯曲的部位，可以把图片划分成网格，并设置骨骼权重，让多个骨骼共同控制图片变形。
4. 制作动画关键帧
在时间轴上设置骨骼的位置、旋转、缩放，还可以设置图片切换、透明度、绘制顺序和网格变形，制作 `idle`、`walk`、`attack` 等动画。
5. 导出供游戏使用的资源
| 资源 | 用途 |
|---|---|
| `.json` 或 `.skel` | 骨骼、皮肤、动画等数据 |
| `.atlas` | 图片在图集中的位置和相关信息 |
| `.png` | 实际显示的图集图片 |

所以它通常不是预先渲染的一段视频，也不是每帧一张图片。游戏运行时，Spine 根据动画数据计算骨骼姿态，再生成和绘制网格。

# 在 Unity 中使用
先安装官方 spine-unity 运行库。运行库版本要与导出资源所用的 Spine 编辑器主次版本匹配，例如 Spine 4.2 导出的资源，应使用对应的 4.2 运行库。

把导出文件放入 Unity 的 Assets 后，运行库会生成相关资源，核心是 SkeletonDataAsset，然后选择适合的组件：
| 使用场景 | 组件 |
|---|---|
| 场景里的角色、怪物、特效 | `SkeletonAnimation` |
| Canvas 中的 UI 动画 | `SkeletonGraphic` |
| 想通过 Unity Animator 状态机控制 | `SkeletonMecanim` |

最常见的是 SkeletonAnimation。下面是使用方式示例：
```cs
using UnityEngine;
using Spine.Unity;

public class SpineExample : MonoBehaviour
{
    [SerializeField] private SkeletonAnimation skeleton;

    private void Start()
    {
        // 在第 0 条轨道循环播放 idle。
        skeleton.AnimationState.SetAnimation(0, "idle", true);
    }

    public void Attack()
    {
        // 播放一次 attack，结束后回到 idle。
        skeleton.AnimationState.SetAnimation(0, "attack", false);
        skeleton.AnimationState.AddAnimation(0, "idle", true, 0f);
    }
}
```

这里的 `"idle"` 和 `"attack"`，必须是 Spine 资源中已有的动画名称。
Spine 还支持动画混合、多轨道叠加、换皮肤和动画事件。例如下半身继续走路，上半身播放攻击；或者通过皮肤切换衣服、武器。

Unity 原生动画主要通过动画曲线，修改 GameObject、Transform 和组件属性。
Spine 则有自己的骨骼、插槽、附件、皮肤、网格变形和绘制顺序。它的骨骼通常不是 Unity 层级里的一组普通 Transform，所以 Unity 原生 Animation 组件无法直接读取 Spine 导出的骨骼动画数据。

* `SkeletonMecanim` 提供一层桥接，让 Animator 状态机选择和混合 Spine 动画。最终的骨骼计算与绘制仍然由 Spine 运行库完成。
* 或者可以用 Unity 原生动画让整个 Spine 角色移动、缩放，同时让 Spine 播放角色内部的走路动画。若两套动画同时修改同一个属性，则需要明确由哪一套负责。