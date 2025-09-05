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

![](p9.png "空间变换总览")

坐标空间有：世界空间、模型空间、摄像机空间、齐次裁剪空间、屏幕空间，以及法线映射会用到的切线空间（之前的纹理基础篇就讲到过）。 

**那为什么会有这么多个坐标空间呢？**

> 一些概念只有在特定的坐标空间下才有意义，才更容易理解。这就是为什么在渲染中我们要使用这么多坐标空间。
<div style="text-align: right">——《Unity Shader 入门精要》</div>

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

### 顶点变换Step 1 - 模型变换(MVP中的M)
Model Transformation 把3D物体从模型空间变换到世界空间

![](p1.png "cube坐标变换前")

在Unity中，我们直接给Cube拖出来就行

![](p2.png "cube坐标变换后")

我们发现，Cube位置从(0, 2, 4, 1) -> (9, 4, 18.071)，unity帮助我们把Cube的坐标完成了模型变换

---

# 2. 世界空间(World Space)

游戏场景中的最大空间，一般世界空间的原点会放在游戏空间的正中心，同时世界空间的位置就是绝对位置——这个绝对位置你可以理解成Unity里没有父节点（parent）的游戏对象的位置。

### 顶点变换Step 2 - 观察变换(MVP中的V)
View Transformation 把3D物体从世界空间变换到观察空间

此时的变换矩阵等于：把观察空间当做一个模型空间，将其变换到世界空间（用模型变换的方法），然后取此变换矩阵的逆即为 观察空间 <- 世界空间

![](p4.png "cube坐标变换前")

在unity中，我们可以直接把Cube从世界空间拖到Main Camera下，此时Cube的Transform组件变为观察空间下的坐标信息

![](p5.png "cube坐标变换后")

我们发现Cube位置变为(9, 8.839, 27.310)，但此时Transform信息是左手坐标系下的，因此正确描述在观察空间中的Cube坐标应为(9, 8.839, -27.310)

---

# 3. 观察空间(View Space)

也叫做摄像机空间(Camera Space)，摄像机在场景中不可见，但是一般会给它生成一个图标并在场景窗口可视化，例如：

![](p3.png "摄像机在场景中可视化")

* In Unity's camera/view space, the forward direction (the direction the camera is looking) is along the negative Z-axis. This means objects further away from the camera will have larger negative Z values in view space.
* 注意区分*观察空间*和*屏幕空间*，观察空间是3d，屏幕空间是2d，观察空间 -> 屏幕空间需要经过投影操作

### 顶点变换Step 3 - 投影变换(MVP中的P)
Projection Transformation 把3D物体从观察空间变换到裁剪空间

观察空间->裁剪空间的变换矩阵有更准确的称呼 —— 裁剪矩阵(Clipping Matrix)，也被叫做投影矩阵(Projection Matrix)。
此时的坐标为顶点着色器的输出，即图元顶点在裁剪空间中的坐标。

![](p10.jpg "投影矩阵")

---

# 4. 裁剪空间(Clip Space)

也被称为其次裁剪空间，渲染管线中几何阶段的裁剪步骤在这一环节完成，这一环节里我们无法操控(Non-programmable)，完全由GPU去做

## 4.1 视锥体

从观察空间到屏幕空间的途中，需要经过裁剪空间，其目的在于剔除在视野范围外的图元，而由视锥体(Frustum)决定的裁剪空间为这一剔除过程提供了便利。
很显然，场景越大，裁剪的优越性更加突出，如果不进行裁剪就直接投影到2d屏幕空间，后续会产生非常多不必要的开销，例如渲染完全在电脑屏幕外的图元。

视锥体是观察空间中的一块区域，决定着摄像机的可见范围（即最终在屏幕上可见并渲染的物体），它由六个面组成，被称为裁剪平面(Clipping Planes)。

## 4.1.1 透视投影

![](p6.png "透视投影参数")

* FOV: 视角度数，同时FOV Axis决定这个视角是横向还是纵向
* Clipping Planes: 设置近裁剪平面距离 和 远裁剪平面距离
* Viewport Rect: This refers to the Camera.rect property, which defines the portion of the screen where a camera's view is drawn. By adjusting these values, you can control where a camera renders on the screen and how much of the screen it occupies. This config is commonly used for split-screen effects.
* Depth: This property controls the order in which multiple cameras in a scene render their output. A camera with a lower depth value renders before a camera with a higher depth value. This is crucial for achieving effects like picture-in-picture, UI overlays, or rendering specific layers with different cameras. If multiple cameras have the same depth value, their rendering order is determined by their order in the scene hierarchy.

## 4.1.2 正交投影

![](p7.png "正交投影参数")

## 4.2 投影矩阵的目的

前面已经讨论过了裁剪的必要性 —— 进行渲染提出视野范围外的图元；这里需要讨论的是，为何不在视锥体裁剪，而是要先变换到裁剪空间再进行裁剪？

《Unity Shader 入门精要》中做了很清楚的解释：“直接在视锥体定义的空间进行裁剪，对于透视投影的视锥体想要判断一个顶点是否在这个空间内是十分麻烦的，我们需要一种更加通用、整洁方便的方式进行裁剪，因此就需要一个投影矩阵将顶点转换到一个裁剪空间(Clip space)中。”

从观察空间到裁剪空间的变换叫做投影变换。虽然叫做投影变换，但是投影变换并没有进行真正的投影。

## 4.2.1 为真正的投影做准备

真正的投影可以理解成空间的降维，4d -> 3d，3d -> 2d，真正的投影发生在屏幕映射过程中对顶点进行齐次除法后获得其二维坐标这一步，而投影矩阵只是进行了坐标空间转换，并没有实实在在地进行投影这一操作。
齐次（裁剪）空间实质上是一个四维空间，变换到齐次空间的顶点之间仍然是线性相关的，可以使用线性插值。（此时没有除以W变成3D坐标，是齐次坐标）

## 4.2.2 对x、y、z进行缩放

投影矩阵虽然叫做投影矩阵，但并没有真正进行投影，而是为投影做准备。经过投影矩阵的缩放后，我们可以直接使用w分量作为范围值，只有x，y，z分量都位于这个范围内的顶点才认为是在裁剪空间内。并且w分量在真正的投影时也会用到。

---

# 标准化设备坐标(Normalized Device Coordinate)

在齐次裁剪空间的基础上进行透视除法(Perspective division)或称齐次除法(Homogeneous division)，得到的坐标叫做NDC空间坐标。

* 裁剪空间是顶点乘以MVP矩阵之后所在的空间，**Vertex Shader的输出就是在裁剪空间上**（划重点）。
* 接着由GPU自己做**透视除法**，将顶点转移到标准化设备坐标(NDC)。

![](p8.png "透视除法")

---

# 5. 屏幕空间(Screen Space)

完成了裁剪工作，下一步就是进行真正的投影了，将视锥体投影到屏幕空间中，这一步会得到真正的像素位置，而不是虚拟的三维坐标。
这一环节可以理解为做了以下两步：

### 5.1 齐次除法

首先进行标准齐次除法（homogeneous division），也被称为透视除法（perspective division），其实就是x、y、z分别除以w，经过齐次除法后的裁剪空间会变成一个单位立方体，这个立方体空间里的坐标叫做归一化的设备坐标（也就是之前提到的NDC）。因此，也可以说齐次除法是做了空间裁剪坐标到NDC坐标的转换操作。

### 5.2 屏幕映射（渲染管线中几何阶段的一步）

这里就顺利的跟之前的渲染管线GPU负责的几何阶段部分联系在一起了。在获得了NDC立方体后，接下来就是根据变换后的x、y坐标映射输出窗口对应的像素坐标，本质就是个缩放的过程。

* 虽然屏幕是2d空间，但z分量此时并没有被抛弃，会被储存起来（深度缓存或者其他的储存格式）

我们前面说到Vertex Shader的输出在Clip Space，接着GPU会做透视除法变到NDC。这之后GPU还有一步，应用视口变换(Viewport Transformation)，转换到屏幕空间，输入给Fragment Shader：

**(Vertex Shader) => Clip Space => (透视除法) => NDC => (视口变换) => Window Space => (Fragment Shader)**

![](p11.png "总流程")

---

# Credits:

MVP矩阵：https://blog.csdn.net/qq_41835314/article/details/126851074

---