---
title: "Unreal Engine 三种 Build 类型总结"
date: 2026-08-05
tags:
  - Unreal Engine
  - UE5
  - Build
  - Engine
categories:
  - Unreal Engine
summary: "总结 Unreal Engine 中 Launcher Build、GitHub Source Build 和 Installed Build 三种常见 Engine Build 的特点、区别及适用场景。"
---

<!--more-->

# Unreal Engine 三种 Build 类型总结

在 Unreal Engine 的开发过程中，最常见的 Engine Build 主要有三种：

1. **Launcher Build（官方安装版）**
2. **GitHub Source Build（源码版）**
3. **Installed Build（安装版源码引擎）**

它们最大的区别在于：

- Engine 是否可以修改
- Engine 是否需要自己编译
- 团队协作方式
- 适用的开发场景

---

# 三种 Build 对比

| Build 类型 | 来源 | 是否可修改 Engine | 是否需要编译 Engine | 适合对象 | 优点 | 缺点 |
|------------|------|------------------|--------------------|---------|------|------|
| **Launcher Build** | Epic Games Launcher | ❌ | ❌ | 普通游戏开发者、美术、策划 | 安装简单、更新方便、稳定 | 无法修改 Engine 源码 |
| **GitHub Source Build** | Epic GitHub | ✅ | ✅ | Engine Programmer、图形开发 | 完整源码，可自由修改 | 编译时间长，占用空间大 |
| **Installed Build** | BuildGraph | ⚠️（通常重新生成而不是直接修改） | 首次需要 | 企业团队 | 使用方便、统一版本 | 维护 BuildGraph 成本较高 |

---

# 1. Launcher Build（官方安装版）

## 来源

通过 Epic Games Launcher 安装。

```text
Epic Games Launcher
        │
        ▼
 已编译好的 Unreal Engine
        ├── UEEditor.exe
        ├── Engine DLL
        └── Headers
```

---

## 特点

Launcher Build 已经完成全部 Engine 编译。

开发者可以：

- 开发游戏
- 编译自己的 C++ Project
- 阅读 Engine Source（安装 Source 时）

但是不能：

- 修改 Engine
- Rebuild Engine
- 修改 Renderer
- 修改 Physics
- 修改 Animation Framework
- 修改 Lumen / Nanite

---

## 适用场景

推荐给：

- 游戏开发者
- 美术
- 技术美术（TA）
- 关卡设计
- 小型团队

> 对于绝大多数 UE 开发者来说，Launcher Build 已经足够使用。

---

# 2. GitHub Source Build（源码版）

## 来源

Epic GitHub：

```text
https://github.com/EpicGames/UnrealEngine
```

需要关联 Epic 与 GitHub 账号后才能访问。

---

## 编译流程

```text
Git Clone
      │
      ▼
 Setup.bat
      │
      ▼
 GenerateProjectFiles.bat
      │
      ▼
 Visual Studio
      │
      ▼
 Build UE5
```

最终得到：

```text
UE Source
     │
     ▼
 自己编译
     │
     ▼
 UEEditor.exe
```

---

## 最大优势

拥有完整 Engine Source。

可以修改任何模块，例如：

```text
Engine/
├── Runtime/
├── Renderer/
├── Core/
├── Slate/
├── Physics/
├── Animation/
├── AI/
└── Niagara/
```

全部都可以：

- Debug
- 修改源码
- Recompile
- 添加自己的 Engine Feature

---

## 适用场景

推荐给：

- Engine Programmer
- Rendering Engineer
- Physics Programmer
- Plugin Developer
- 学习 UE 底层实现

---

# 3. Installed Build（安装版源码引擎）

很多大型游戏公司都会采用这种方式。

典型流程如下：

```text
GitHub Source
       │
       ▼
 修改 Engine
       │
       ▼
 BuildGraph
       │
       ▼
 Installed Build
       │
       ▼
 分发整个团队
```

开发人员拿到的是：

```text
Custom Unreal Engine
        │
        ▼
 类似 Launcher 的安装体验
```

目录通常包括：

```text
UEEditor.exe
Engine/
Templates/
Plugins/
```

开发体验几乎与 Launcher Build 相同。

---

## 为什么需要 Installed Build？

企业通常都会对 Engine 做一些定制，例如：

- Renderer 修改
- Console 修改
- Build Tool 修改
- 内部 Plugin
- 第三方 SDK
- Performance Patch
- Bug Fix

如果每位开发者都自行编译：

- 编译时间长
- 容易版本不一致
- 环境配置复杂

因此通常由 Engine Team：

1. 修改 Engine
2. 编译一次
3. 生成 Installed Build
4. 分发给整个团队

其他开发人员无需重新编译即可直接使用。

---

# 三者关系

```text
                    Epic Games
                        │
                        ▼
              GitHub Source Code
                        │
          ┌─────────────┴─────────────┐
          │                           │
          ▼                           ▼
 Launcher Build              GitHub Source Build
（官方二进制）                 （源码编译）
                                      │
                                      ▼
                            修改 Engine Source
                                      │
                                      ▼
                                BuildGraph
                                      │
                                      ▼
                               Installed Build
                             （团队统一版本）
```

---

# 如何选择？

| 场景 | 推荐 |
|------|------|
| 普通游戏开发 | Launcher Build |
| 学习 Engine 源码 | GitHub Source Build |
| 修改 Engine | GitHub Source Build |
| 修改 Renderer | GitHub Source Build |
| 公司多人协作 | Installed Build |
| 企业统一开发环境 | Installed Build |

---

# 三种 Build 对比总结

| 项目 | Launcher | Source | Installed |
|------|----------|---------|-----------|
| 官方支持 | ✅ | ✅ | 企业内部 |
| 已编译 | ✅ | ❌ | ✅ |
| 可修改 Engine | ❌ | ✅ | ⚠️（通过重新生成） |
| 编译耗时 | 无 | 很长 | 首次生成较长 |
| 团队统一版本 | 一般 | 不方便 | ✅ |
| 更新简单 | ✅ | ❌ | ✅ |
| Engine Debug | 有限 | 完整支持 | 支持（取决于符号和源码配置） |

---

# 一句话总结

- **Launcher Build**：官方提供的二进制版本，安装简单，适合绝大多数游戏开发。
- **GitHub Source Build**：完整源码版本，适合修改 Engine、学习底层实现以及开发自定义功能。
- **Installed Build**：基于 Source Build 构建出的企业发行版本，兼顾源码定制能力和团队分发效率，是中大型 UE 项目的主流方案。