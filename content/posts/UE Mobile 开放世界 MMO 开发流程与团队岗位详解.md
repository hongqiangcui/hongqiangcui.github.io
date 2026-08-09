---
title: UE Mobile 开放世界 MMO 开发流程与团队岗位详解
date: 2026-08-09
tags:
  - Unreal Engine
  - MMO
  - Game Development
  - Mobile
  - Rendering
  - Engine
categories:
  - Game Development
summary: 以 UE5 Mobile 开放世界 MMO 为例，系统介绍游戏研发流程、团队组织架构、各岗位职责、客户端与服务器分工以及 UE 技术团队的工作内容。
---

<!--more-->

# UE Mobile 开放世界 MMO 开发流程与团队岗位详解

本文以 **UE5 Mobile 开放世界 MMO**（类似《原神》《鸣潮》《幻塔》等）的研发流程为例，介绍：

- 一个大型游戏团队如何分工
- 每个岗位负责什么
- 最终如何落地到 Unreal Engine
- 客户端与服务器如何协同工作
- Rendering Team 在整个项目中的位置

---

# 一、整体组织架构

```text
                          制作人
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
     策划部门             美术部门             技术部门
        │                    │                    │
        └──────────────┬─────┴──────────────┘
                       │
                  Unreal Engine
                       │
               Client + Server
                       │
                     QA & Ops
```

---

# 二、团队组成总览

| 部门 | 主要职责 | 主要输出 |
|------|----------|----------|
| 制作 | 项目管理、排期、资源协调 | Roadmap、版本计划 |
| 策划 | 游戏玩法、数值、关卡、剧情 | Design Doc、Excel、DataTable |
| 美术 | 模型、贴图、动画、特效、灯光 | FBX、Texture、Material |
| 客户端 | Gameplay、UI、Rendering、Engine | UE C++、Blueprint |
| 服务器 | 登录、战斗、同步、数据库 | Server Code |
| TA | 美术工具、Shader、自动化 | Material、Python、Pipeline |
| QA | Bug、性能、兼容性 | Bug Report |
| 运维 | 部署、监控、热更新 | DevOps |

---

# 三、策划部门

| 岗位 | 工作内容 | UE 最终落地 |
|------|----------|------------|
| 主策划 | 世界观、核心玩法 | Design Document |
| 系统策划 | 技能、装备、商城 | DataTable |
| 数值策划 | HP、伤害、Buff、掉率 | Excel → UDataTable |
| 关卡策划 | 地图、NPC、机关 | Level、World Partition |
| 剧情策划 | 剧情、对白、演出 | Level Sequence |

---

## 数值策划的工作

例如：

| 系统 | 内容 |
|------|------|
| HP | 玩家生命值 |
| MP | 法力值 |
| Attack | 攻击力 |
| Buff | 增益/减益状态 |
| 掉率 | Boss Loot |

最终：

```text
Excel
      │
      ▼
CSV
      │
      ▼
UE DataTable
```

---

# 四、美术部门

## 角色美术

流程：

```text
Concept
    │
    ▼
ZBrush
    │
    ▼
Retopo
    │
    ▼
UV
    │
    ▼
Texture
    │
    ▼
Rig
    │
    ▼
FBX
```

---

## 场景美术

负责：

- 房屋
- 山体
- 道路
- 岩石
- 桥梁

输出：

Static Mesh。

---

## 道具美术

例如：

- 武器
- 宝箱
- 路灯
- 桌椅

输出：

Static Mesh。

---

## 植被美术

负责：

- 树
- 草
- 花
- 灌木

同时需要考虑：

| PC | Mobile |
|----|--------|
| Nanite | LOD |
| Wind | Wind |
| Virtual Shadow | Shadow Map |

---

## Material Artist

负责：

- PBR Material
- Master Material
- Landscape Material
- Water Material

不是写 Gameplay。

---

## Lighting Artist

Lighting Artist 并不是简单地"放灯"。

主要负责：

| 工作 | 内容 |
|------|------|
| HDRI | 环境光 |
| SkyLight | 天空光 |
| Directional Light | 太阳/月亮 |
| Sky Atmosphere | 大气 |
| Fog | 雾 |
| Exposure | 曝光 |
| Color Grading | 调色 |

---

### HDRI 是什么？

HDRI（High Dynamic Range Image）

不是普通图片。

它记录：

- 颜色
- 光照方向
- 光照强度

例如：

```
HDRI
    │
    ▼
SkyLight
    │
    ▼
Image Based Lighting (IBL)
```

Lighting Artist 负责整个游戏画面的氛围。

---

## Animator

负责：

- 跑步
- 跳跃
- 攻击
- 技能
- 表情

输出：

- Animation Sequence
- Montage
- BlendSpace

---

## VFX Artist

负责：

- Niagara
- 火焰
- 爆炸
- 魔法
- 天气

---

## Technical Artist（TA）

TA 是连接程序和美术的重要岗位。

主要负责：

| 工作 | 内容 |
|------|------|
| Shader | 材质 |
| Material | 主材质 |
| Python | 自动导入 |
| Pipeline | 美术流程 |
| LOD | 自动生成 |
| Blueprint | 工具 |

---

# 五、客户端程序

客户端一般可以分成几个方向。

---

## Gameplay Programmer

负责：

- 技能
- Buff
- AI
- 战斗
- 玩家交互

---

### Buff 是什么？

Buff：

就是角色身上的持续状态（Status Effect）。

例如：

| Buff | 效果 |
|------|------|
| 攻击提升 | Attack +20% |
| 中毒 | 每秒掉血 |
| 冰冻 | 无法移动 |
| 减速 | Speed -40% |
| 护盾 | 吸收伤害 |

大型项目：

一般都会有：

```
Character
      │
      ▼
Buff Manager
      │
      ├── Buff A
      ├── Buff B
      ├── Buff C
```

UE GAS 中：

GameplayEffect 基本就是 Buff。

---

## UI Programmer

负责：

- HUD
- 商城
- 背包
- 聊天
- 小地图

---

## AI Programmer

负责：

- Behavior Tree
- EQS
- Navigation
- NPC

---

## Physics Programmer

负责：

- Chaos
- Vehicle
- Collision
- Ragdoll

---

## Rendering Programmer

这是 UE 底层开发的重要岗位。

负责：

| 模块 | 内容 |
|------|------|
| Renderer | 渲染器 |
| Shader | Shader |
| Lighting | 光照 |
| Material | 材质 |
| Mesh Pipeline | Mesh 渲染 |
| GPU Scene | GPU 驱动渲染 |
| Virtual Shadow | 阴影 |
| Virtual Texture | VT |
| Instancing | 实例化 |
| Occlusion | 遮挡剔除 |

---

## Engine Programmer

负责：

修改 UE 源码。

例如：

- TaskGraph
- Animation
- Memory
- Asset Streaming
- RHI
- Renderer

---

# 六、GPU Scene 到底需要程序员做什么？

很多人认为：

GPU Scene 是 UE 自带。

程序员不用管。

其实：

需要区分：

## 游戏开发

基本：

不用修改。

直接：

使用 UE。

---

## 引擎开发

很多 AAA：

都会修改：

例如：

| 修改方向 | 举例 |
|----------|------|
| Instance Culling | 自定义剔除策略 |
| GPU Buffer Layout | 修改 Instance 数据结构 |
| Streaming | GPU Scene Streaming |
| Draw Command | 合并 Draw Command |
| Cluster | 自定义 Cluster |

---

### 不是 Override FInstanceCullingContext

很多人误会：

需要：

```cpp
class MyCulling
: public FInstanceCullingContext
```

实际上：

不是。

FInstanceCullingContext：

更多是：

Renderer 内部使用的 Context。

真正修改：

一般是：

- GPU Scene 写入
- Draw Command Build
- Instance 数据结构
- GPU Buffer

而不是：

Override Context。

---

# 七、服务器程序

服务器通常包括：

| 模块 | 功能 |
|------|------|
| Login | 登录 |
| Gateway | 网络转发 |
| World | 世界 |
| Battle | 战斗 |
| Chat | 聊天 |
| Database | 存档 |

---

# 八、客户端与服务器关系

很多新人认为：

客户端和服务器：

共用一套代码。

实际上：

不是。

---

## 可以共享

| 模块 | 是否共享 |
|------|----------|
| Protocol | ✅ |
| Enum | ✅ |
| 配置 | ✅ |
| 数学库 | ✅ |
| Utility | ✅ |

---

## 客户端独有

| 模块 |
|------|
| Rendering |
| Animation |
| UI |
| Camera |
| Audio |

---

## 服务器独有

| 模块 |
|------|
| Login |
| Battle |
| Database |
| AI |
| Save |
| Sync |

---

## 一个技能如何工作？

例如：

FireBall。

配置：

共享：

```
Damage
Mana
Cooldown
```

客户端：

负责：

- 播放动画
- 播放特效
- 播放声音

服务器：

负责：

- 扣血
- 计算暴击
- 网络同步

因此：

逻辑不同。

代码不同。

---

# 九、UE Dedicated Server

UE：

支持：

```
同一工程

↓

Client

↓

Dedicated Server
```

例如：

```cpp
#if UE_SERVER

CalculateDamage();

#endif
```

服务器：

编译。

客户端：

不会。

---

## 真正的大型 MMO

很多：

不会直接使用 UE Server。

例如：

```
UE Client
      │
      ▼
Gateway
      │
      ▼
Go / C++ / Java / Rust
```

因为：

服务器：

更关注：

- 并发
- 网络
- 数据

而不是：

Renderer。

---

# 十、项目资源流转

```text
策划
 │
 ▼
Design Doc
 │
 ▼
Concept
 │
 ▼
角色/场景/道具
 │
 ▼
FBX
 │
 ▼
TA 自动导入 UE
 │
 ▼
Material
 │
 ▼
Gameplay
 │
 ▼
Level
 │
 ▼
Server
 │
 ▼
QA
 │
 ▼
发布
```

---

# 十一、典型团队规模（约100人）

| 部门 | 人数 |
|------|------|
| 制作 | 2~4 |
| 策划 | 10~15 |
| 美术 | 35~45 |
| 客户端 | 15~20 |
| 服务器 | 10~15 |
| 音频 | 2~4 |
| QA | 10~15 |

---

# 十二、如果想成为 UE 技术负责人

建议重点学习：

| 方向 | 学习内容 |
|------|----------|
| Rendering | Mesh、Shader、GPU Scene、Lighting、Material、Instancing |
| Engine | TaskGraph、Memory、Streaming、Animation |
| Gameplay | GAS、Buff、AI、Network |
| Optimization | CPU、GPU、Memory、Mobile Profiling |

建议学习路线：

```text
UE Client Tech
│
├── Rendering
│   ├── Mesh Pipeline
│   ├── Material
│   ├── Shader
│   ├── Lighting
│   ├── GPU Scene
│   ├── Instancing
│   ├── Occlusion
│   └── Mobile Rendering
│
├── Engine
│   ├── TaskGraph
│   ├── Memory
│   ├── Streaming
│   ├── Animation
│   └── RHI
│
├── Gameplay
│   ├── GAS
│   ├── Buff
│   ├── AI
│   └── Networking
│
└── Optimization
    ├── CPU
    ├── GPU
    ├── Memory
    ├── Asset Pipeline
    └── Mobile Performance
```

---

# 总结

一个 UE Mobile 开放世界 MMO 的开发，本质上是多个专业团队共同协作的结果：

- **策划** 定义游戏规则和玩法。
- **美术** 负责创造世界、角色和视觉效果。
- **TA** 搭建美术与引擎之间的桥梁，实现自动化流程。
- **客户端程序** 负责 Gameplay、UI、Rendering、Engine 等功能。
- **服务器程序** 负责登录、世界、战斗、同步和数据存储。
- **QA 与运维** 保证游戏质量和持续运营。

对于希望深入 Unreal Engine 底层的开发者而言，**Rendering、Engine、GPU Scene、Asset Streaming、移动端性能优化以及网络同步** 是开放世界 MMO 最具技术挑战、也是最值得长期投入的核心方向。