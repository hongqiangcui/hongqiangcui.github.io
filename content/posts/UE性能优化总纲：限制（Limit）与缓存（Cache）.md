---
title: "Unreal Engine 性能优化总纲：限制（Limit）与缓存（Cache）"
date: 2026-08-05
tags:
  - Unreal Engine
  - UE5
  - Optimization
  - Rendering
  - Mobile
categories:
  - Unreal Engine
summary: "总结 Unreal Engine 的性能优化思想，从空间域、时间域、分辨率域、光照域四个维度理解商业级项目的优化体系。"
---

<!--more-->

# Unreal Engine 性能优化总纲

很多人学习 UE 优化，会先学习：

- LOD
- HLOD
- Nanite
- World Partition
- GPU Scene
- FSR
- PSO

但这些其实只是各种优化技术。

如果从更高层来看，UE5 的性能优化几乎都围绕两个核心思想展开：

> **Limit（限制）**
>
> **Cache（缓存）**

一句话总结：

> **尽可能少算（Limit），尽可能不要重复算（Cache）。**

---

# UE5 优化四个维度

UE Renderer 几乎所有优化，都可以归纳到下面四个维度。

| 优化维度 | 主要目标 | 典型技术 |
|----------|----------|----------|
| **空间域（Spatial Domain）** | 少画、不画 | Cull、LOD、HLOD、HISM、World Partition、GPU Scene |
| **分辨率域（Resolution Domain）** | 少计算 Pixel | Dynamic Resolution、TSR、FSR、Screen Percentage |
| **时间域（Temporal Domain）** | 少更新 | TAA、TSR、Animation Budget、Tick Budget、多帧累积 |
| **光照域（Lighting Domain）** | 少计算 Lighting | Lightmap、Reflection Capture、Lumen Budget、Shadow Budget |

可以理解为：

```text
                Performance

        ┌──────── Spatial ────────┐
        │      少画（Cull）        │
        └──────────────────────────┘
                    │
        ┌────── Resolution ───────┐
        │     少算 Pixel          │
        └──────────────────────────┘
                    │
        ┌──────── Temporal ───────┐
        │     少更新              │
        └──────────────────────────┘
                    │
        ┌──────── Lighting ───────┐
        │     少算 Lighting       │
        └──────────────────────────┘
```

---

# 一、空间域（Spatial Domain）

空间域解决的问题：

> **哪些东西根本不用画？**

这是收益最高的一层优化。

## 常见技术

| 技术 | 作用 |
|------|------|
| Frustum Culling | 视锥剔除 |
| Occlusion Culling | 遮挡剔除 |
| Distance Culling | 距离剔除 |
| LOD | 模型降级 |
| HLOD | 合并远景 |
| HISM / ISM | 自动实例化 |
| Foliage | 植被批处理 |
| World Partition | 世界分块 |
| GPU Scene | GPU 场景管理 |
| Primitive Octree | 空间索引 |

一句话：

> **看不见的不画，远处画得更简单。**

---

# 二、分辨率域（Resolution Domain）

分辨率域解决：

> **每一个 Pixel 是否都值得计算？**

## 常见技术

| 技术 | 作用 |
|------|------|
| Dynamic Resolution | 动态分辨率 |
| TSR | Temporal Super Resolution |
| AMD FSR | FidelityFX Super Resolution |
| Mobile FSR | 移动端超分 |
| Screen Percentage | 屏幕百分比 |
| Secondary Resolution | 中间分辨率 |
| Upscale | 图像重建 |

一句话：

> **不是所有像素都必须按最终分辨率计算。**

---

# 三、时间域（Temporal Domain）

时间域解决：

> **是不是每一帧都必须重新计算？**

很多数据其实可以跨帧复用。

## 常见技术

| 技术 | 作用 |
|------|------|
| TSR | 时间域重建 |
| TAA | Temporal Anti-Aliasing |
| Animation Budget Allocator | 动画预算分配 |
| Tick Interval | Tick 降频 |
| Multi-frame Accumulation | 多帧累积 |
| Async Update | 异步更新 |
| Significance Manager | 重要性驱动更新 |

一句话：

> **能复用上一帧，就不要重新计算。**

---

# 四、光照域（Lighting Domain）

光照域解决：

> **是否必须实时计算光照？**

## 常见技术

| 技术 | 作用 |
|------|------|
| Lightmap | 光照烘焙 |
| Reflection Capture | 反射缓存 |
| Shadow Cache | 阴影缓存 |
| Lumen Budget | Lumen 预算 |
| Virtual Shadow Map | 阴影优化 |
| Skylight Cache | 环境光缓存 |

一句话：

> **静态光照尽量预计算，动态光照控制预算。**

---

# 商业项目常见优化技术

下面这些几乎覆盖了商业级 UE 项目的核心优化方案。

| 分类 | 技术 |
|------|------|
| Rendering | 前向渲染（Forward）、延迟渲染（Deferred） |
| Upscaling | AMD FSR、Mobile FSR、TSR |
| Rendering Budget | 动态分辨率、帧率调校 |
| Instancing | HISM、ISM、自动实例化 |
| Animation | Animation Budget Allocator |
| Material | Material Quality Level、Shader Quality |
| Streaming | World Partition、Texture Streaming |
| Shader | PSO Precache、Shader Pipeline Cache |
| Device | Device Profiles |
| Analysis | Unreal Insights、RenderDoc、MemReport |

---

# UE5 的核心优化手段

如果继续抽象，可以归纳为下面几类。

| 优化策略 | UE 典型技术 |
|-----------|------------|
| **预加载（Preload）** | Async Loading、PSO Precache、Asset Manager |
| **预计算（Precompute）** | Lightmap、Reflection Capture、HLOD |
| **缓存（Cache）** | GPU Scene、Shadow Cache、PSO Cache |
| **并行（Parallel）** | Task Graph、RDG、Async Compute |
| **跨帧复用（Temporal）** | TSR、TAA、History Buffer |
| **降频更新（Reduce Frequency）** | Tick Interval、Animation Budget |
| **预算限制（Budget）** | Dynamic Resolution、Material Quality、Lumen Budget |
| **空间裁剪（Cull）** | Frustum、Occlusion、Distance Culling |
| **实例化（Instance）** | ISM、HISM、Foliage |
| **Profiling（分析）** | Unreal Insights、RenderDoc、MemReport、DumpTicks |

---

# Gameplay 层的优化系统

除了 Renderer，Gameplay 也有自己的预算管理系统。

| 系统 | 作用 |
|------|------|
| Significance Manager | 根据重要性动态调整对象更新频率 |
| Animation Budget Allocator | 动画预算控制 |
| Iris Networking | 新一代网络同步系统 |
| Mass Framework | 大规模实体管理 |
| Gameplay Ability System | 能力系统（减少重复逻辑） |

---

# Profiling 工具

优化之前，必须先 Profiling。

| 工具 | 分析内容 |
|------|----------|
| Unreal Insights | CPU / Task / Thread |
| RenderDoc | GPU Frame |
| PIX / Nsight | GPU 调试 |
| MemReport | 内存 |
| DumpTicks | Tick 分析 |
| Stat GPU | GPU 时间 |
| Stat Unit | CPU / GPU |
| Stat Streaming | Streaming |

---

# UE5 优化思想总结

如果把 UE5 的所有优化技术放到一个框架里，可以归纳为：

```text
                   Performance

                         │

              Profiling（定位瓶颈）

                         │

      ┌────────────────────────────────┐
      │                                │
      ▼                                ▼

      Limit（限制）                Cache（缓存）

      少画                         不重复算
      少更新                       跨帧复用
      少计算                       预计算
      少上传                       预加载

      │                                │
      └────────────────────────────────┘

                         │

         Spatial / Resolution / Temporal / Lighting
```

---

# 总结

商业级 Unreal Engine 项目的优化，本质上并不是不断发明新的算法，而是在不同维度回答四个问题：

- **空间域（Spatial）**：什么不需要画？
- **分辨率域（Resolution）**：什么不需要按最高分辨率画？
- **时间域（Temporal）**：什么不需要每帧重新计算？
- **光照域（Lighting）**：什么不需要实时计算？

最终，这些优化都会落到两个核心原则上：

> **Limit（限制）**：减少需要处理的数据。

> **Cache（缓存）**：避免重复计算已经得到的结果。

这也是理解 UE5 Renderer、World Partition、GPU Scene、TSR、Lumen、HLOD 等技术背后共同设计思想的一把钥匙。