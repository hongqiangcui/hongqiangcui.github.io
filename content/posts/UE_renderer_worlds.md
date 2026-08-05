---
title: "理解 UE5 Renderer：两个 World、渲染职责与源码学习路线"
date: 2026-08-05
tags:
  - Unreal Engine
  - UE5
  - Rendering
  - Renderer
  - World Partition
  - GPU Scene
  - Mobile
categories:
  - Unreal Engine
summary: "从 Game World 与 Render World 的角度理解 UE5 Renderer，并总结商业级 Mobile 项目的优化重点及推荐源码学习路线。"
---

<!--more-->

# 理解 UE5 Renderer：两个 World

学习 Unreal Engine Renderer，我认为最重要的第一件事，就是建立 **"UE5 有两个 World"** 的概念。

很多初学 Renderer 的同学容易把 Gameplay 和 Rendering 混在一起。

实际上，UE 从架构设计上就把它们完全解耦。

```text
                UE5

      ┌──────────────────────┐
      │      Game World      │
      └──────────┬───────────┘
                 │
          Synchronization
                 │
                 ▼
      ┌──────────────────────┐
      │     Render World     │
      └──────────────────────┘
```

Game World 描述的是：

> **世界里有什么。**

Render World 描述的是：

> **GPU 应该画什么。**

---

# Game World

Game World 属于 Gameplay。

这里保存的是：

```text
Actor

Component

Transform

Collision

Animation

Physics

AI

Gameplay
```

这里的数据主要运行在：

```text
Game Thread
```

例如：

```cpp
AActor

↓

USceneComponent

↓

UStaticMeshComponent
```

所有 Gameplay 都发生在这里。

例如：

- 玩家移动
- AI 行为
- Physics
- Collision
- Animation Update

这些都不会直接操作 GPU。

---

# Render World

Game Thread 每一帧都会把需要渲染的数据同步到：

```text
Render World
```

这一层才是真正属于 Renderer。

例如：

```text
PrimitiveSceneProxy

GPU Scene

Mesh Draw Command

Instance Culling

Renderer

RHI
```

它们主要运行在：

```text
Render Thread
```

Renderer 根本不知道：

```text
Player

Enemy

NPC
```

它只知道：

```text
Mesh

Material

GPU Buffer
```

---

# 两个 World 如何连接？

中间最重要的对象就是：

```cpp
FPrimitiveSceneProxy
```

完整的数据流如下：

```text
Game Thread

Actor
 │
 ▼
UPrimitiveComponent
 │
 │ Synchronization
 ▼
FPrimitiveSceneProxy
 │
 ▼
Render Thread
 │
 ▼
GPU Scene
 │
 ▼
Mesh Draw Command
 │
 ▼
GPU
```

可以认为：

> **PrimitiveSceneProxy 就是两个 World 之间的桥梁。**

---

# UE Worlds：Renderer 的完整工作流

站在 Renderer 的角度来看，一个世界最终会经过下面这条流水线：

```text
World Partition
        │
        ▼
Scene Visibility
        │
        ▼
PrimitiveOctree
        │
        ▼
GPU Scene
        │
        ▼
Mesh Draw Command
        │
        ▼
Instance Culling
        │
        ▼
RHI
        │
        ▼
GPU
```

每一层都有明确的职责。

| 模块 | 主要职责 |
|------|----------|
| World Partition | 管理世界分块、Streaming 与加载 |
| Scene Visibility | 判断当前 Camera 可见对象 |
| PrimitiveOctree | 空间索引，加速查询与剔除 |
| GPU Scene | 将场景数据上传 GPU，统一管理实例数据 |
| Mesh Draw Command | 组织 Draw Call，减少 CPU 提交开销 |
| Instance Culling | GPU 或 CPU 剔除不可见实例 |
| RHI | 将渲染命令提交到 DirectX、Vulkan、Metal 等图形 API |

这条流水线就是 **Render World** 的核心工作流程。

---

# UE 已经帮你解决了什么？

UE Renderer 已经替开发者解决了一个非常复杂的问题：

> **怎么渲染（How to Render）**

例如：

- Shader 编译
- Draw Call 管理
- GPU Resource
- Render Pass
- PSO
- RDG
- RHI

这些都是 Engine Layer 的职责。

大多数情况下，开发者无需自己实现。

---

# 商业项目真正投入优化的地方

对于商业级项目（尤其是 Mobile 项目），额外投入的大量优化工作，更多是在回答另外三个问题：

```text
什么该渲染？

什么时候渲染？

渲染到什么程度？
```

也就是：

- **What should be rendered?**
- **When should it be rendered?**
- **To what level should it be rendered?**

例如：

- World Partition Streaming
- HLOD
- LOD
- Instance Culling
- Occlusion Culling
- Texture Streaming
- Mesh Streaming
- Visibility
- Foliage 合批
- HISM
- Shadow Distance
- Nanite Cluster Streaming

这些决定了 Renderer 是否高效，而不是 Renderer 是否能够工作。

换句话说：

> **UE 默认解决的是"如何画"，商业项目更多关注的是"画什么、何时画、画多少"。**

---

# 推荐的源码学习路线

如果目标是：

- 理解 UE Renderer
- 学习大世界
- 掌握 Mobile 优化
- 阅读 Engine 源码

相比直接研究 Lyra，我更推荐下面这条路线。

按收益排序：

```text
★★★★★ StaticMesh
        │
        ▼
★★★★★ HISM
        │
        ▼
★★★★★ Landscape
        │
        ▼
★★★★★ Foliage
        │
        ▼
★★★★★ World Partition
        │
        ▼
★★★★ GPU Scene
        │
        ▼
★★★★ Renderer
        │
        ▼
★★★★ Nanite
```

---

# 为什么推荐这条路线？

## ① Static Mesh（★★★★★）

所有 Renderer 的起点。

建议重点理解：

- UStaticMesh
- FStaticMeshRenderData
- FStaticMeshLODResources
- Vertex Buffer
- Index Buffer

这是后续所有渲染对象的基础。

---

## ② HISM（★★★★★）

商业项目最重要的优化技术之一。

重点包括：

- Instancing
- Draw Call 合并
- GPU Instance Buffer
- Instance Culling

几乎所有植被和重复物体都会使用。

---

## ③ Landscape（★★★★★）

学习：

- Landscape Component
- Section
- LOD
- Streaming Proxy

理解 UE 如何管理超大地形。

---

## ④ Foliage（★★★★★）

这是 HISM 的实际应用。

重点包括：

- Foliage Actor
- ISM/HISM
- Cull Distance
- Cluster
- Instance Buffer

商业项目中出现频率极高。

---

## ⑤ World Partition（★★★★★）

理解 UE5 大世界的关键。

学习：

- Runtime Hash
- Streaming Cell
- Data Layer
- Spatial Loading

这是 UE5 World Streaming 的核心。

---

## ⑥ GPU Scene（★★★★☆）

这是 UE5 Renderer 最重要的新架构之一。

学习：

- GPU Primitive Buffer
- GPU Instance Data
- Primitive Upload
- Scene Data 管理

---

## ⑦ Renderer（★★★★☆）

最后再深入：

- Scene Visibility
- Mesh Pass
- Render Graph（RDG）
- Shadow
- Lighting
- Deferred Renderer

此时再阅读 Renderer 源码会轻松很多。

---

## ⑧ Nanite（★★★★☆）

最后研究 Nanite。

重点包括：

- Cluster
- Page Streaming
- Visibility Buffer
- Software Rasterizer

Nanite 建立在前面所有知识之上，放到最后学习效率最高。

---

# 这条路线覆盖了哪些核心能力？

按照以上顺序学习，几乎可以覆盖 UE5 Renderer 中最重要的内容：

- 大世界（World Partition）
- Mobile 优化
- Streaming
- LOD 系统
- GPU 提交
- CPU 提交
- HISM / Instancing
- Landscape
- Foliage
- Renderer Pipeline
- GPU Scene
- Nanite

它既符合 UE5 的整体架构，也贴近商业项目中的实际需求。

---

# 总结

理解 UE Renderer，可以先建立两个核心认知：

## 一、UE5 有两个 World

- **Game World**：负责 Gameplay，由 Game Thread 更新。
- **Render World**：负责 Rendering，由 Render Thread 驱动。

二者通过 `FPrimitiveSceneProxy` 完成数据同步。

---

## 二、Renderer 的核心工作流

```text
World Partition
        ↓
Scene Visibility
        ↓
PrimitiveOctree
        ↓
GPU Scene
        ↓
Mesh Draw Command
        ↓
Instance Culling
        ↓
RHI
        ↓
GPU
```

这是 Render World 从"世界数据"到"GPU 绘制"的核心流水线。

---

## 三、商业项目真正优化的重点

UE 已经帮开发者解决了：

> **"怎么渲染（How to Render）"**

而商业级项目真正需要投入精力的是：

- **什么该渲染（What）**
- **什么时候渲染（When）**
- **渲染到什么程度（How Much）**

因此，大世界管理、Streaming、LOD、Visibility、Instance Culling 等技术，往往比单纯研究 Renderer 的实现更能直接提升项目性能，也是学习 UE5 渲染源码时最具性价比的方向。