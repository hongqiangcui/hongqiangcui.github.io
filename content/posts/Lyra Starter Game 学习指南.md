---
title: "Lyra Starter Game 学习指南"
date: 2026-08-06
tags:
  - Unreal Engine
  - UE5
  - Lyra
  - Gameplay Ability System
  - Game Framework
categories:
  - Unreal Engine
summary: "Epic 官方 Lyra Starter Game 学习路线与知识点整理。"
---

<!--more-->

# Lyra Starter Game 学习指南

> Lyra Starter Game 是 Epic 官方推出的 UE5 最佳实践项目，它不仅是一个示例游戏，更是一套现代 Unreal Engine 游戏架构参考实现。

推荐学习顺序：

> Framework → Experience → GameFeature → GAS → Input → Character → Animation → Weapon → Inventory → UI → Multiplayer → Asset → Performance

---

# 学习总览

| 模块 | 推荐指数 | 难度 | 是否核心 |
|------|----------|------|----------|
| Game Framework | ⭐⭐⭐⭐⭐ | ★★☆☆☆ | ✅ |
| Experience System | ⭐⭐⭐⭐⭐ | ★★★★☆ | ✅ |
| Game Feature Plugin | ⭐⭐⭐⭐⭐ | ★★★★☆ | ✅ |
| Gameplay Ability System | ⭐⭐⭐⭐⭐ | ★★★★★ | ✅ |
| Enhanced Input | ⭐⭐⭐⭐⭐ | ★★☆☆☆ | ✅ |
| Character | ⭐⭐⭐⭐⭐ | ★★★☆☆ | ✅ |
| Animation | ⭐⭐⭐⭐⭐ | ★★★★☆ | ✅ |
| Camera | ⭐⭐⭐⭐⭐ | ★★★☆☆ | ✅ |
| Weapon | ⭐⭐⭐⭐⭐ | ★★★★☆ | ✅ |
| Inventory | ⭐⭐⭐⭐☆ | ★★★☆☆ | ✅ |
| CommonUI | ⭐⭐⭐⭐⭐ | ★★★☆☆ | ✅ |
| Multiplayer | ⭐⭐⭐⭐⭐ | ★★★★★ | ✅ |
| AI | ⭐⭐⭐⭐☆ | ★★★☆☆ | |
| Audio | ⭐⭐⭐⭐☆ | ★★☆☆☆ | |
| Asset Manager | ⭐⭐⭐⭐⭐ | ★★★☆☆ | ✅ |
| Performance | ⭐⭐⭐⭐⭐ | ★★★★☆ | ✅ |

---

# 1. Game Framework

## 学习目标

理解 UE5 游戏生命周期。

## 主要类

| 类 | 作用 |
|-----|------|
| LyraGameMode | 游戏模式 |
| LyraGameState | 游戏状态同步 |
| LyraPlayerController | 输入管理 |
| LyraPlayerState | GAS入口 |
| LyraCharacter | 玩家角色 |
| LyraPawn | Pawn封装 |
| LyraWorldSettings | 世界配置 |

## 学习重点

- 生命周期
- GameMode
- PlayerState
- Controller
- Pawn
- Character
- Replication

---

# 2. Experience System（Lyra 核心）

Experience 可以理解为：

> 一套完整的游戏模式配置。

例如：

- Shooter
- TopDown
- Elimination
- Capture The Flag

Experience 包含：

- Pawn
- HUD
- UI
- Camera
- Ability
- Input
- Components
- Plugins

## 启动流程

```
GameMode
    │
    ▼
ExperienceManager
    │
    ▼
Load Experience
    │
    ▼
GameFeature
    │
    ▼
Pawn
    │
    ▼
Ability
    │
    ▼
HUD
```

学习重点：

- Experience Definition
- Experience Action Set
- Async Loading
- Gameplay Tags

---

# 3. Game Feature Plugin

目录示例：

```
Plugins/

    ShooterCore/

    TopDownArena/
```

学习内容：

| 功能 | 描述 |
|------|------|
| Runtime加载 | 热插拔插件 |
| Ability注册 | 自动注入 |
| UI扩展 | 自动添加Widget |
| Input扩展 | 自动绑定输入 |
| Pawn扩展 | 自动增加组件 |

重点：

官方推荐的大型项目架构。

---

# 4. Gameplay Ability System（GAS）

Lyra 对 GAS 的应用非常完整。

学习内容：

| 模块 |
|------|
| Ability |
| Gameplay Effect |
| Gameplay Cue |
| Attribute |
| Ability Set |
| Cooldown |
| Cost |
| Gameplay Tags |
| Prediction |
| Replication |

流程：

```
Input

↓

Ability

↓

GameplayEffect

↓

Attribute

↓

GameplayCue

↓

Replication
```

重点类：

- LyraAbilitySystemComponent
- LyraGameplayAbility
- LyraAttributeSet
- LyraAbilitySet

---

# 5. Enhanced Input

学习内容：

- Input Action
- Input Mapping Context
- Input Config
- Gameplay Tag
- Ability绑定

Lyra 最大特点：

```
按键

↓

GameplayTag

↓

Ability
```

而不是：

```
按键

↓

函数
```

实现输入与逻辑解耦。

---

# 6. Character

学习内容：

- Character Movement
- Pawn Extension
- Team System
- Health
- Death
- Respawn

重点：

组件化角色。

---

# 7. Animation

学习内容：

- Anim Blueprint
- Linked Anim Graph
- Layer
- Motion Warping
- Distance Matching
- Orientation Warping
- Root Motion
- Aim Offset
- Foot IK

重点：

体验 UE5 最新动画系统。

---

# 8. Camera

学习内容：

- Camera Mode
- Camera Stack
- ADS
- Third Person
- First Person

流程：

```
Player

↓

CameraMode

↓

CameraStack

↓

Camera
```

---

# 9. Weapon

学习内容：

| 模块 |
|------|
| Weapon Definition |
| Weapon Instance |
| Fire Ability |
| Projectile |
| HitScan |
| ADS |
| Ammo |
| Reload |
| Spread |

流程：

```
WeaponDefinition

↓

WeaponInstance

↓

Ability

↓

Fire

↓

Damage
```

---

# 10. Inventory

学习内容：

- Inventory Item
- Equipment
- Pickup
- QuickBar
- Inventory Fragment

特点：

全部 DataAsset 驱动。

---

# 11. UI（CommonUI）

Lyra 全部采用 CommonUI。

学习内容：

- HUD
- Activatable Widget
- Common Button
- Layer
- UI Policy
- Async UI

流程：

```
Player

↓

HUD

↓

Layer

↓

Widget
```

---

# 12. Multiplayer

Lyra 从设计之初即支持多人联机。

学习内容：

- Replication
- RPC
- Prediction
- Authority
- GAS同步
- Weapon同步
- Character同步

重点：

Epic 官方多人最佳实践。

---

# 13. AI

学习内容：

- AI Controller
- Behavior Tree
- Blackboard
- EQS
- Perception
- Team

---

# 14. Audio

学习内容：

- MetaSound
- Gameplay Cue
- Audio Component
- Spatial Audio

---

# 15. Asset Manager

学习内容：

- Primary Asset
- Asset Bundle
- Soft Object
- Async Loading
- StreamableManager

Lyra 大量采用：

- SoftObjectPtr
- PrimaryAsset
- AssetManager

实现资源按需加载。

---

# 16. SaveGame

学习内容：

- SaveGame
- User Settings
- Local Settings

Lyra 保存内容较少，主要作为参考实现。

---

# 17. Performance

学习内容：

- Async Loading
- Soft Reference
- Object Pool
- DataAsset
- Component化
- GameFeature 按需加载

重点：

大型项目资源管理。

---

# 18. 设计模式

Lyra 大量采用现代软件设计思想。

| 模式 | 应用 |
|------|------|
| 数据驱动 | Experience、Weapon、Inventory |
| Component | Character、Weapon |
| Plugin | Game Feature |
| Strategy | Camera |
| Factory | Pawn创建 |
| Observer | Gameplay Message |
| Command | Enhanced Input |
| State | Character状态 |
| Event Bus | Gameplay Message System |

---

# 推荐学习路线

| 周数 | 学习内容 |
|------|----------|
| 第1周 | Framework |
| 第2周 | Experience |
| 第3周 | Game Feature Plugin |
| 第4周 | Enhanced Input |
| 第5周 | Gameplay Ability System |
| 第6周 | Character + Animation |
| 第7周 | Weapon + Inventory |
| 第8周 | CommonUI |
| 第9周 | Multiplayer |
| 第10周 | Asset + Performance + AI |

---

# 总结

Lyra Starter Game 最值得学习的五大核心：

1. Experience System
2. Game Feature Plugin
3. Gameplay Ability System（GAS）
4. Enhanced Input
5. CommonUI

掌握以上内容后，再深入研究 Weapon、Inventory、Animation、Multiplayer 与 AssetManager，将能够建立现代 UE5 大型项目的整体架构思维，为开发可扩展、可维护的游戏奠定坚实基础。