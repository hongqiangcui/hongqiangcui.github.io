---
title: "Mesh Shader Pipeline vs Traditional Raster Pipeline —— 从UE5 Render架构角度理解"
date: 2026-08-06
tags:
  - Unreal Engine
  - UE5
  - Rendering
  - Mesh Shader
  - Nanite
  - GPU Driven Rendering
summary: "从UE资深Render工程师的角度分析Mesh Shader Pipeline与传统Raster Pipeline的本质区别，以及Mesh Shader、GPU Driven Rendering和Nanite之间的关系。"
---

<!--more-->

# Mesh Shader Pipeline vs Traditional Raster Pipeline

很多文章介绍 Mesh Shader 时都会说：

> Mesh Shader 能减少 Draw Call，提高 GPU 性能。

这句话并没有错，但也并不完整。

如果站在 **UE5 Render Tech Lead / Graphics Architect** 的角度来看，**Mesh Shader 并不是单纯替代 Vertex Shader，而是 GPU Geometry Pipeline 的一次重新架构。**

真正发生变化的是：

> **Geometry Management（几何管理）的控制权，从 CPU 转移到了 GPU。**

---

# 一、传统 Raster Pipeline 的工作方式

传统 Pipeline（DX11 / DX12）如下：

```text
CPU

DrawIndexed()

        │
        ▼
Input Assembler (IA)
        │
Vertex Shader
        │
Hull Shader（可选）
        │
Tessellation
        │
Domain Shader
        │
Geometry Shader（很少使用）
        │
Primitive Assembly
        │
Rasterizer
        │
Pixel Shader
```

这里 CPU 的职责非常重：

- 遍历 Scene
- 判断哪些 Mesh 可见
- 选择 LOD
- 排序
- 提交 Draw Call

例如：

```cpp
for (Mesh mesh : VisibleMeshes)
{
    DrawIndexed(mesh);
}
```

GPU 只是负责执行。

---

# 二、传统 Pipeline 的几个核心问题

## 1. CPU 负责组织 Geometry

CPU 必须知道：

- 哪个 Mesh 需要绘制
- 绘制顺序
- LOD
- Instance

Scene 越大：

```text
更多 Mesh

↓

更多 Draw

↓

CPU 更忙
```

这也是 UE4 大场景下 CPU Render Thread 的主要瓶颈之一。

---

## 2. Vertex Shader 完全不知道 Mesh 的整体结构

Vertex Shader 的输入只有：

```text
一个 Vertex
```

它不知道：

- Mesh
- Cluster
- Bounding Box
- LOD
- Visibility

例如：

```text
Vertex 12345

↓

Vertex Shader

↓

计算 Position
```

它无法判断：

```cpp
if (Mesh 被遮挡)
{
    不画;
}
```

因为 VS 根本不知道自己属于哪个 Mesh。

---

## 3. IA（Input Assembler）完全固定

传统硬件要求：

```text
Vertex Buffer

+

Index Buffer

↓

IA

↓

Triangle
```

Triangle 必须来自 Index Buffer。

GPU 无法自己决定：

- 哪些 Triangle 要画
- Triangle 如何组织

---

# 三、Mesh Shader Pipeline

Mesh Shader Pipeline 变成：

```text
CPU

↓

DispatchMesh

↓

Task Shader（Amplification Shader）

↓

Mesh Shader

↓

Rasterizer

↓

Pixel Shader
```

最大的变化：

> **Input Assembler 消失了。**

GPU 不再必须依赖固定的 Vertex Buffer + Index Buffer。

Mesh Shader 可以直接生成 Triangle。

---

# 四、Mesh Shader 真正改变了什么？

## 1. Geometry 的组织权开始交给 GPU

传统：

```text
CPU

↓

Draw Mesh A

Draw Mesh B

Draw Mesh C
```

Mesh Shader：

```text
Task Shader

↓

判断哪些 Mesh / Cluster 可见

↓

生成 Mesh Shader 工作
```

GPU 可以自己决定：

- 是否需要执行 Mesh Shader
- 执行多少个 Mesh Shader

---

## 2. GPU 可以直接完成 Culling

传统：

```text
CPU

↓

Frustum Cull

↓

Draw
```

Mesh Shader：

```text
Task Shader

↓

Bounding Box Test

↓

LOD

↓

Occlusion

↓

Dispatch Mesh Shader
```

例如：

```text
10000 Clusters

↓

Task Shader

↓

1200 Visible

↓

只执行 1200 个 Mesh Shader
```

其余 8800 个 Cluster 根本不会进入 Rasterizer。

---

## 3. Meshlet 成为新的调度单位

传统：

```text
Vertex

↓

Triangle
```

Mesh Shader：

```text
Meshlet

↓

Vertices

+

Triangles
```

例如：

```text
Meshlet

64 Vertices

126 Triangles
```

Mesh Shader 一次处理整个 Meshlet。

---

# 五、为什么 Mesh Shader 特别适合 Nanite？

Nanite 本身就是：

```text
Mesh

↓

Cluster

↓

Cluster

↓

Cluster
```

每个 Cluster：

```text
64~128 Vertices

约128 Triangles
```

这与 Mesh Shader 的输出规模天然匹配。

因此：

```text
Nanite Cluster

↓

Mesh Shader

↓

直接输出 Triangle
```

完全绕过传统 IA。

---

# 六、一个经常被误解的问题

很多资料都会说：

> Mesh Shader 可以减少 CPU Draw Call。

实际上需要区分两个层面。

---

## 第一层：Mesh Shader API

如果只是使用 Mesh Shader API：

以前：

```cpp
DrawIndexed(meshA);
DrawIndexed(meshB);
DrawIndexed(meshC);
```

现在：

```cpp
DispatchMesh(meshA);
DispatchMesh(meshB);
DispatchMesh(meshC);
```

CPU 调用次数：

没有变化。

只是：

```text
DrawIndexed

↓

DispatchMesh
```

API 本身不会自动减少 Dispatch。

---

## 第二层：GPU Driven Rendering

真正减少 CPU Draw Call 的，并不是 Mesh Shader。

而是：

- GPU Scene
- GPU Culling
- ExecuteIndirect / DispatchMeshIndirect

真正流程变成：

```text
CPU

↓

Dispatch Compute Cull

↓

GPU 生成 Visible List

↓

GPU 生成 Dispatch Arguments

↓

ExecuteIndirect

↓

GPU 自动执行大量 Mesh Shader
```

CPU 不再：

```cpp
for(mesh)
{
    DispatchMesh(mesh);
}
```

而是：

```cpp
ExecuteIndirect();
```

CPU 甚至不知道最终执行了多少个 Mesh Shader。

---

# 七、Nanite 更进一步

Nanite 连 Mesh 都不是调度单位。

例如：

```text
Building

↓

5000 Clusters
```

GPU：

```text
Cull Cluster

↓

Visible Cluster

↓

DispatchMeshIndirect
```

CPU：

```text
Dispatch Compute

↓

ExecuteIndirect
```

结束。

CPU 不再管理：

- Cluster
- Draw
- Dispatch

这些全部发生在 GPU。

---

# 八、Mesh Shader 与 GPU Driven Rendering 的关系

很多文章把这两个概念混在一起。

实际上它们属于不同层次。

| 技术 | 作用 |
|------|------|
| Mesh Shader | 新的几何处理 Shader Stage |
| Task Shader | Mesh Shader 工作调度 |
| GPU Scene | Scene 数据常驻 GPU |
| GPU Culling | GPU 决定哪些对象可见 |
| ExecuteIndirect / DispatchMeshIndirect | GPU 自己发起绘制 |
| Nanite | Cluster + GPU Culling + Indirect Dispatch + Mesh Shader（可选） |

Mesh Shader 并不会自动带来 GPU Driven Rendering。

GPU Driven Rendering 也并不一定要求 Mesh Shader。

两者结合，才形成 UE5 新一代几何渲染架构。

---

# 九、传统 Pipeline vs Mesh Shader Pipeline

| 传统 Raster Pipeline | Mesh Shader Pipeline |
|----------------------|----------------------|
| CPU 组织 Draw | GPU 可以组织 Geometry |
| IA 固定读取 VB/IB | IA 被移除 |
| Vertex Shader 处理单个 Vertex | Mesh Shader 处理整个 Meshlet |
| Draw Call 数量大 | 更适合 GPU Indirect Dispatch |
| CPU 做 Culling | GPU 做 Culling |
| Primitive 来自 Index Buffer | Primitive 可编程生成 |
| Vertex 不知道 Mesh | Mesh Shader 知道整个 Cluster |

---

# 十、总结

很多人认为：

> Mesh Shader = Draw Call 更少。

其实并不准确。

真正准确的理解应该是：

> **Mesh Shader 提供了新的几何处理能力。**

而真正让 CPU 不再逐个提交 Draw 的，是：

- GPU Scene
- GPU Culling
- ExecuteIndirect / DispatchMeshIndirect

Mesh Shader 只是 GPU Driven Rendering 中负责 Geometry Processing 的最后一环。

因此，从 UE5 Render 架构来看：

> **传统 Raster Pipeline 的核心思想是：CPU 负责组织 Geometry，GPU 负责执行 Rasterization。**

而现代 Mesh Shader + GPU Driven Rendering 的核心思想则是：

> **GPU 不仅负责 Rasterization，也负责 Geometry 的组织、可见性判断、LOD 选择以及 Primitive 的生成。**

这也是 Nanite、GPU Scene、Cluster Culling、Mesh Shader 等技术能够组合在一起，构成 UE5 新一代 Geometry Pipeline 的根本原因。