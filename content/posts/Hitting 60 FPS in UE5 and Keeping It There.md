---
title: "Hitting 60 FPS in UE5：60 FPS Engineering Checklist"
date: 2026-08-05
tags:
  - Unreal Engine
  - UE5
  - Performance
  - Optimization
  - Rendering
  - Mobile
categories:
  - Unreal Engine
summary: "结合 Epic GDC《Hitting 60 FPS in UE5 and Keeping It There》，总结商业级 UE5 项目实现稳定 60 FPS 的整体工程体系。"
---

<!--more-->

# Hitting 60 FPS in UE5

Epic 在 GDC 分享的 **《Hitting 60 FPS in UE5 and Keeping It There》** 并不是一场 Renderer 优化课程。

它真正想表达的是：

> **60 FPS 不是 Renderer 做出来的，而是整个项目从 Day-1 就共同设计出来的。**

也就是说：

```text
Stable 60 FPS

≠

Renderer Optimization

=

Whole Project Engineering
```

整个 UE5 项目，都应该围绕：

> **Budget（预算）**

进行设计。

---

# UE5 60 FPS Engineering Checklist

Epic 基本把一个商业项目拆成下面几个模块。

| 系统 | 重点关注 | 核心关键词 |
|------|---------|-----------|
| 🌍 World Building | 世界是否足够高效 | PCG、World Partition、HLOD、Async Budget |
| 🎮 Gameplay | Gameplay 是否可扩展 | Blueprint、C++、Modular Gameplay、Movement |
| 🤖 AI | AI 是否能规模化 | State Tree、Navigation、Mass |
| 🕺 Animation | 动画是否可预算化 | Parallel Animation、Contextual Animation、Modular Animation |
| 💥 Simulation | 模拟是否可控制 | Destruction、Physics、Trace Channel |
| 🎨 Rendering | Renderer 是否有预算 | Geometry、Lighting、Shadow、Upscaling |
| 🌤 Rendering Details | Renderer 是否足够轻 | Texture Streaming、Atmosphere、Post Process |
| ☀ Time of Day | 光照是否预算化 | Dynamic Lighting、Sky、Exposure |
| ⚡ PSO | Shader 是否不卡顿 | PSO Bundle、Precache |
| 🖥 UI | UI 是否不会拖慢 GPU | MVVM、Invalidation |
| ✨ Niagara | 特效是否预算化 | Lightweight Emitters、Effect Type |
| ⚙ Scalability | 是否支持不同设备 | Device Profiles、Quality Levels |

---

# 一、World Building

世界设计决定了：

> Renderer 有没有机会快。

Epic 推荐：

| 技术 | 目的 |
|------|------|
| PCG | 自动生成内容 |
| World Partition | 世界 Streaming |
| HLOD | 合并远景 |
| Async Budget | Streaming Budget |

重点思想：

> 世界越大，

越需要：

> Streaming。

而不是：

> 一次加载全部。

---

# 二、Gameplay

Gameplay 也是性能。

Epic 建议：

| 技术 | 目的 |
|------|------|
| Blueprint | 快速开发 |
| C++ | 高性能 |
| Modular Gameplay | 模块化 |
| Character Movement | 控制 Tick |

重点：

不要：

```text
所有东西：

Every Tick
```

---

# 三、Game AI

现代 UE5：

推荐：

| 技术 | 用途 |
|------|------|
| State Tree | 新一代 AI State Machine |
| Navigation | 导航 |
| Mass Framework | 大规模 NPC |

Mass：

就是：

为了：

```text
1000 NPC
```

而设计。

---

# 四、Animation

Epic 提出的几个问题：

```text
In-engine?

Contextual?

Parallel?

Modularity?
```

其实对应：

Animation：

是不是：

> 可预算。

重点：

- Parallel Animation
- Animation Budget Allocator
- Contextual Animation
- Modular Character

---

# 五、Simulation

包括：

- Physics
- Chaos
- Destruction

重点：

```text
复杂度

+

预算
```

例如：

Trace：

不要：

```text
ECC_ALL
```

应该：

合理：

Object Channel。

---

# 六、Rendering

Renderer：

不是：

画得越好。

而是：

预算：

是否：

合理。

Epic：

主要关注：

| 模块 | 技术 |
|------|------|
| Geometry | Nanite |
| Lighting | Lumen |
| Shadow | VSM |
| Upscaling | TSR |

Renderer：

始终：

围绕：

Budget。

---

# Nanite

Nanite：

并不是：

无限面数。

Epic：

特别强调：

| 检查项 |
|---------|
| LOD Fallback |
| Material Hierarchy |
| Modularity |

重点：

Nanite：

也需要：

Budget。

---

# Lumen

重点：

```text
HWRT

vs

SWRT
```

以及：

Reflection。

不是：

所有平台：

都：

HWRT。

---

# Shadow

Epic：

建议：

根据：

项目：

选择：

| 技术 |
|------|
| Virtual Shadow Maps |
| Megalights |

重点：

Shadow：

预算：

通常：

比：

Lighting：

更贵。

---

# Upscaling

Epic：

建议：

统一：

Upscaling Strategy。

包括：

| 技术 |
|------|
| TSR |
| TAA |
| FXAA |
| MSAA |
| DLSS |
| FSR |

不要：

项目：

里面：

多个：

混用。

---

# Rendering Considerations

除了：

Renderer。

还有：

很多：

隐藏：

成本。

---

## Texture Streaming

重点：

| 技术 |
|------|
| Virtual Texture |
| Pool Size |
| Runtime Streaming |
| Responsiveness |

---

## Atmospherics

包括：

- Fog
- Clouds
- Sky Scattering

这些：

GPU：

成本：

非常高。

---

## Post Processing

重点：

- Color Grading
- Bloom
- Resolution
- Utilization

---

# Time Of Day

Epic：

建议：

Time Of Day：

不要：

无限：

实时。

而是：

预算。

例如：

Sky Update：

可以：

降频。

---

# PSO

Epic：

今年：

最大的建议：

就是：

PSO。

包括：

| 技术 |
|------|
| Bundling |
| Precache |
| Hybrid |

目标：

避免：

Shader Hitch。

---

# UI

UI：

不要：

每帧：

重新：

Layout。

推荐：

MVVM：

```text
Model

↓

ViewModel

↓

View
```

以及：

Invalidation。

---

# Niagara

重点：

不是：

Emitter。

而是：

Budget。

Epic：

推荐：

| 技术 |
|------|
| Systems as a Service |
| Lightweight Emitters |
| Effect Type |

---

# Scalability

Epic：

最后：

强调：

Scalability。

任何：

Feature。

都：

应该：

支持：

```text
Low

Medium

High

Epic
```

而不是：

固定：

一种：

配置。

---

# View Modes（性能分析）

Epic 推荐：

开发阶段经常切换 View Modes。

| View Mode | 用途 |
|-----------|------|
| Instance Overlap | 检查实例重叠 |
| Lumen Performance | Lumen 开销 |
| Nanite Overdraw | Nanite Over绘制 |
| WPO Enabled | World Position Offset |
| Light Complexity | 光照复杂度 |
| SMRT Visualization | Shadow Map Ray Tracing |
| Stack Count | 材质层数 |
| Material Count | 材质数量 |

这些 View Modes 能快速发现：

- 哪些区域实例过密
- 哪些区域材质过于复杂
- 哪些区域光照成本过高
- 哪些 Mesh 存在大量 WPO 或 Nanite Overdraw

---

# 60 FPS 的真正含义

Epic 想表达的并不是：

> Renderer 如何跑到 60 FPS。

而是：

```text
整个项目

↓

共同维护

↓

Performance Budget

↓

持续保持

↓

60 FPS
```

Renderer、

Gameplay、

Animation、

AI、

Streaming、

PSO、

UI、

Niagara、

Scalability，

每一个模块，

都需要：

遵守：

自己的：

Budget。

---

# 总结

Epic 在《Hitting 60 FPS in UE5 and Keeping It There》中传递的核心思想可以总结为一句话：

> **60 FPS 不是 Renderer 的目标，而是整个项目的工程目标（Engineering Goal）。**

每个系统都需要围绕预算（Budget）设计：

- 世界需要 Streaming Budget。
- Gameplay 需要 Tick Budget。
- Animation 需要 Animation Budget。
- Renderer 需要 GPU Budget。
- Shader 需要 PSO Budget。
- UI 需要 Layout Budget。
- Niagara 需要 FX Budget。

最终，通过各系统共同遵守预算，才能真正实现并持续保持稳定的 60 FPS。