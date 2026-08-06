---
title: "Unreal Engine C++ Handbook"
subtitle: "Based on Tom Looman's UE C++ Guide (Reorganized)"
description: "UE5 C++ 开发知识手册，从 C++ 基础到 Unreal Reflection、Gameplay Framework、GC、Modules 等核心知识。"

tags:
  - Unreal Engine
  - UE5
  - C++
  - Gameplay Framework
  - UObject

categories:
  - Unreal Engine

---

<!--more-->

# Unreal Engine C++ Handbook

> 本文是一份 Unreal Engine C++ 开发知识手册，结合社区最佳实践与 Unreal Engine 官方架构，对 UE C++ 的核心概念进行系统整理。

---

# 为什么学习 UE C++

很多刚接触 UE 的开发者都会问：

> **既然 Blueprint 可以完成大部分功能，为什么还要学习 C++？**

实际上，两者并不是竞争关系，而是互补关系。

推荐采用 **Hybrid Development（混合开发）**：

```
            Gameplay

       Blueprint Layer
        UI
        Animation
        FX
        Audio
        Designers

-----------------------------

        C++ Layer

 Game Framework
 Ability System
 SaveGame
 AI
 Networking
 Inventory
 Core Gameplay

-----------------------------

 Unreal Engine Runtime
```

## Blueprint 更适合

- UI
- Widget
- Animation Blueprint
- Niagara
- 特效
- 快速验证玩法
- Level Script

优点：

- 开发速度快
- 易于调试
- 美术、策划均可参与

---

## C++ 更适合

适用于需要：

- 高性能
- 可维护性
- 大规模项目
- 网络同步
- 模块化设计

例如：

- Inventory
- GAS
- AI
- Save System
- Save Serialization
- Online System
- Gameplay Framework

---

## 推荐架构

```
Designer

      │

Blueprint

      │

Extend

      ▼

Native C++ Classes

      │

Gameplay Framework

      │

Engine
```

Blueprint 应尽量负责：

- 调整参数
- 组合功能
- 美术表现

核心逻辑尽量放在 C++。

---

# UE C++ 项目结构

典型项目：

```
Source/

    MyGame/

        Public/

        Private/

        MyGame.Build.cs

        MyGame.cpp

        MyGame.h
```

其中：

## Public

对外暴露 Header。

例如：

```
Public/

InventoryComponent.h

Weapon.h

HealthComponent.h
```

---

## Private

实现文件：

```
Private/

InventoryComponent.cpp

Weapon.cpp

HealthComponent.cpp
```

---

# Header 与 CPP

推荐：

```
Weapon.h

Weapon.cpp
```

不要把实现写进 Header。

例如：

错误：

```cpp
int Add(int A,int B)
{
    return A+B;
}
```

应该：

```cpp
// Header

int Add(int A,int B);
```

CPP：

```cpp
int UMath::Add(int A,int B)
{
    return A+B;
}
```

这样可以减少编译依赖。

---

# Include 的原则

不要：

```cpp
#include "Everything.h"
```

推荐：

```cpp
#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
```

保持 Include 最少。

---

# Forward Declaration

Forward Declaration（前向声明）是 UE 项目中最重要的技巧之一。

例如：

错误：

```cpp
#include "InventoryComponent.h"

class APlayerCharacter
{
    InventoryComponent* Inventory;
};
```

实际上只需要：

```cpp
class UInventoryComponent;
```

即可：

```cpp
class UInventoryComponent;

class APlayerCharacter
{
    UInventoryComponent* Inventory;
};
```

真正需要 Include 的地方：

```cpp
#include "InventoryComponent.h"
```

放在：

```
PlayerCharacter.cpp
```

这样可以：

- 降低编译时间
- 减少依赖
- 避免循环引用

---

# Pointer

UE 本质仍然是 C++。

因此：

```cpp
AActor* Actor;
```

表示：

```
Actor
   │
   ▼
Heap Object
```

Actor 本身存在堆上。

变量保存的是地址。

---

访问：

```cpp
Actor->Destroy();
```

而不是：

```cpp
Actor.Destroy();
```

---

# Reference

引用：

```cpp
FVector& Location
```

特点：

- 必须初始化
- 不允许为空
- 更安全

例如：

```cpp
void Move(FVector& Position)
{
    Position.X += 100;
}
```

相比：

```cpp
void Move(FVector* Position)
```

无需：

```cpp
if(Position)
```

---

# Const

推荐：

```cpp
const FString& Name
```

而不是：

```cpp
FString Name
```

避免：

- 大对象复制
- 临时对象

例如：

```cpp
void SetName(const FString& NewName);
```

---

返回值：

```cpp
const UTexture2D* GetIcon() const;
```

表示：

- 不修改对象
- 返回对象也不可修改

---

# nullptr

现代 C++：

推荐：

```cpp
Actor = nullptr;
```

不要：

```cpp
Actor = NULL;
```

也不要：

```cpp
Actor = 0;
```

---

# Auto

适用于：

```cpp
auto Player = GetWorld()->GetFirstPlayerController();
```

但不要：

```cpp
auto X = 3;
```

导致类型不明确。

---

# Enum Class

推荐：

```cpp
enum class EWeaponType
{
    Rifle,
    Pistol,
    Rocket
};
```

避免：

```cpp
enum WeaponType
```

使用：

```cpp
if(Type == EWeaponType::Rifle)
```

更加安全。

---

# Move Semantics

例如：

```cpp
TArray<FString> Names;

Names.Add(MoveTemp(Name));
```

而不是：

```cpp
Names.Add(Name);
```

减少复制。

UE 中大量使用：

```
MoveTemp()
```

---

# UE Containers

常见容器：

| 类型 | 类似 STL | 用途 |
|------|----------|------|
| TArray | vector | 动态数组 |
| TMap | unordered_map | 哈希 |
| TSet | unordered_set | 集合 |
| FString | string | 字符串 |
| FName | interned string | 高性能名称 |
| FText | localized text | 本地化文本 |

---

## FString

普通字符串：

```cpp
FString PlayerName;
```

支持：

```cpp
Append()

Printf()

Replace()
```

---

## FName

适用于：

- Socket Name
- Bone Name
- Gameplay Tag
- Asset Name

特点：

比较速度极快。

例如：

```cpp
FName("Head")
```

比：

```cpp
FString("Head")
```

更加适合作为 Key。

---

## FText

用于：

- UI
- Localization

例如：

```cpp
FText::FromString("Hello")
```

而不要：

```cpp
FString
```

用于 UI。

---

# Reflection System

Reflection 是 Unreal 最重要的系统之一。

它支持：

- GC
- Blueprint
- Serialization
- Network
- Editor
- Details Panel

```
           UHT

            │

Reflection Metadata

            │

------------------------

Blueprint

GC

Replication

Editor

Serialization
```

---

# UObject

几乎所有可反射对象都继承：

```cpp
UObject
```

例如：

```
UActorComponent

UGameInstance

UUserWidget

USaveGame

UDataAsset
```

---

## UObject 特点

拥有：

- Reflection
- GC
- RTTI
- Serialization

没有：

- Transform
- Tick
- World

因此：

不是所有 UObject 都能放进场景。

---

# AActor

Actor 继承：

```
UObject

    │

AActor
```

增加：

- Transform
- Tick
- Networking
- Components

因此：

```
Actor
```

才能存在于世界。

---

# UActorComponent

Component：

```
UObject

      │

UActorComponent
```

负责：

- 可复用逻辑
- 组合设计

例如：

```
HealthComponent

InventoryComponent

WeaponComponent

InteractionComponent
```

推荐：

> 能写 Component，就不要全部写进 Character。

---

# UCLASS()

任何 UObject 派生类：

```cpp
UCLASS()
class MYGAME_API UInventoryComponent
    : public UActorComponent
{
    GENERATED_BODY()
};
```

宏会告诉 UHT：

> 请生成 Reflection 信息。

---

# GENERATED_BODY()

必须存在：

```cpp
GENERATED_BODY()
```

否则：

- Reflection 不工作
- Blueprint 不工作
- GC 不工作

---

# UPROPERTY()

示例：

```cpp
UPROPERTY(EditAnywhere, BlueprintReadWrite)
int32 Health;
```

作用：

- Editor
- Save
- GC
- Replication
- Blueprint

---

常见 Specifier：

| Specifier | 含义 |
|-----------|------|
| EditAnywhere | 编辑器可修改 |
| EditDefaultsOnly | 默认值可修改 |
| VisibleAnywhere | 只读 |
| BlueprintReadWrite | 蓝图读写 |
| BlueprintReadOnly | 蓝图只读 |
| Transient | 不保存 |
| Replicated | 网络同步 |

---

# UFUNCTION()

例如：

```cpp
UFUNCTION(BlueprintCallable)
void Fire();
```

Blueprint 即可调用。

常见 Specifier：

- BlueprintCallable
- BlueprintPure
- Server
- Client
- NetMulticast
- BlueprintImplementableEvent
- BlueprintNativeEvent

---

# USTRUCT()

用于：

- 数据结构
- SaveGame
- Config
- DataTable

例如：

```cpp
USTRUCT(BlueprintType)
struct FInventoryItem
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere)
    FName ItemID;

    UPROPERTY(EditAnywhere)
    int32 Count;
};
```

---

# UENUM()

推荐：

```cpp
UENUM(BlueprintType)
enum class EItemType : uint8
{
    Weapon,
    Consumable,
    Quest
};
```

Blueprint 自动支持下拉框。

---

# 本章小结

本部分介绍了：

- UE C++ 开发理念
- Hybrid Architecture
- 项目目录结构
- Include 与 Forward Declaration
- Pointer / Reference / Const
- Unreal 容器
- Reflection System
- UObject、Actor、Component
- UCLASS / UPROPERTY / UFUNCTION / USTRUCT / UENUM

下一部分将进入：

- UObject 生命周期
- Garbage Collection
- TObjectPtr
- TWeakObjectPtr
- TSoftObjectPtr
- Gameplay Framework（GameMode、GameState、PlayerController、Pawn、Character、Subsystem）

# UObject 生命周期与内存管理

## UObject 的创建方式

Unreal Engine 中，大多数 UObject 都不能直接通过 `new` 创建，而应使用引擎提供的工厂函数，以便对象能够参与反射、垃圾回收和序列化。

### NewObject

适用于普通 UObject：

```cpp
UMyObject* Obj = NewObject<UMyObject>(Outer);
```

常见用途：

- 数据对象
- Manager
- Runtime Helper
- 非场景对象

---

### CreateDefaultSubobject

仅能在构造函数中使用，用于创建组件。

```cpp
AMyCharacter::AMyCharacter()
{
    Camera = CreateDefaultSubobject<UCameraComponent>(TEXT("Camera"));
}
```

特点：

- 自动注册
- 自动参与 GC
- 可以在 Blueprint 中继承修改

---

### SpawnActor

Actor 必须通过 World 创建。

```cpp
AEnemy* Enemy = GetWorld()->SpawnActor<AEnemy>(
    EnemyClass,
    SpawnTransform
);
```

不要：

```cpp
new AEnemy();
```

---

## Outer

每个 UObject 都有一个 Outer。

例如：

```
GameInstance
    └── InventoryManager
            └── InventoryItem
```

Outer 表示对象的拥有关系：

- 生命周期管理
- 路径组成
- GC 引用链

需要注意的是，Outer **不等于所有权**，真正决定对象是否被回收的是 GC 的引用可达性。

---

## Garbage Collection

UE 使用 **标记-清除（Mark & Sweep）** 垃圾回收机制。

简化流程：

```
GC Root
   │
   ├── Object A
   │      └── Object B
   │
   └── Object C
```

GC 会从 Root 开始遍历：

- 可达对象：保留
- 不可达对象：销毁

因此，一个 UObject 如果没有任何可达引用，最终会被 GC 回收。

---

## UPROPERTY 与 GC

推荐：

```cpp
UPROPERTY()
UMyObject* Inventory;
```

不要：

```cpp
UMyObject* Inventory;
```

原因：

- UPROPERTY 会参与反射
- GC 能识别引用关系
- Blueprint 可访问（取决于修饰符）

裸指针不会自动成为 GC 引用。

---

## TObjectPtr

UE5 推荐使用：

```cpp
UPROPERTY()
TObjectPtr<UStaticMeshComponent> Mesh;
```

优势：

- 更好的编辑器支持
- 更安全的对象访问
- 与未来引擎优化兼容

对于需要反射和长期保存的 UObject 引用，优先选择 `TObjectPtr`。

---

## TWeakObjectPtr

弱引用不会阻止对象被 GC。

```cpp
TWeakObjectPtr<AActor> Target;
```

使用前：

```cpp
if (Target.IsValid())
{
    Target->Destroy();
}
```

适用场景：

- AI Target
- UI 引用 Actor
- 缓存对象

---

## TSoftObjectPtr

软引用保存资源路径，而不是立即加载资源。

```cpp
UPROPERTY(EditAnywhere)
TSoftObjectPtr<UTexture2D> Icon;
```

优点：

- 延迟加载
- 减少启动内存
- Asset Manager 友好

---

## TSoftClassPtr

软引用 Blueprint Class。

```cpp
UPROPERTY(EditDefaultsOnly)
TSoftClassPtr<AEnemy> EnemyClass;
```

运行时再加载：

```cpp
EnemyClass.LoadSynchronous();
```

---

## TSubclassOf

限制 Blueprint 可选择的类型。

```cpp
UPROPERTY(EditDefaultsOnly)
TSubclassOf<AWeapon> WeaponClass;
```

编辑器中只能指定 `AWeapon` 或其子类。

---

# Actor 生命周期

Actor 从创建到销毁通常经历以下阶段：

```text
Constructor
      │
PostInitializeComponents
      │
BeginPlay
      │
Tick（可选）
      │
EndPlay
      │
Destroy
```

---

## Constructor

构造函数中适合：

- 创建默认组件
- 设置默认值

避免：

- 访问 World
- 获取 PlayerController
- 加载运行时资源

---

## BeginPlay

对象进入世界后调用。

适合：

- 初始化运行逻辑
- 注册事件
- 获取其他 Actor

```cpp
void AEnemy::BeginPlay()
{
    Super::BeginPlay();
}
```

---

## Tick

每帧执行。

```cpp
PrimaryActorTick.bCanEverTick = true;
```

如无必要，应关闭 Tick：

```cpp
PrimaryActorTick.bCanEverTick = false;
```

优先使用：

- Timer
- Delegate
- Event

降低每帧开销。

---

## EndPlay

Actor 离开世界时调用。

适合：

- 注销委托
- 停止 Timer
- 清理资源

---

# Gameplay Framework

Gameplay Framework 定义了游戏各类对象的职责。

```text
GameInstance
      │
    World
      │
 GameMode ─── GameState
      │
PlayerController
      │
     Pawn
      │
   Character
```

---

## GameInstance

整个应用生命周期唯一。

适合保存：

- 登录状态
- 全局配置
- 在线服务
- 跨关卡数据

不会因关卡切换而销毁。

---

## GameMode

仅服务器存在。

职责：

- 游戏规则
- 玩家出生
- 胜负条件

客户端无法直接访问。

---

## GameState

所有客户端同步。

适合：

- 比赛时间
- 当前分数
- 回合状态

---

## PlayerController

代表玩家输入。

职责：

- 输入处理
- Camera 控制
- UI 管理
- Possess Pawn

一个玩家通常对应一个 PlayerController。

---

## Pawn

可被控制的实体。

例如：

- 车辆
- 飞机
- 无人机

---

## Character

Character 继承自 Pawn，并内置：

- CapsuleComponent
- CharacterMovementComponent
- SkeletalMeshComponent

适用于大多数角色游戏。

---

## PlayerState

保存玩家状态。

例如：

- 玩家名称
- 分数
- Ping
- 队伍

支持网络同步。

---

## HUD

负责传统 HUD 绘制。

现代项目更多使用 UMG，但 HUD 仍可作为 UI 管理入口。

---

# Subsystem

Subsystem 用于构建全局管理模块，避免大量 Singleton。

常见类型：

| 类型 | 生命周期 |
|------|----------|
| EngineSubsystem | 引擎级 |
| EditorSubsystem | 编辑器级 |
| GameInstanceSubsystem | 游戏级 |
| WorldSubsystem | 世界级 |
| LocalPlayerSubsystem | 本地玩家级 |

例如：

```cpp
UInventorySubsystem
UQuestSubsystem
UAudioSubsystem
```

相比自定义 Singleton，Subsystem 生命周期更清晰，也更易测试和扩展。

---

# 本章小结

本章介绍了：

- UObject 创建方式
- NewObject / CreateDefaultSubobject / SpawnActor
- Outer 与对象关系
- Garbage Collection
- UPROPERTY 与对象引用
- TObjectPtr / TWeakObjectPtr / TSoftObjectPtr
- TSubclassOf
- Actor 生命周期
- Gameplay Framework 各核心类职责
- Subsystem 架构

下一部分将深入介绍：

- Delegate（委托）
- Interface（接口）
- Timer
- Asset Manager
- 异步资源加载
- Modules 与 Build.cs
- Logging 与调试技巧

# Delegate（委托）

Delegate 是 Unreal Engine 中最重要的事件通信机制之一，可用于对象之间的解耦。

相比直接持有对象引用，Delegate 能够：

- 降低模块耦合
- 支持广播
- 支持 Blueprint
- 更易扩展

---

## Delegate 类型

| 类型 | 是否有返回值 | 是否支持多个绑定 | Blueprint |
|-------|--------------|------------------|------------|
| DECLARE_DELEGATE | ✔ | ✘ | ✘ |
| DECLARE_MULTICAST_DELEGATE | ✘ | ✔ | ✘ |
| DECLARE_DYNAMIC_DELEGATE | ✔ | ✘ | ✔ |
| DECLARE_DYNAMIC_MULTICAST_DELEGATE | ✘ | ✔ | ✔ |

---

## 单播 Delegate

```cpp
DECLARE_DELEGATE(FOnWeaponFire);

FOnWeaponFire OnWeaponFire;
```

绑定：

```cpp
OnWeaponFire.BindUObject(this, &AMyCharacter::Fire);
```

执行：

```cpp
if (OnWeaponFire.IsBound())
{
    OnWeaponFire.Execute();
}
```

---

## 多播 Delegate

```cpp
DECLARE_MULTICAST_DELEGATE(FOnHealthChanged);
```

绑定多个对象：

```cpp
OnHealthChanged.AddUObject(this, &UMyWidget::UpdateUI);

OnHealthChanged.AddUObject(this, &UMyAudio::PlaySound);
```

广播：

```cpp
OnHealthChanged.Broadcast();
```

所有监听者都会收到通知。

---

## Dynamic Multicast Delegate

用于 Blueprint：

```cpp
DECLARE_DYNAMIC_MULTICAST_DELEGATE(FOnDead);
```

声明：

```cpp
UPROPERTY(BlueprintAssignable)
FOnDead OnDead;
```

Blueprint：

```
Character

↓

OnDead

↓

Play Animation

↓

Spawn FX

↓

Play Audio
```

这是 Blueprint Event Dispatcher 的底层机制。

---

# Interface

Interface 用于描述能力（Capability），而不是继承关系。

例如：

```
Character

Door

Chest

NPC
```

都可能支持：

```
Interact()
```

不应该都继承同一个父类。

---

## 创建 Interface

```cpp
UINTERFACE(BlueprintType)
class UInteractable : public UInterface
{
    GENERATED_BODY()
};
```

接口：

```cpp
class IInteractable
{
    GENERATED_BODY()

public:

    virtual void Interact() = 0;
};
```

---

## 调用

推荐：

```cpp
if (Actor->Implements<UInteractable>())
{
    IInteractable::Execute_Interact(Actor);
}
```

不要：

```cpp
Cast<IInteractable>(Actor)
```

Blueprint Interface 应优先使用 `Execute_XXX()`。

---

# Timer

不要为了等待几秒而使用 Tick。

推荐：

```cpp
GetWorld()->GetTimerManager().SetTimer(
    FireHandle,
    this,
    &AMyCharacter::Reload,
    2.f,
    false
);
```

---

循环 Timer：

```cpp
SetTimer(
    Handle,
    this,
    &AMyCharacter::RecoverHP,
    1.f,
    true
);
```

停止：

```cpp
ClearTimer(Handle);
```

---

# Asset 引用

UE 中主要有三种资源引用方式：

```
Hard Reference

Soft Reference

Primary Asset
```

---

## Hard Reference

例如：

```cpp
UPROPERTY(EditAnywhere)
UTexture2D* Icon;
```

特点：

- 自动加载
- 简单
- 内存占用较高

适用于：

- 必定需要的资源

---

## Soft Reference

```cpp
UPROPERTY(EditAnywhere)
TSoftObjectPtr<UTexture2D> Icon;
```

不会立即加载资源。

优点：

- 减少启动时间
- 节省内存
- 支持按需加载

---

## Asset Manager

大型项目推荐统一管理资源。

例如：

```
Weapons

Characters

Abilities

Items
```

都可以作为：

```
Primary Asset
```

管理。

好处：

- 按需加载
- 批量加载
- 支持 DLC
- 易于资源分类

---

## 异步加载

推荐：

```cpp
StreamableManager.RequestAsyncLoad(
    Asset.ToSoftObjectPath(),
    FStreamableDelegate::CreateLambda(
        []()
        {
            UE_LOG(LogTemp, Log, TEXT("Loaded"));
        })
);
```

避免同步加载导致卡顿。

---

# Module

大型 UE 项目通常拆分多个 Module。

例如：

```
MyGame

Inventory

Combat

AI

UI

Online
```

每个 Module：

```
Public/

Private/

Build.cs
```

互相通过依赖关系协作。

---

# Build.cs

典型配置：

```csharp
PublicDependencyModuleNames.AddRange(
new string[]
{
    "Core",
    "CoreUObject",
    "Engine"
});
```

如果使用：

- UMG
- GameplayTasks
- AIModule

需要添加对应模块依赖。

---

# 日志系统

推荐使用分类日志。

```cpp
DEFINE_LOG_CATEGORY(LogInventory);
```

输出：

```cpp
UE_LOG(
    LogInventory,
    Warning,
    TEXT("Item Count=%d"),
    Count
);
```

避免长期使用：

```cpp
LogTemp
```

---

## 日志等级

```text
Fatal

Error

Warning

Display

Log

Verbose

VeryVerbose
```

开发阶段可提高 Verbose 等级帮助排查问题。

---

# Assert

检查不可恢复错误：

```cpp
check(Player);
```

失败会中断程序。

---

非致命检查：

```cpp
ensure(Player);
```

仅记录错误并继续运行。

适用于：

- 调试
- 非关键逻辑验证

---

# Cast

推荐：

```cpp
ACharacter* Character =
Cast<ACharacter>(Actor);
```

不要：

```cpp
static_cast
```

用于 UObject 类型。

原因：

Cast 会利用 UE Reflection 判断类型安全。

---

# 常见编码建议

## 优先组合，而不是继承

推荐：

```
Character

├── InventoryComponent

├── HealthComponent

├── AbilityComponent

└── InteractionComponent
```

而不是：

```
Character

↓

CombatCharacter

↓

InventoryCharacter

↓

MagicCharacter

↓

BossCharacter
```

组件化更易维护。

---

## 减少 Tick

优先选择：

- Delegate
- Timer
- Event
- Gameplay Ability
- Animation Notify

只有确实需要每帧更新时再开启 Tick。

---

## 最小化 Include

Header：

```cpp
class UInventoryComponent;
```

CPP：

```cpp
#include "InventoryComponent.h"
```

这样可以显著减少编译时间。

---

## 避免 God Object

不要把所有逻辑都放进：

```
Character.cpp
```

超过几千行代码通常意味着需要拆分。

建议：

- InventoryComponent
- EquipmentComponent
- InteractionComponent
- AttributeComponent
- QuestComponent

---

# 推荐项目结构

```text
Source/

    MyGame/

        Public/

        Private/

    Inventory/

        Public/

        Private/

    Combat/

    UI/

    AI/

    Online/

    Save/

Content/

Config/

Plugins/
```

随着项目规模增长，可进一步拆分为独立插件（Plugins），实现更高的模块复用。

---

# 本章小结

本章介绍了：

- Delegate 与事件驱动
- Interface 设计
- Timer 的正确使用
- Asset 引用与 Asset Manager
- 异步资源加载
- Module 与 Build.cs
- 日志系统
- Cast 与类型安全
- 组件化设计原则
- 大型项目目录组织

下一部分将重点介绍：

- Profiling 与性能优化
- 常见性能陷阱
- Networking 最佳实践
- SaveGame 架构
- Coding Standard
- UE C++ 开发最佳实践总结
- 推荐学习资源

# Performance Profiling（性能分析）

性能优化应遵循一个原则：

> **先测量，再优化（Measure First, Optimize Second）。**

不要凭感觉优化代码，而应借助 Unreal Engine 提供的分析工具定位瓶颈。

---

## Unreal Insights

Unreal Insights 是 UE 官方性能分析工具。

适合分析：

- CPU Timeline
- Game Thread
- Render Thread
- Async Task
- Memory
- Asset Loading

推荐流程：

```
发现卡顿
      │
Capture Trace
      │
Unreal Insights
      │
定位瓶颈
      │
优化
```

---

## Stat Commands

开发过程中常用：

```text
stat fps
stat unit
stat game
stat gpu
stat memory
stat anim
stat ai
```

例如：

```text
stat unit
```

可以看到：

```
Frame

Game

Draw

GPU
```

快速判断瓶颈来自 CPU 还是 GPU。

---

## 常见性能问题

### Tick 太多

例如：

```
500 Actor

↓

500 Tick

↓

每秒 30000 次 Tick
```

解决方案：

- Timer
- Delegate
- Event
- AI Perception
- Animation Notify

---

### GetAllActorsOfClass

不要频繁调用：

```cpp
UGameplayStatics::GetAllActorsOfClass(...)
```

尤其不要在 Tick 中调用。

推荐：

- Subsystem 管理
- 注册机制
- Gameplay Tag 查询
- Manager 缓存

---

### Cast 过多

例如：

```cpp
Tick()

↓

Cast()

↓

Cast()

↓

Cast()
```

频繁 Cast 会增加运行时开销。

推荐：

BeginPlay 缓存：

```cpp
CachedPlayer = Cast<AMyPlayer>(GetPlayerPawn());
```

---

### 动态加载资源

避免：

```cpp
LoadObject()
```

出现在：

```
Tick

BeginPlay

Animation Tick
```

推荐：

Asset Manager

Soft Reference

Async Loading

---

# Networking

UE 网络采用：

```
Server Authority
```

模型。

```
Client

      │

RPC

      │

Server

      │

Replication

      │

Client
```

---

## Replication

变量：

```cpp
UPROPERTY(Replicated)
int32 Health;
```

需要：

```cpp
GetLifetimeReplicatedProps(...)
```

注册。

---

## RepNotify

例如：

```cpp
UPROPERTY(ReplicatedUsing=OnRep_Health)
int32 Health;
```

同步后：

```cpp
OnRep_Health()
```

自动调用。

适合：

- 更新 UI
- 播放特效
- 更新动画

---

## RPC

主要分三类：

Server：

```cpp
UFUNCTION(Server,Reliable)
```

Client：

```cpp
UFUNCTION(Client,Reliable)
```

Multicast：

```cpp
UFUNCTION(NetMulticast,Reliable)
```

推荐：

```
Input

↓

Server

↓

Validate

↓

Game Logic

↓

Replication

↓

Clients
```

不要相信客户端。

---

# SaveGame

推荐：

```
USaveGame
```

例如：

```cpp
UCLASS()
class UMySaveGame
: public USaveGame
{
    GENERATED_BODY()
};
```

保存：

```
Player

Inventory

Quest

Level

Settings
```

不要直接保存 Actor。

推荐保存：

```
ID

Transform

Data
```

重新生成对象。

---

# Data Driven Design

推荐：

```
Weapon

↓

DataAsset

↓

Blueprint

↓

Runtime
```

而不是：

```
Weapon.cpp

↓

if

↓

switch

↓

switch

↓

switch
```

数据驱动：

- 易扩展
- 美术可配置
- 无需修改代码

---

# Gameplay Tags

推荐：

```
Weapon.Rifle

Weapon.Pistol

Enemy.Boss

Ability.Fire
```

不要：

```
if(Name=="Boss")
```

Gameplay Tag：

- 更安全
- 可组合
- 支持查询

---

# Coding Standard

推荐遵循 Epic Coding Standard。

例如：

类：

```cpp
AMyCharacter
```

UObject：

```cpp
UMyInventory
```

Struct：

```cpp
FInventoryItem
```

Interface：

```cpp
IInteractable
```

Enum：

```cpp
EWeaponType
```

Boolean：

```cpp
bIsDead
```

---

变量：

```cpp
Health

CurrentAmmo

MaxAmmo
```

函数：

```cpp
Reload()

TakeDamage()

BeginPlay()
```

保持：

- 动词
- 可读

---

# 注释原则

不要：

```cpp
// Add one
A++;
```

推荐：

解释原因：

```cpp
// Delay weapon activation until animation finishes.
```

说明：

为什么这样做。

而不是：

做了什么。

---

# Architecture Best Practice

推荐：

```
Subsystem

↓

Manager

↓

Component

↓

Actor

↓

Widget
```

避免：

```
Widget

↓

Character

↓

Character

↓

Widget

↓

Manager

↓

Character
```

循环依赖。

---

推荐：

```
Character

↓

Component

↓

Delegate

↓

Widget
```

单向依赖。

---

# 推荐目录结构

```
Source/

    Core/

    Gameplay/

    Combat/

    Inventory/

    Interaction/

    UI/

    AI/

    Save/

    Online/

Plugins/

Content/

Config/
```

大型项目建议：

一个系统

=

一个 Module

---

# 常见错误

## 忘记 UPROPERTY

```cpp
UMyObject* Obj;
```

可能被 GC 回收。

---

## BeginPlay 做太多事情

不要：

```
Load

Spawn

Init

Search

Save

Network

UI
```

全部放进去。

推荐：

Subsystem

+

Manager

负责初始化。

---

## Character 超过五千行

意味着：

需要拆分 Component。

---

## Tick 中查找对象

不要：

```cpp
GetPlayerController()

GetWorld()

GetActorLocation()

Cast()
```

每帧重复获取。

推荐：

缓存。

---

# 学习路线

建议顺序：

```
Modern C++

↓

UE Reflection

↓

Gameplay Framework

↓

UObject

↓

GC

↓

Component

↓

Delegate

↓

Interface

↓

Asset Manager

↓

Networking

↓

GAS

↓

Slate

↓

Engine Source
```

---

# 推荐阅读

## Epic 官方

- Unreal Engine Documentation
- Unreal Engine API Reference
- Gameplay Framework
- Networking Guide
- Asset Manager

---

## 官方源码

建议阅读：

```
Actor

ActorComponent

Character

CharacterMovementComponent

GameModeBase

GameInstance

PlayerController
```

理解：

Epic 如何组织代码。

---

## 推荐项目

学习：

- Lyra Starter Game
- ActionRPG Sample
- ShooterGame（历史示例）

重点关注：

- 模块划分
- Gameplay Framework
- GAS
- Input
- UI

---

# UE C++ 开发核心原则

可以总结为以下十条：

1. 使用 C++ 构建框架，Blueprint 负责扩展。
2. 优先组合（Component），避免过深继承。
3. 使用 `UPROPERTY` 管理 UObject 引用。
4. 使用 `TObjectPtr`、`TWeakObjectPtr`、`TSoftObjectPtr` 管理不同生命周期的对象。
5. 减少 Tick，优先事件驱动。
6. 使用 Delegate 解耦模块。
7. 使用 Interface 表达能力，而不是滥用继承。
8. 使用 Soft Reference 和 Asset Manager 管理大型资源。
9. 通过 Unreal Insights 和 Stat 命令定位性能问题，而不是凭感觉优化。
10. 保持模块化和数据驱动设计，让系统更易维护和扩展。

---

# 结语

Unreal Engine 的 C++ 并不是为了取代 Blueprint，而是为了构建稳定、可维护、可扩展的底层架构。

优秀的 UE 项目通常遵循以下分工：

```
                Designer
                    │
             Blueprint / DataAsset
                    │
             Gameplay Components
                    │
            Gameplay Framework
                    │
             Engine Runtime
```

当你能够熟练掌握 UObject 生命周期、反射系统、Gameplay Framework、组件化设计、事件驱动、资源管理以及网络同步后，就已经具备了开发中大型 Unreal Engine 项目的核心能力。

后续可以进一步深入：

- Gameplay Ability System（GAS）
- Slate UI
- Editor Extension
- Rendering Pipeline
- Animation System
- Mass Entity
- Chaos Physics
- Unreal Engine Source Code