---
title: 裁剪空间和NDC
date: 2025-09-05 11:35:46
categories: Graphics
tags:
- Geometry
- Matrix
- Transformation
- Rendering pipeline
---

Vertex Shader的输出在Clip Space，然后GPU自己做透视除法变到了NDC( 取值范围[-1, 1] )。

---

# 裁剪空间

裁剪空间变换的思路是，对平截头体进行缩放，使近裁剪面和远裁剪面变成正方形，使坐标的w分量表示裁剪范围，此时，只需要简单的比较x,y,z和w分量的大小即可裁剪图元。
完全位于这块空间内部的图元将会被保留，完全位于这块空间外部的图元将会被剔除，与这块空间边界相交的图元就会被裁剪。而这块空间就是由视椎体来决定的。

In clip space, clipping is not done against a unit cube. It is done against a cube with side-length w. Points are inside the visible area if each of their x,y,z coordinate is smaller than their w coordinate.

In the example you have, the point [6, 0, 6.29, 7] is visible because all three coordinates (x,y,z) are smaller than 7.

### 透视投影矩阵

![](p1.jpg "Perspective Projection Matrix") <br>

此时我们就可以按如下不等式来判断一个变换后的顶点是否位于视椎体内
![](p3.png "Clipping") <br>

### 正交投影矩阵

![](p2.jpg "Orthogonal Projection Matrix") <br>

判断一个变换后的顶点是否位于视椎体内使用的不等式和透视投影中的一样，这种通用性也是为什么要使用投影矩阵的原因之一。

---

# NDC

齐次除法将Clip Space顶点的4个分量都除以w分量，就从Clip Space转换到了NDC了。
而NDC是一个长宽高取值范围为[-1, 1]的立方体，超过这个范围的顶点，会被GPU剪裁。也就是说，每个顶点的x，y，z坐标都应该在-1.0到1.0之间，超出这个坐标范围的顶点都将不可见。

### 透视投影除法

![](p5.png "齐次除法")

### 正交投影除法

![](p6.png "齐次除法") <br>

细心一点会发现，齐次坐标对于透视投影空的裁剪空间变化更大，而对正交投影的裁剪空间没有影响（正交投影的裁剪空间中顶点的w已经是1了）。

### 视口变换(Viewport Transformation)

At this moment, we’re still in 3D space.How do we get to 2D space?

We need to transform our vertex from 3D NDC to 2D screen coordinates.

When initializing the Canvas, we are responsible for configuring its size. This size is used to convert our NDC coordinates to screen coordinates.

![](p4.png "viewport transformation")