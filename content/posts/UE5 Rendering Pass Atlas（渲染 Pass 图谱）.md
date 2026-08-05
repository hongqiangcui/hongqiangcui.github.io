---
title: "UE5 Rendering Pass Atlas（渲染 Pass 图谱）"
date: 2026-08-05
tags:
  - Unreal Engine
  - UE5
  - Rendering
  - Renderer
  - RDG
  - GPU
categories:
  - Unreal Engine
summary: "完整梳理 Unreal Engine 5 Renderer 中各类 Render Pass 的职责、输入输出以及所属模块，建立 Renderer 全景图。"
---

<!--more-->

# UE5 Rendering Pass Atlas（渲染 Pass 图谱）

很多开发者第一次打开：

- ProfileGPU
- RenderDoc
- PIX
- RDG Viewer

都会看到几十甚至上百个 Pass。

实际上，这些 Pass 并不是杂乱无章的。

按照 Renderer 的职责，可以整理成下面这张图。

```text
Scene Preparation
        │
        ▼
Visibility & Culling
        │
        ▼
Shadow
        │
        ▼
Depth
        │
        ▼
Decal
        │
        ▼
Geometry
        │
        ▼
Lighting
        │
        ▼
Ray Tracing（Optional）
        │
        ▼
Composition
        │
        ▼
Post Processing
        │
        ▼
UI
        │
        ▼
Present
```

---

# Rendering Pass Atlas

| Pass Family | 常见 Pass | 输入 | 输出 | 是否必经 |
|-------------|-----------|------|------|----------|
| Scene Preparation | GPU Scene、Primitive Update、View Setup | World | Scene | ✅ |
| Visibility | Frustum、HZB、Occlusion、Instance Culling | Scene | Visible Primitive | ✅ |
| Shadow | Shadow Depth、VSM、CSM、Projection | Primitive | Shadow Atlas | △ |
| Depth | Depth PrePass、CustomDepth、Stencil | Primitive | Depth Buffer | △ |
| Decal | DBuffer、Deferred Decal | GBuffer | Modified GBuffer | △ |
| Geometry | BasePass、Nanite、Velocity | Primitive | GBuffer | ✅ |
| Lighting | Deferred、Lumen、Reflection、SSAO | GBuffer | SceneColor | ✅ |
| Ray Tracing | HWRT、SWRT、PathTracing | BVH | Lighting | △ |
| Composition | Sky、Cloud、Fog、Water、Hair、Niagara、Translucency | SceneColor | SceneColor | △ |
| Post Processing | TSR、TAA、Bloom、DoF、Tonemap | SceneColor | FinalColor | ✅ |
| UI | Slate、UMG、Debug | FinalColor | Backbuffer | △ |
| Output | Resolve、Present | Backbuffer | Monitor | ✅ |

> **说明**
>
> - ✅：绝大多数 Frame 都会执行。
> - △：取决于项目配置、平台和渲染特性。

---

# 一、Scene Preparation

## 目标

准备本帧所有渲染数据。

## 典型 Pass

| Pass | 作用 |
|------|------|
| Primitive Update | 更新 Primitive 数据 |
| GPU Scene Upload | 上传 GPU Scene |
| View Setup | 创建 View Family |
| Uniform Buffer | 更新 View 参数 |
| Instance Upload | 上传实例数据 |

输入：

```text
Game World
```

输出：

```text
Render Scene
```

---

# 二、Visibility & Culling

回答：

> **哪些对象需要参与渲染？**

## 包括

| Pass |
|------|
| PrimitiveOctree |
| Frustum Culling |
| Distance Culling |
| HLOD Selection |
| HZB |
| Occlusion Query |
| GPU Scene Visibility |
| Instance Culling |

输出：

```text
Visible Primitive List
```

---

# 三、Shadow

负责所有阴影。

## 包括

| Pass |
|------|
| Shadow Depth |
| Virtual Shadow Maps |
| Cascaded Shadow Maps |
| Per Object Shadow |
| Shadow Projection |
| Cached Shadow |

输出：

```text
Shadow Atlas
```

---

# 四、Depth

建立：

Depth Buffer。

## 包括

| Pass |
|------|
| Depth PrePass |
| Early-Z |
| CustomDepth |
| CustomStencil |

输出：

```text
Depth Buffer
```

---

# 五、Decal（贴花）

贴花负责修改 GBuffer。

## DBuffer Decal

流程：

```text
Depth

↓

DBuffer

↓

BasePass
```

修改：

- BaseColor
- Normal
- Roughness

---

## Deferred Decal

流程：

```text
BasePass

↓

Deferred Decal
```

主要：

- Emissive
- Translucent Decal

---

# 六、Geometry

真正开始 Rasterization。

## 包括

| Pass |
|------|
| BasePass |
| Nanite Raster |
| Velocity |
| DBuffer Resolve |
| Mesh Draw Pass |

输出：

```text
Depth

Normal

BaseColor

Roughness

Metallic

Velocity
```

也就是：

```text
GBuffer
```

---

# 七、Lighting

负责所有光照计算。

## 包括

| Pass |
|------|
| Deferred Lighting |
| Lumen GI |
| Lumen Reflection |
| Reflection Environment |
| SSAO |
| Skylight |
| SSGI |

输出：

```text
Scene Color
```

---

# 八、Ray Tracing

开启 RT 时：

包括：

| Pass |
|------|
| Hardware RT |
| Software RT |
| Lumen HWRT |
| RT Shadow |
| RT Reflection |
| Path Tracing |

不是：

所有平台：

都会执行。

---

# 九、Composition

开始：

叠加：

各种效果。

## 包括

| Pass |
|------|
| Sky Atmosphere |
| Exponential Height Fog |
| Volumetric Fog |
| Volumetric Cloud |
| Water |
| Hair Strands |
| Niagara |
| Translucency |
| Subsurface |

输出：

```text
Scene Color
```

---

# 十、Post Processing

数量最多。

## 包括

| Pass |
|------|
| TSR |
| TAA |
| FXAA |
| Motion Blur |
| Depth of Field |
| Bloom |
| Lens Flare |
| Chromatic Aberration |
| Auto Exposure |
| Tonemap |
| Color Grading |
| Vignette |
| Film Grain |
| Upscale |

输出：

```text
Final Color
```

---

# 十一、UI

最后：

绘制：

| Pass |
|------|
| Slate |
| UMG |
| Debug Overlay |
| Profiler |

输出：

```text
Backbuffer
```

---

# 十二、Output

最终：

包括：

| Pass |
|------|
| Resolve |
| Present |
| SwapChain |

输出：

```text
Monitor
```

---

# Pass Family 与 UE5 模块对应关系

| UE5 模块 | 主要 Pass |
|-----------|-----------|
| World Partition | Scene Preparation |
| GPU Scene | Scene + Visibility |
| PrimitiveOctree | Visibility |
| HLOD | Visibility |
| Nanite | Geometry |
| Mesh Draw Pipeline | Geometry |
| Virtual Shadow Maps | Shadow |
| Lumen | Lighting |
| Deferred Renderer | Lighting |
| SkyAtmosphere | Composition |
| Niagara | Composition |
| Water | Composition |
| Hair Strands | Composition |
| TSR | Post Processing |
| RDG | 调度所有 Pass |
| Slate | UI |
| RHI | Output |

---

# GPU 时间主要花在哪里？

典型 Deferred Renderer：

| Pass Family | GPU 占比 |
|--------------|---------|
| Visibility | ★☆☆☆☆ |
| Shadow | ★★★★★ |
| Geometry | ★★★★★ |
| Lighting | ★★★★★ |
| Ray Tracing | ★★★★★（开启时） |
| Composition | ★★★☆☆ |
| Post Processing | ★★★☆☆ |
| UI | ★☆☆☆☆ |
| Output | ★☆☆☆☆ |

通常：

- Shadow
- Geometry
- Lighting

三者占据 GPU 时间的大部分。

---

# 学习路线建议

建议按下面顺序阅读源码：

```text
Scene

↓

Visibility

↓

GPU Scene

↓

Mesh Draw Pipeline

↓

BasePass

↓

Deferred Lighting

↓

Lumen

↓

Shadow

↓

Translucency

↓

Post Processing

↓

RDG

↓

RHI
```

不要一开始就研究 Lumen 或 Nanite，而是先理解整个 Pass Atlas，再逐步深入各个模块。

---

# 总结

UE5 Renderer 中数量众多的 Render Pass，本质上可以归纳为 12 个功能域：

1. Scene Preparation
2. Visibility & Culling
3. Shadow
4. Depth
5. Decal
6. Geometry
7. Lighting
8. Ray Tracing
9. Composition
10. Post Processing
11. UI
12. Output

理解这些 Pass Family 后，再阅读 RDG、RenderDoc 或 ProfileGPU，就能快速定位每个 Pass 的职责、输入输出以及性能开销。

> **一句话总结：**
>
> **Render Pass 是执行单元，Pass Family 是功能模块，而 RDG 则负责组织它们之间的依赖关系。**