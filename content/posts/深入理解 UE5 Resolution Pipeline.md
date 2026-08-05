---
title: "深入理解 UE5 Resolution Pipeline：Primary、Secondary 与 Final Resolution"
date: 2026-08-05
tags:
  - Unreal Engine
  - UE5
  - Rendering
  - TSR
  - Resolution
  - Upscaling
categories:
  - Unreal Engine
summary: "深入理解 Unreal Engine 5 的多分辨率渲染架构，包括 Dynamic Primary Resolution、Secondary Resolution、Final Resolution、TSR 的工作流程以及性能优化思路。"
---

<!--more-->

# 深入理解 UE5 Resolution Pipeline

很多人理解 UE 的分辨率只有一个：

> **1920×1080**
>
> 或者
>
> **3840×2160**

实际上，从 UE5 开始，Renderer 内部已经同时存在多个 Resolution。

Epic 在介绍 TSR（Temporal Super Resolution）时，把整个 Pipeline 画成了下面这种形式：

![UE5 Resolution Pipeline](/images/ue/resolution-pipeline.png)

理解这张图，对于理解：

- TSR
- Dynamic Resolution
- Screen Percentage
- DLSS
- FSR
- Mobile Upscaling

都会轻松很多。

---

# UE5 为什么需要多个 Resolution？

答案只有一个：

> **不是所有 Pass 都值得在最终分辨率运行。**

例如：

假设目标输出：

```text
3840 × 2160（4K）
```

真正最贵的是：

```text
Geometry

Shadow

Lumen

Nanite

BasePass
```

如果全部在 4K：

GPU 成本极高。

因此：

Renderer 希望：

```text
先低分辨率渲染

↓

恢复画质

↓

最后输出 4K
```

于是就产生了：

三个 Resolution。

---

# UE5 三种 Resolution

整个 Pipeline 可以理解成：

```text
Primary Resolution

↓

Secondary Resolution

↓

Final Resolution
```

它们代表三个不同阶段。

---

# ① Dynamic Primary Resolution

这是：

真正进行 Scene Rendering 的 Resolution。

例如：

```text
800p

900p

1080p
```

都是可能的。

Epic 图里的：

```text
Dynamic Primary Resolution

800~1080p
```

就是说：

Renderer 会动态改变它。

---

## 哪些 Pass 在这里运行？

例如：

```text
Geometry

Depth

Shadow

Nanite

Lumen

Base Pass

Lighting

DoF
```

几乎所有：

最贵的 Pass。

所以：

Primary Resolution 越低：

GPU 开销越低。

---

# 为什么叫 Dynamic？

因为：

它可以实时变化。

例如：

目标：

```text
60 FPS
```

GPU 开始掉帧：

```text
60 FPS

↓

55 FPS
```

UE：

自动：

```text
1080p

↓

1000p

↓

900p
```

GPU：

恢复：

60 FPS。

这就是：

Dynamic Resolution。

---

# Primary Resolution 是性能核心

GPU Pixel 数：

```text
1920×1080

≈ 2M Pixels
```

而：

```text
3840×2160

≈ 8M Pixels
```

直接：

4 倍。

所以：

降低 Primary Resolution：

收益非常大。

---

# ② TSR（Temporal Super Resolution）

这是：

UE5 最重要的一步。

TSR：

负责：

```text
低分辨率

↓

恢复高质量图像
```

例如：

```text
900p

↓

1440p
```

这里：

Renderer 利用：

- Motion Vector
- History Buffer
- Temporal AA
- Reprojection

恢复细节。

因此：

TSR：

不是：

```text
Resize
```

而是：

```text
Temporal Reconstruction
```

---

# 为什么 DoF 在 TSR 前？

因为：

DoF：

属于：

Scene Rendering。

它依赖：

真实 Scene Depth。

所以：

必须：

在：

Primary Resolution。

否则：

会导致：

错误的 Blur。

---

# ③ Secondary Resolution

TSR 输出之后：

得到：

Secondary Resolution。

例如：

```text
1440p
```

这里：

已经：

恢复了：

比较完整的图像。

于是：

后面的：

Post Process：

可以继续工作。

---

## 哪些 Pass 在这里？

例如：

```text
Motion Blur

Bloom

Tonemap
```

这些：

已经：

不需要：

Scene Geometry。

因此：

可以：

在：

Secondary Resolution。

这样：

成本：

远低于：

4K。

---

# 为什么 Motion Blur 在 TSR 后？

Motion Blur：

需要：

完整画面。

但是：

不用：

重新渲染 Geometry。

所以：

TSR：

恢复画面。

↓

Motion Blur

↓

Bloom

↓

Tonemap

更加合理。

---

# ④ Upscale

最后：

还有一步：

```text
Upscale
```

例如：

```text
1440p

↓

2160p（4K）
```

这里：

可能：

只是：

Image Upscale。

不是：

重新渲染。

---

# ⑤ UI

为什么：

UI：

放在：

Upscale 后？

因为：

UI：

必须：

保持：

Pixel Perfect。

例如：

字体。

如果：

UI：

跟着：

900p：

渲染。

最后：

放大。

字体：

一定：

模糊。

所以：

UE：

先：

完成：

Scene。

最后：

再：

绘制：

UI。

---

# ⑥ Backbuffer

最终：

写入：

```text
Backbuffer
```

例如：

```text
3840×2160
```

然后：

Swap Chain：

Present。

最终：

显示：

屏幕。

---

# 整个 Pipeline

整理一下：

```text
Primary Resolution

Geometry

Depth

Nanite

Lumen

Lighting

DoF

↓

TSR

↓

Secondary Resolution

↓

Motion Blur

↓

Bloom

↓

Tonemap

↓

Upscale

↓

UI

↓

Backbuffer

↓

Monitor
```

---

# 三种 Resolution 对比

| Resolution | 主要职责 | 是否动态 | 典型分辨率 |
|------------|----------|----------|------------|
| Primary Resolution | 场景渲染（Geometry、Lighting、Nanite 等） | ✅ | 800p～1080p |
| Secondary Resolution | TSR 输出后的后处理阶段 | 一般固定 | 1440p |
| Final Resolution | 最终输出到 Backbuffer | 固定 | 4K |

---

# 为什么这样设计？

原因就是：

GPU 最贵的是：

```text
Geometry

Shadow

Lighting

Lumen
```

而：

Bloom

Tonemap

UI

这些：

成本：

低很多。

因此：

Renderer：

选择：

```text
低分辨率

↓

恢复画面

↓

最后输出高分辨率
```

这样：

GPU：

获得：

巨大收益。

---

# 与 DLSS、FSR、XeSS 的关系

实际上：

DLSS

FSR

XeSS

和 TSR：

都属于：

这一层：

```text
Primary Resolution

↓

Upscaler

↓

Higher Resolution
```

区别在于：

| 技术 | 实现 |
|------|------|
| TSR | UE 自研 Temporal Upscaler |
| DLSS | NVIDIA AI Upscaler |
| FSR2 / FSR3 | AMD Temporal Upscaler |
| XeSS | Intel AI Upscaler |

它们工作的阶段基本一致，只是重建算法不同。

---

# 对性能优化意味着什么？

这张图告诉我们一个重要事实：

> **降低最终输出分辨率，并不一定是最有效的优化方式。**

真正影响 GPU 成本的是：

- **Primary Resolution（Primary Screen Percentage）**
- **Geometry Pass**
- **Lumen**
- **Nanite**
- **Shadow**
- **Lighting**

因此，商业项目中更常见的策略是：

- 动态调整 Primary Resolution（Dynamic Resolution）
- 使用 TSR / DLSS / FSR 恢复画质
- 保持 Final Resolution 不变，确保 UI 和显示效果稳定

这样既能维持目标帧率，又能尽量保留高分辨率的视觉体验。

---

# 总结

UE5 的 Resolution 不再是一个单一的概念，而是一个贯穿整个 Renderer 的分层体系：

```text
Primary Resolution
（真正渲染场景）

        ↓

Temporal Reconstruction（TSR / DLSS / FSR）

        ↓

Secondary Resolution
（后处理）

        ↓

Upscale

        ↓

Final Resolution
（Backbuffer）
```

**一句话总结：**

> **UE5 不再追求“全程 4K 渲染”，而是追求“在最合适的阶段使用最合适的分辨率”。**
>
> 这种多分辨率渲染架构，是 UE5 能够在大型场景中兼顾画质与性能的关键设计之一。