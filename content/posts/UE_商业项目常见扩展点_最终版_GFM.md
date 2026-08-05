---
title: "UE 商业项目常见扩展点（最终版大表）"
date: 2026-08-05
tags:
  - Unreal Engine
  - UE5
  - Architecture
categories:
  - Unreal Engine
description: "整理商业 UE 项目最常见的扩展入口与 Override 点。"
draft: false
---

<!--more-->

# Unreal Engine 商业项目常见扩展点（最终版大表）

> 本文采用 **GitHub Flavored Markdown (GFM)** 编写，可直接用于 **Hugo (PaperMod)**、GitHub、Obsidian、Typora。

| 层级 | 常见 Override / 扩展点 | 主要用途 | 典型内容 |
|------|------------------------|----------|----------|
| **事件解耦** | Delegate、Multicast Delegate、Dynamic Delegate | 模块解耦、事件广播、异步回调 | Gameplay Message、Observer Pattern |
| **Gameplay Framework** | GameMode、GameState、PlayerController、PlayerState、Pawn、Character、Actor、ActorComponent | 游戏主体架构 | 生命周期、输入、规则、角色、组件 |
| **组件生命周期** | OnComponentCreated、RegisterComponent、OnRegister、InitializeComponent、PostInitializeComponents、BeginPlay、TickComponent、OnUnregister | 初始化、注册、Tick、销毁 | 最常见 Override 点 |
| **显隐 / 渲染 / 物理** | SetVisibility、SetHiddenInGame、OnVisibilityChanged、SetupAttachment、AttachToComponent、CreateSceneProxy、ShouldCreatePhysicsState | 可见性、Render、Physics | LOD、Streaming、SceneProxy |
| **Activation** | Activate、Deactivate、OnComponentActivated、OnComponentDeactivated | 功能启停、对象池 | 延迟初始化 |
| **Subsystem** | EngineSubsystem、GameInstanceSubsystem、WorldSubsystem、LocalPlayerSubsystem、EditorSubsystem | 全局管理器 | Manager、Service、Cache |
| **GAS** | AbilitySystemComponent、GameplayAbility、GameplayEffect、GameplayCue、AttributeSet | 技能系统 | RPG、Buff、属性 |
| **数据驱动 / 契约层** | Interface、GameplayTags、PrimaryDataAsset、DataAsset、DataTable、CurveTable、AssetManager、DeveloperSettings | 数据驱动、解耦 | 配置、资源管理 |
| **Networking** | Replication、RPC、ReplicateSubobjects、FastArraySerializer、ReplicationGraph、Prediction、Reconciliation | 联机同步 | MMO、多人游戏 |
| **World Partition / Streaming** | PlayerController Streaming Source、StreamingSourceComponent、WorldPartitionSubsystem、StreamingPolicy、RuntimeSpatialHash、RuntimeGrid、DataLayer、HLOD | 开放世界 | Cell Streaming |
| **Plugin / Module / Game Feature** | Runtime Plugin、Editor Plugin、Runtime Module、Editor Module、Developer Module、Game Feature Plugin、Experience | 模块化、热更新、DLC | LiveOps |
| **AI** | AIController、BehaviorTree、StateTree、EQS、Navigation、AIPerception、SmartObject | AI 系统 | NPC、导航 |
| **Input** | Enhanced Input、Input Action、Mapping Context、Trigger、Modifier | 输入系统 | 多平台输入 |
| **UI** | UMG、Slate、Widget、CommonUI | UI 系统 | HUD、菜单 |
| **Blueprint** | Blueprint、BlueprintImplementableEvent、BlueprintNativeEvent、Blueprint Function Library、Blueprint Interface | 设计师扩展 | 可视化脚本 |
| **Animation** | AnimInstance、AnimGraph、AnimNode、AnimNotify、Control Rig、Motion Matching | 动画系统 | 角色动画 |
| **Rendering** | SceneViewExtension、PostProcess、Material、Niagara、Custom Pass、PrimitiveSceneProxy | 图形渲染 | 高级渲染扩展 |
| **Physics** | Chaos、MovementComponent、Collision、PhysicalMaterial | 物理系统 | 碰撞、角色移动 |
| **Editor / Tooling** | EditorSubsystem、Editor Module、Detail Customization、AssetTypeActions、Editor Utility Widget、Validation、Commandlet | 编辑器工具 | 自动化、生产力 |
| **Build / Packaging** | UBT、Build.cs、Target.cs、Cook、Pak、IoStore | 构建发布 | 打包、CI |
| **Mod** | Plugin-based Mod、Game Feature Mod、Pak Mod | Mod 支持 | 玩家扩展内容 |

---

# 推荐学习优先级

1. Gameplay Framework
2. Actor / ActorComponent 生命周期
3. Delegate
4. Subsystem
5. 数据驱动（Gameplay Tags / DataAsset）
6. GAS
7. Plugin + Module
8. Networking
9. World Partition / Streaming
10. Blueprint
11. AI
12. Animation
13. Rendering
14. Editor / Tooling
15. Build / Packaging
16. Mod
