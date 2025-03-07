---
title: Unity摄像头参数
date: 2025-03-07 11:30:06
categories: Game Engine
tags:
- Unity
- Camera
---

# Unity 2022.3 Camera Component

<br>

![相机参数](image.png "Unity 2022.3.55f1")

## Clear Flags(清除标记):
A camera in Unity holds info about the displayed object’s colors and depths.
In this case, depth info means the game world distance from the displayed object to the camera.

When a camera renders an image it can get rid of all or just some of the old image information and then displays the new image on top of what’s left.
Depending on what clear flags you select, the camera gets rid of different info.

- The skybox clear flag means that when the camera renders a new frame, it clears everything from the old frame and it displays the new image on top of a skybox.
- The solid color clear flag means that when the camera renders a new frame, it clears everything from the old frame and it displays the new image on top of a solid color.
- The depth only clear flag means that when the camera renders a new frame, it clears only the depth information from the old frame and it keeps the color information on top of which displays the new frame. This means that the old frame can still show but it doesn’t have depth information, meaning the new objects to be displayed can’t be shown as intersected or obstructed by the old objects, because there’s no information about how far the old objects are(depth info was cleared). So the new objects will be on top of the old ones, mandatory.

## Background(背景): 
The color applied to the remaining screen after (1) all elements in view have been drawn and (2) there is no skybox.

## Culling Mask(剔除遮罩): 
Includes or omits layers of objects to be rendered by the Camera.

## Projection(投射方式): 
- Perspective(透视): Camera will render objects with perspective intact.
    - Field of view: The Camera’s view angle, measured in degrees along the axis specified in the FOV Axis drop-down.
- Orthographic(正交): Camera will render objects uniformly, with no sense of perspective. NOTE: Deferred rendering is not supported in Orthographic mode. Forward rendering is always used.
    - Size: The viewport(The user’s visible area of an app on their screen.) size of the Camera when set to Orthographic.

## Clipping Planes(剪裁平面): 
Distances from the camera to start and stop rendering.
- Near(近点): The closest point relative to the camera that drawing will occur.
- Far(远点): The furthest point relative to the camera that drawing will occur.

## Normalized Viewport Rect(标准视图矩形): 
Four values that indicate where on the screen this camera view will be drawn. Measured in Viewport Coordinates (values 0–1).
- X: The beginning horizontal position that the camera view will be drawn.
- Y: The beginning vertical position that the camera view will be drawn.
- W: Width of the camera output on the screen.
- H: Height of the camera output on the screen.
It’s easy to create a two-player split screen effect using Normalized Viewport Rectangle. After you have created your two cameras, change both camera’s H values to be 0.5 then set player one’s Y value to 0.5, and player two’s Y value to 0. This will make player one’s camera display from halfway up the screen to the top, and player two’s camera start at the bottom and stop halfway up the screen.

## Depth(相机深度): 
The camera’s position in the draw order. Cameras with a larger value will be drawn on top of cameras with a smaller value.

## Rendering Path(渲染路径): 
Options for defining what rendering methods will be used by the camera.

- Use Graphics Settings: This camera will use whichever Rendering Path is set in the Project Settings -> Player
- Forward(快速渲染): 摄像机将所有游戏对象将按每种材质一个通道的方式来渲染。对于实时光影来说，Forward的消耗比Deferred更高，但是Forward更加适合用于半烘焙半实时的项目。Forward解决了一个Deferred没能解决的问题：Deferred不能让Mixed模式的Directional Light将动态阴影投射在一个经过烘焙了的静态物体上。
- Deferred(延迟光照): 最大的特点是对于实时光影来说的性能消耗更低了，这个模式是最适合动态光影的。对于次时代PC或者主机游戏, 当然要选择这个。次时代游戏几乎不需要烘焙光照贴图了，全都使用实时阴影是很好的选择。通过阴影距离来控制性能消耗。而在Viking Village的场景中，由于整个场景全部使用了动态光源，Forward的Rendering方面的性能消耗要比Deferred高出一倍！ 因此在完全使用动态光源的项目中千万不能使用Forward。
- Vertex Lit(顶点光照): All objects rendered by this camera will be rendered as Vertex-Lit objects.

Rendering path is the technique that a render pipeline uses to render graphics. Choosing a different rendering path affects how lighting and shading are calculated. Some rendering paths are more suited to different platforms and hardware than others.

- ### Forward vs Deferred Rendering

Forward rendering does all of the work for rendering geometry up front when it gets submitted to the rendering pipeline. You submit your geometry and materials to the pipeline, and at some point the fragments(pixels) that represent your geometry are calculated and invokes a fragment shader to figure out the final "color" of the fragment based on where it is located on the screen (render target). This is implemented as logic in a pixel shader that does the expensive lighting and special effects calculations on a per-pixel basis.

The inefficiency with this approach is, when pixels get overwritten by geometry submitted later in the pipeline that appear in front of it. You did all of that expensive work for nothing. Enter deferred rendering:

Deferred rendering should really be called deferred shading because the geometry is (or can be) submitted to the pipeline very much in the same way as forward rendering. The difference is that the result is not actually a final color value for the final image. The pipeline is configured in a way such that instead of actually going through with the calculations, all of the information is stored in a G-buffer to do the calculations later. That way, this information can be overwritten multiple times, without ever having calculated the final color value until the last fragment's information is written. At the very end, the entire G-buffer is processed and all of the information it stored is used to calculate the final color value.

Last words:

Neither technique is really harder to learn. We're just coming from a forward rendering past. Once you have a good grasp of deferred rendering (and perhaps have a solid computer graphics background), you realize it's just another way to do things.

It's hard to say which technique is better. As pixel/fragment shaders get more GPU processing expensive, deferred shading becomes more effective. If early Z testing (https://docs.unity3d.com/6000.0/Documentation/Manual/SL-ZTest.html) can be employed or other effective culling techniques are used, deferred shading becomes less important. G-buffers also take up a lot of graphics memory.

## Target Texture(目标纹理): 
相机渲染不再显示在屏幕上，而是映射到纹理上。一般用于制作导航图或者画中画等效果。

Reference to a Render Texture that will contain the output of the Camera view. Setting this reference will disable this Camera’s capability to render to the screen. 
Render Texture is a special type of Texture that is created and updated at runtime. To use them, first create a new Render Texture and designate one of your Cameras to render into it. Then you can use the Render Texture in a Material just like a regular Texture.

## Occlusion Culling(遮挡剔除): 
Occlusion culling is a process which prevents Unity from performing rendering calculations for GameObjects that are completely hidden from view (occluded) by other GameObjects, for example if they are behind walls. (https://docs.unity.cn/Manual/OcclusionCulling.html)

## Allow HDR(渲染高动态色彩画面): 
Enables High Dynamic Range rendering for this camera.

## Allow MSAA(硬件抗锯齿): 
Enables multi sample antialiasing for this camera.

## Allow Dynamic Resolution(动态分辨率渲染): 
Enables Dynamic Resolution rendering for this camera.

## Target Display(目标显示器): 
A camera has up to 8 target display settings. The camera can be controlled to render to one of up to 8 monitors. This is supported only on PC, Mac and Linux. In Game View the chosen display in the Camera Inspector will be shown.

---
# Credits

Unity documentation: https://docs.unity.cn/Manual/class-Camera.html
Unity 摄像机参数介绍：https://blog.csdn.net/Scopperil/article/details/80440448

---