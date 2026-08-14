---
title: unity 计算点到直线距离
date: 2026-08-07 17:44:17
categories:
- Game Engine
- Unity
tags:
- Geometry
---

## 点到直线距离

### 方法一：投影 + 勾股定理

利用点到直线上某点的向量长度，以及它在直线方向上的投影长度，用勾股定理求出垂直分量。

```csharp
// 点到直线距离
public static float DistancePoint2Line(Vector3 point, Vector3 linePoint1, Vector3 linePoint2)
{
    float fProj = Vector3.Dot(point - linePoint1, (linePoint1 - linePoint2).normalized);
    return Mathf.Sqrt((point - linePoint1).sqrMagnitude - fProj * fProj);
}
```

原理：`|d|² = |斜边|² - |投影|²`（勾股定理）。

> 注意：在浮点误差下 `sqrMagnitude - fProj²` 可能出现极小的负数，导致 `Mathf.Sqrt` 返回 `NaN`，必要时可对被开方数做一次 `Mathf.Max(0, ...)` 保护。

### 方法二：叉乘（面积法）

两向量叉乘的模等于它们所构成平行四边形的面积，面积除以底边长度即为高（也就是点到直线的距离）。这种方式在 3D 中最推荐，代码简洁且数值稳定。

```csharp
public static float DistancePoint2Line_Cross(Vector3 point, Vector3 linePoint1, Vector3 linePoint2)
{
    Vector3 dir = linePoint2 - linePoint1;
    Vector3 v = point - linePoint1;
    return Vector3.Cross(v, dir).magnitude / dir.magnitude;
}
```

原理：`距离 = 平行四边形面积 / 底 = |v × dir| / |dir|`。

### 方法三：先求投影点，再求两点距离

先算出点在直线上的投影点坐标，再直接用 `Vector3.Distance`。好处是能顺便拿到最近点（投影点）。

```csharp
public static float DistancePoint2Line_Project(Vector3 point, Vector3 linePoint1, Vector3 linePoint2)
{
    Vector3 dir = (linePoint2 - linePoint1).normalized;
    Vector3 projPoint = linePoint1 + Vector3.Dot(point - linePoint1, dir) * dir;
    return Vector3.Distance(point, projPoint);
}
```

> 补充：以上三种求的都是**无限长直线**的距离。如果要的是**线段（Segment）**距离，需要把投影参数 `t` 用 `Mathf.Clamp01` 限制在 `[0, 1]` 之间，再求点到该最近点的距离。

## 点到面距离

点到平面的距离公式：`d = |Ax0 + By0 + Cz0 + D| / √(A² + B² + C²)`。

平面的一般式方程为 `Ax + By + Cz + D = 0`，平面的法向量为 `(A, B, C)`。

> 如果一个非零向量 n 与平面 a 垂直，则称向量 n 为平面 a 的法向量。垂直于平面的直线所表示的向量即为该平面的法向量。每一个平面存在无数个法向量。

### 方法一：用 Plane 类

Unity 内置的 `Plane` 类可以由三点构造，并直接求点到平面的（有符号）距离。

```csharp
public static float DisPoint2Surface_Plane(Vector3 point, Vector3 surfacePoint1, Vector3 surfacePoint2, Vector3 surfacePoint3)
{
    Plane plane = new Plane(surfacePoint1, surfacePoint2, surfacePoint3);
    return plane.GetDistanceToPoint(point);
}
```

### 方法二：用法线 + 点乘

由平面上两条边叉乘得到法向量，再把「点到平面上一点」的向量投影到法向量方向上，其绝对值即为距离。

```csharp
public static float DisPoint2Surface_Normal(Vector3 point, Vector3 surfacePoint1, Vector3 surfacePoint2, Vector3 surfacePoint3)
{
    Vector3 cross = Vector3.Cross(surfacePoint1 - surfacePoint2, surfacePoint1 - surfacePoint3);
    float proj = Vector3.Dot(point - surfacePoint1, cross.normalized);
    return Mathf.Abs(proj);
}
```