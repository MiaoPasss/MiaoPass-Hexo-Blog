---
title: 坐标空间和MVP变换
date: 2025-08-26 17:47:30
categories: Graphics
tags:
- Geometry
- Matrix
- Transformation
- Rendering pipeline
---

---

* 坐标空间转换：在渲染管线中，把一个点或一个向量从一个坐标空间转换到另一个坐标空间，比如模型空间 -> 裁剪空间
* 变换矩阵：实现坐标空间转换的过程，则需要用到变换矩阵
* 顶点着色器：顶点着色器是图形渲染管线中对每个顶点进行坐标变换的程序，而MVP变换（模型Model - 视图View - 投影Projection）是顶点着色器中一种将顶点坐标从模型空间转换为裁剪空间的常用技术。

---

# 1. 模型空间(Model Space)

又称为对象空间(Object Space)或局部空间(Local Space)，每个模型都有属于自己的模型空间

* 以模型本身为参考系，会随着模型的旋转/移动而旋转/移动
* 包含前、后、左、右等自然方向概念. 

在模型空间中描述模型上的某一点位置时，坐标会被扩展到齐次坐标系下：(1,0,0) -> (1,0,0,1)，为顶点变换中的平移变换做准备

---

# 2. 世界空间(World Space)



---

# Credits:

https://blog.csdn.net/qq_41835314/article/details/126851074

---