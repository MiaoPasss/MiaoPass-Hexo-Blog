---
title: Bleeding 渗色
date: 2026-04-01 16:59:48
categories: Graphics
tags: 
- Rendering pipeline
---

# Bleeding

纹理渗色/纹理出血，这是一种图像处理或显示技术问题，指一种颜色“溢出”到相邻像素的现象，导致颜色边界不清晰、产生伪影或颜色混杂。

![](p1.png "2D Sprite Renderer Bleeding")

Texture bleeding is when pixels from outside a sprite's intended boundary leak into the visible edge of the sprite.

Here's why it happens. When the GPU samples a texture, it rarely samples at an exact pixel center. 
Bilinear filtering works by taking the 4 nearest pixels and blending them together based on how close the sample point is to each one. 
At the very edge of a sprite, one of those 4 neighbors is outside the sprite — it could be a transparent black pixel, a pixel from another sprite in an atlas, or garbage data. That outside color gets mixed into your edge pixel, producing a fringe（边缘）.

For example:
```
[ edge pixel: orange ][ outside: transparent black (0,0,0,0) ]
                      ^
              bilinear samples here → blends orange + black = dark muddy fringe
```

This is exactly what you saw — the logo's colored edge pixels were bleeding into the transparent area around it, and because the shader was Opaque, those blended semi-transparent edge pixels were written as solid color instead of being discarded.

There are three common sources of bleeding:
* Wrap Mode: Repeat — the GPU wraps around and samples the opposite edge of the texture
* Atlas packing with no padding — neighboring sprites bleed into each other
* Opaque surface type with a transparent sprite — alpha is ignored so blended edge pixels render solid

# Opaque vs Transparent

The GPU renders geometry in two stages: rasterization and blending. 
When Surface Type is Opaque, the blending stage is skipped entirely — the GPU writes every pixel the fragment shader touches directly to the framebuffer, alpha value be damned. 
Your shader was correctly computing alpha from the texture, but the pipeline threw it away before it ever reached the screen.

The "noise" you saw was bilinear filtering. At the sprite's edges, the sampler blends the last row/column of real pixels with whatever is just outside the sprite boundary (either transparent black rgba(0,0,0,0) or neighboring texture data). 
With opaque mode, those semi-transparent edge pixels get written as fully solid, so you see that fringe of mixed color.

Use `Opaque` when:
The object has no see-through areas at all (terrain, walls, characters without cutouts)
Performance matters — opaque objects are rendered in a single front-to-back pass and benefit from early-z culling, which skips fragment shading on occluded pixels entirely

Use `Transparent` when:
The object has partial or full transparency (UI sprites, glass, particles, logos with alpha)
You need the blending stage to mix the fragment color with what's already in the framebuffer
The cost of transparent is real — transparent objects can't use early-z, must be sorted back-to-front, and can cause overdraw issues. So don't default everything to transparent.

## Blend Mode (within Transparent)

```
| Mode | Formula | Use case |
| --- | --- | --- |
| Alpha     |   src.rgb * src.a + dst.rgb * (1 - src.a) | Standard transparency, sprites, UI
| Additive  |   src.rgb * src.a + dst.rgb   |   Glows, fire, particles that should brighten
| Multiply  |	src.rgb * dst.rgb   |   Darkening effects, shadows, tinted overlays
| Premultiply |	src.rgb + dst.rgb * (1 - src.a) |   Textures with premultiplied alpha baked in
```

# The Z-Buffer

The GPU maintains a depth buffer (z-buffer) alongside the color buffer. Every pixel stores the depth of the closest thing rendered to it so far. 
When a new fragment arrives, the GPU compares its depth against what's already in the buffer — if it's further away, it gets discarded.

## Front-to-Back Pass

For opaque objects, Unity sorts them front-to-back before drawing. So the closest objects get drawn first.
```
Camera → [Chair] → [Table] → [Wall behind]
           drawn     drawn      drawn
           first     second     third
```
This matters because of early-z.

## Early-Z Culling
Once the chair is drawn and its depth is written to the z-buffer, when the wall fragment arrives at the same pixel, the GPU checks:
```
wall depth (5.0) > z-buffer value (1.2) → discard, skip fragment shader
```
It never runs the fragment shader for the wall pixel at all. This is early-z culling — rejecting fragments before the expensive shading work happens.
The savings are significant. Fragment shaders do lighting, texture sampling, math — all skipped for occluded pixels.

## Why transparent objects break this
Transparent objects need to blend with what's behind them, so they must be drawn last, back-to-front:
```
Camera → [Wall] → [Table] → [Glass in front]
```

The glass needs the wall already in the framebuffer to blend against it. This means:
* You can't sort front-to-back
* Early-z culling doesn't apply
* Every transparent fragment runs the full fragment shader regardless of what's in front of it

That's the real performance cost of transparency — not the blending itself, but losing early-z and the overdraw that comes with back-to-front rendering.