---
title: "UE5 优化分层模型（Optimization Layer Model）"
date: 2026-08-05
tags:
  - Unreal Engine
  - UE5
  - Optimization
  - Profiling
  - Performance
categories:
  - Unreal Engine
summary: "总结 Unreal Engine 5 的性能优化分层模型，从 CPU、GPU、Memory、Streaming、Shader 五个维度建立系统性的优化思路。"
---

<!--more-->

# UE5 优化分层模型（Optimization Layer Model）

很多开发者学习 UE 优化时，会直接去研究：

- Nanite
- Lumen
- RDG
- GPU Scene
- HLOD

但实际上，这些只是**优化手段（Optimization Techniques）**。

真正做商业项目时，更重要的是先建立一套**性能分析（Profiling）分层模型**。

我习惯将 UE5 的优化划分为五个层级：

```text
Profiling
 ├── CPU
 ├── GPU
 ├── Memory
 ├── IO / Streaming
 └── Shader / PSO
```

当项目出现性能问题时，第一步不是优化，而是回答：

> **瓶颈到底属于哪一层？**

只有定位正确，后续的优化才有意义。

---

# 优化分层总览

| 层级 | 关注对象 | 常见问题 | 常用工具 | 常见优化方向 |
|------|----------|----------|----------|--------------|
| **CPU** | Gameplay、Tick、Animation、Physics | Game Thread 过高 | Unreal Insights、Stat Game、Stat Tick | Tick 优化、逻辑拆分、异步任务、减少 Actor |
| **GPU** | Rendering、Lumen、Nanite、Niagara | GPU 时间过长 | ProfileGPU、Stat GPU、RenderDoc | LOD、材质优化、阴影、光照、减少 Overdraw |
| **Memory / Asset** | Mesh、Texture、Animation | 内存占用过高 | MemReport、Stat Memory、LLM | Texture Streaming、压缩资源、减少驻留 |
| **IO / Streaming** | World Partition、Asset Loading | 加载卡顿、Streaming Hitch | Unreal Insights、Stat Streaming | World Partition、Async Loading、HLOD |
| **Shader / PSO** | Shader、Pipeline State | Shader Hitch、首次卡顿 | PSO Cache、Shader Compile、Insights | Shader Permutation、PSO Cache、预热 |

---

# 一、CPU（Gameplay）

CPU 层主要负责：

```text
Gameplay

AI

Physics

Animation

Tick

Blueprint

Task Graph
```

也就是：

```text
Game Thread
```

---

## 常见瓶颈

例如：

```text
Tick 太多

Actor 太多

Blueprint Tick

Animation Update

Physics

AI

Path Finding
```

商业项目中：

CPU 很多时候不是 Renderer，而是：

```text
Gameplay
```

---

## 常用工具

```text
Stat Game

Stat Tick

Stat Unit

Unreal Insights
```

---

## 常见优化方法

- 减少 Tick
- Tick Interval
- Event 驱动代替 Tick
- Async Task
- Object Pool
- Gameplay 拆分
- Mass Framework（大量实体）

---

# 二、GPU（Rendering）

GPU 层负责：

```text
Lumen

Nanite

Shadow

PostProcess

Niagara

Material

Lighting

Renderer
```

主要对应：

```text
Render Thread

GPU
```

---

## 常见瓶颈

例如：

```text
Lumen GI

Virtual Shadow Map

Nanite

Material

Translucency

Overdraw

Post Process

Niagara
```

---

## 常用工具

```text
ProfileGPU

Stat GPU

RenderDoc

PIX

Nsight
```

---

## 常见优化方法

- LOD
- HLOD
- Instance
- HISM
- Foliage
- Material Instance
- Shader Complexity
- Shadow Distance
- Niagara Budget

---

# 三、Memory / Asset

很多项目 FPS 正常。

但是：

```text
OOM

Memory Leak

Texture Pool Overflow
```

这些都属于：

Memory Layer。

---

## 关注对象

```text
Texture

Static Mesh

Skeletal Mesh

Animation

Audio

Virtual Texture
```

---

## 常见问题

例如：

```text
Texture Pool 满

大量资源常驻

动画过大

重复资源

Mesh 太大
```

---

## 常用工具

```text
MemReport

Stat Memory

Stat TextureGroup

LLM
```

---

## 常见优化方法

- Texture Streaming
- Virtual Texture
- Mesh LOD
- Animation Compression
- Asset Audit
- 减少常驻资源
- 合理设置 Streaming Pool

---

# 四、IO / Streaming

UE5 大世界项目：

很多性能问题其实来自：

```text
Streaming
```

而不是 GPU。

---

## 包括

```text
World Partition

Level Streaming

Async Loading

Asset Manager

Data Layer
```

---

## 常见问题

例如：

```text
Streaming Hitch

Level Loading

Cell Loading

IO Stall

Asset Blocking
```

---

## 常用工具

```text
Unreal Insights

Stat Streaming

IO Insights
```

---

## 常见优化方法

- World Partition
- Data Layer
- Async Loading
- HLOD
- Streaming Source
- Streaming Distance
- Asset Chunk

---

# 五、Shader / PSO

这一层主要解决：

```text
Shader Compile

PSO Compile

Shader Hitch
```

尤其是在：

DX12

Vulkan

Metal

都会出现：

首次进入场景：

```text
卡一下
```

这通常就是：

Shader Hitch。

---

## 常见问题

例如：

```text
Shader Compile

PSO Compile

Permutation Explosion

首次 Draw 卡顿
```

---

## 常用工具

```text
Shader Compile Worker

PSO Cache

Insights

Shader Stats
```

---

## 常见优化方法

- PSO Cache
- PSO Precache
- Shader Pipeline Cache
- 减少 Material Permutation
- Material Instance
- Shared Shader

---

# 五层之间的关系

可以理解成下面这样：

```text
                 Profiling

        ┌────────────────────────┐
        │        CPU             │
        │ Gameplay / Tick        │
        └────────────────────────┘
                    │
                    ▼
        ┌────────────────────────┐
        │        GPU             │
        │ Renderer / Lumen       │
        └────────────────────────┘
                    │
                    ▼
        ┌────────────────────────┐
        │   Memory / Asset       │
        │ Texture / Mesh         │
        └────────────────────────┘
                    │
                    ▼
        ┌────────────────────────┐
        │   IO / Streaming       │
        │ World Partition        │
        └────────────────────────┘
                    │
                    ▼
        ┌────────────────────────┐
        │    Shader / PSO        │
        │ Compile / Cache        │
        └────────────────────────┘
```

需要注意的是：

这些层并不是严格的上下游关系，而是**五个独立但相互影响的性能维度**。

例如：

- Texture Streaming 会影响 GPU Memory。
- World Partition 会影响 IO 与 CPU。
- Material 数量会影响 GPU 和 Shader。
- Actor 数量会同时影响 CPU 和 Renderer。

因此，定位问题时要避免只关注某一个指标。

---

# 商业项目中的优化思路

很多人一提到 UE5 优化，就会想到：

- Lumen
- Nanite
- GPU Scene

实际上，真正的商业项目通常遵循下面的流程：

```text
Profiling

↓

定位瓶颈

↓

确认属于哪一层

↓

选择对应优化方案

↓

验证优化结果
```

而不是：

```text
FPS 低

↓

开始改 Renderer
```

大多数情况下，这种做法效率并不高。

---

# 推荐的优化学习顺序

建议按照下面的顺序建立性能分析能力：

1. **CPU Profiling**
   - Tick
   - Gameplay
   - Animation
   - Physics

2. **GPU Profiling**
   - Renderer
   - Lumen
   - Nanite
   - Shadow
   - Niagara

3. **Memory**
   - Texture
   - Mesh
   - Streaming Pool
   - Virtual Texture

4. **Streaming**
   - World Partition
   - Async Loading
   - HLOD
   - Asset Manager

5. **Shader / PSO**
   - Shader Compile
   - PSO Cache
   - Pipeline Cache
   - Material Permutation

---

# 总结

UE5 的性能优化可以抽象为五个层级：

| 层级 | 核心问题 |
|------|----------|
| **CPU** | 游戏逻辑是否高效运行？ |
| **GPU** | 每一帧是否能高效完成渲染？ |
| **Memory** | 资源是否占用过多内存？ |
| **IO / Streaming** | 资源是否能及时加载和卸载？ |
| **Shader / PSO** | Shader 与 Pipeline 是否造成编译卡顿？ |

**一句话总结：**

> **优化不是先修改代码，而是先定位瓶颈。**
>
> 建立「CPU → GPU → Memory → IO → Shader」五层 Profiling 模型，比掌握任何单一优化技巧都更重要。