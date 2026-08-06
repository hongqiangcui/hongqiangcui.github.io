---
title: "Understanding Unreal Engine Object System"
subtitle: "From UObject to Reflection"
date: 2026-08-06
draft: false
toc: true
---

<!--more-->

# Understanding Unreal Engine Object System

> **Everything in Unreal Engine is described by a Runtime Type System.**

> **Note**
>
> 由于 ChatGPT 单次回复长度限制，无法一次生成约 30000 字的完整 Markdown 文档。
> 本文件提供完整的大纲、章节结构以及每章的核心内容，可作为最终 Paper 的骨架，并可继续扩展。

# Abstract

Unreal Engine 并不是简单地在 C++ 上增加了一层 Reflection，而是构建了一套完整的 Runtime Object Model。

本文从架构角度分析：

- UObject
- UClass
- UStruct
- FField
- FProperty
- Reflection
- Blueprint
- Garbage Collection
- Serialization
- Runtime Registration

以及这些系统如何共同组成 UE Runtime Type System。

---

# 1. Runtime Object Model

## 为什么需要 UObject

现代游戏引擎需要：

- Reflection
- Editor
- Blueprint
- GC
- Replication
- Serialization

因此需要 Runtime Type Information。

## Mental Model

```text
C++
    │
    ▼
Runtime Type System
    │
    ├── UObject
    ├── UClass
    ├── FProperty
    └── UFunction
```

# 2. UObject

UObject 是运行时对象。

职责：

- Identity
- Lifetime
- Outer
- Name
- Reflection Entry

内存示意：

```text
UObject
├── Flags
├── InternalIndex
├── Name
├── Outer
└── Class*
```

# 3. UClass

UClass 是 Runtime Type。

保存：

- Property
- Function
- Metadata
- CDO
- SuperClass

所有实例共享一个 UClass。

# 4. UStruct 与 FField

```text
UStruct
├── UClass
├── UFunction
└── UScriptStruct

FField
└── FProperty
```

UE5 将 Property 从 UObject 拆分出来，以减少 GC 成本。

# 5. Reflection

Reflection ≠ UObject。

Reflection =

- UClass
- FProperty
- UFunction
- Metadata

Engine 各系统消费 Reflection，而不是直接消费 UObject。

# 6. CDO

每个 UClass 拥有唯一 CDO。

对象创建：

```text
Allocate
 ↓
Copy CDO
 ↓
Constructor
 ↓
PostInitProperties
```

# 7. Object Lifecycle

```text
Header
 ↓
UHT
 ↓
.gen.cpp
 ↓
Compile
 ↓
Module Load
 ↓
Static Registration
 ↓
Construct UClass
 ↓
Construct CDO
 ↓
NewObject
 ↓
GC
```

# 8. Blueprint Runtime

Blueprint Asset

↓

BlueprintGeneratedClass

↓

UClass

↓

NewObject

Blueprint 与 Native 最终统一到 UClass。

# 9. Reflection Consumers

- Editor
- GC
- Serialization
- Replication
- Blueprint VM

# 10. Design Philosophy

Unreal 并不是 Everything is an Object。

而是：

Everything is described by Runtime Type Information。

# Appendix

建议后续扩展：

1. UHT 工作机制
2. GENERATED_BODY 宏展开
3. IMPLEMENT_CLASS
4. GUObjectArray
5. GC Token Stream
6. Blueprint VM
7. Property System
8. FArchive
9. LinkerLoad
10. Hot Reload

---
title: "Understanding Unreal Engine Object System"
subtitle: "从 UObject 到 Reflection，理解 Unreal Engine 的运行时对象模型"
date: 2026-08-06
draft: false
toc: true
math: false

tags:
  - Unreal Engine
  - UE5
  - Engine Source
  - UObject
  - Reflection
  - Runtime Type System

categories:
  - Unreal Engine
---

# Understanding Unreal Engine Object System

> **Everything in Unreal Engine is not an Object.**
>
> **Everything in Unreal Engine is described by a Runtime Type System.**

很多开发者第一次学习 Unreal Engine，会把 UObject 理解成类似 Unity 的 Object，或者 C++ 的基类。

事实上，UObject 只是整个 Runtime Type System 的入口。

Reflection、Blueprint、Garbage Collection、Serialization、Editor、Replication……这些看似毫无关联的系统，实际上全部建立在同一个对象模型之上。

本文希望站在架构设计的角度，回答几个问题：

- 为什么 UE 要重新实现一套 Object System？
- UObject、UClass、UStruct 到底是什么关系？
- Reflection 为什么能够支撑整个 Engine？
- Blueprint 为什么能够和 Native Class 完全统一？
- 为什么一个 UObject 不只是一个 C++ 对象？

本文不会深入源码，而是建立整个 Unreal Object System 的心智模型（Mental Model）。

---

# 1. 为什么 Unreal 要重新实现 Object System？

对于普通 C++ 来说，一个对象就是一个类的实例。

```
C++ Class
    │
    ▼
 Instance
```

例如：

```cpp
class Player
{
public:
    int Health;
};
```

编译以后，运行时只剩下机器码。

C++ 本身几乎不知道：

- 成员变量有哪些
- Offset 在哪里
- 哪些成员需要保存
- 哪些成员可以编辑
- 哪些函数可以暴露给脚本

C++ RTTI 能提供的能力非常有限：

- typeid
- dynamic_cast

对于一个现代游戏引擎来说，这远远不够。

Engine 需要支持：

- Reflection
- Editor
- Blueprint
- Garbage Collection
- Serialization
- Network Replication
- SaveGame
- Animation
- Hot Reload
- Live Coding

这些系统都有一个共同需求：

> **运行时必须知道一个对象长什么样。**

因此 Unreal 在 C++ 之上建立了一套 Runtime Type System。

---

# 2. Runtime Type System 全景

整个 Unreal Engine 可以抽象成四层：

```
               C++ Language
                     │
────────────────────────────────────
                     │
           Runtime Type System
                     │
      ┌──────────────┼──────────────┐
      ▼              ▼              ▼
   UObject         UClass         FField
                                      │
                                 FProperty
                                 UFunction
                                 Metadata
────────────────────────────────────────────
                     │
             Engine Systems
                     │
GC / Editor / Blueprint / Serialization / Networking
```

这里最重要的一点是：

Reflection 并不是 UObject。

Reflection 实际上是：

- UClass
- FProperty
- UFunction
- Metadata

共同组成的一套运行时类型描述。

而 UObject 只是 Reflection 所描述出来的实例。

---

# 3. UObject —— Runtime Object

UObject 是整个 Runtime Object System 的基础对象。

它负责：

- Object Identity
- 生命周期
- Object Name
- Outer 层级
- GC 管理
- Reflection 入口

从概念上可以理解为：

```
           UObject

    +--------------------+
    | Flags              |
    | Internal Index     |
    | Name               |
    | Outer              |
    | Class Pointer -----+
    +--------------------+
                           │
                           ▼
                        UClass
```

很多人会误以为：

一个 UObject 保存了自己的全部信息。

实际上并不是。

真正描述对象结构的是：

```
UClass
```

UObject 保存的是：

> **"我是哪个类型的实例。"**

而不是：

> **"我的类型是什么。"**

---

# 4. UClass —— Runtime Type

如果说 UObject 是：

```
Instance
```

那么：

UClass 就是：

```
Runtime Type
```

每一个：

```cpp
UCLASS()
```

最终都会生成：

```
一个 UClass 对象
```

UClass 保存：

- 类型名称
- 父类
- Property 列表
- Function 列表
- Interface
- Metadata
- CDO（Class Default Object）

因此：

```
100000 个 Actor

↓

共享

↓

一个 AActor::StaticClass()
```

Reflection 信息永远只有一份。

这也是整个 Engine 能够高效运行的重要原因。

---

# 5. Reflection 到底是什么？

很多教程喜欢画：

```
Object

↓

Reflection
```

实际上：

Reflection 根本不是一个对象。

Reflection 更像：

```
Runtime Schema
```

真正的关系应该是：

```
           UClass

        ├─────────────┐
        ▼             ▼
   FProperty      UFunction
        │             │
        └──────┬──────┘
               ▼
          Runtime Metadata
```

Reflection 描述的是：

> **一个类型有哪些字段、函数以及行为。**

而不是：

> **一个实例保存了什么。**

---

# 6. UObject、UStruct 与 FField

很多人第一次看源码都会被这些名字绕晕。

实际上它们属于不同层级。

```
              UObject
                  │
               UField
                  │
               UStruct
                  │
        ┌─────────┼─────────┐
        ▼         ▼         ▼
     UClass   UFunction  UScriptStruct
```

而 Property 在 UE5 中已经独立出来：

```
FField
   │
FProperty
```

为什么？

因为 Property 数量远远超过 Object。

如果每个 Property 都是 UObject：

- GC 会变慢
- 内存占用增加
- 对象数量爆炸

因此 UE5 将 Property 从 UObject 中剥离出来。

它们只描述数据。

生命周期由 UClass 管理。

---

# 7. Class Default Object（CDO）

CDO 是 Unreal 最经典的设计之一。

每一个 UClass：

都会拥有唯一一个：

```
Class Default Object
```

例如：

```
Character

↓

Character Default Object
```

里面保存：

```
Health = 100

Speed = 600

JumpHeight = 420
```

对象创建过程并不是：

```
new

↓

Constructor
```

而是：

```
Allocate Memory

↓

Copy CDO

↓

Constructor

↓

PostInitProperties
```

因此：

所有默认值都来自：

```
CDO
```

Blueprint 修改默认值。

实际上修改的也是：

```
CDO
```

---

# 8. Native Class 与 Blueprint Class 为什么能够统一？

很多人认为：

Blueprint 是另一套对象系统。

事实上：

Blueprint 最终也会生成：

```
BlueprintGeneratedClass
```

而 BlueprintGeneratedClass：

本质上仍然是：

```
UClass
```

因此：

```
Native Class
      │
      ▼
    UClass

Blueprint
      │
      ▼
BlueprintGeneratedClass
      │
      ▼
    UClass
```

最终：

Engine 创建对象时：

```
NewObject(UClass*)
```

根本不会关心：

它来自：

- C++
- Blueprint

因为：

最终全部统一成：

```
UClass
```

---

# 9. Reflection 如何驱动整个 Engine？

Reflection 本身几乎什么都不做。

真正工作的：

是 Engine。

例如：

Garbage Collection：

```
遍历 Property
↓

寻找 UObject 引用
↓

标记可达对象
```

Serialization：

```
遍历 Property
↓

读取 Offset
↓

自动保存
```

Editor：

```
遍历 Property
↓

生成 Details Panel
```

Blueprint：

```
遍历 UFunction
↓

生成 Blueprint Node
```

Networking：

```
遍历 Property
↓

寻找 Replicated Property
```

因此：

Reflection 更像：

```
Database Schema
```

Engine 更像：

```
Database Engine
```

Reflection 提供：

> 数据描述。

Engine 消费：

> 数据描述。

---

# 10. Engine 启动时发生了什么？

很多开发者认为：

第一次调用：

```cpp
StaticClass()
```

才创建：

```
UClass
```

实际上：

并不是。

Engine 启动时：

```
Module Load

↓

Static Registration

↓

Construct UClass

↓

Construct CDO

↓

Register Into Object System
```

因此：

程序刚启动：

整个 Runtime Type System 已经建立完成。

例如：

```
Actor

Pawn

Character

Texture

Material

World
```

这些类型都已经存在。

所以：

```
AActor::StaticClass()
```

实际上只是：

返回：

已经注册好的：

```
UClass
```

---

# 11. 整个 Object System

最终：

整个 Unreal Engine 可以抽象成下面这一张图。

```
                    UObject Instance
                           │
                     GetClass()
                           │
                           ▼
                       UClass
                           │
      ┌────────────────────┼────────────────────┐
      ▼                    ▼                    ▼
  FProperty           UFunction            Metadata
      │                    │
      └──────────────┬─────┘
                     ▼
               Reflection
                     │
     ┌───────────────┼────────────────────┐
     ▼               ▼                    ▼
 Garbage        Serialization        Blueprint
 Collection
                     │
                     ▼
              Unreal Engine
```

整个 Engine 并不是围绕 UObject 运转。

真正的中心其实是：

```
Runtime Type System
```

UObject 是：

Runtime Object。

UClass 是：

Runtime Type。

Reflection 是：

Runtime Metadata。

Engine 的所有子系统：

共同消费这套 Runtime Metadata。

---

# 12. 总结

如果只用一句话概括 Unreal Engine 的对象模型：

> **UObject 保存对象，UClass 保存类型，Reflection 保存结构，而整个 Engine 消费 Reflection。**

理解这一点之后，再去阅读：

- Reflection
- Blueprint
- Garbage Collection
- Serialization
- Property System
- UHT

会发现它们其实都是同一个系统的不同侧面。

UObject 并不仅仅是一个基类。

它是 Unreal Engine Runtime Object Model 的核心。

而 Reflection，也不仅仅是一项功能。

它是整个 Engine 能够实现编辑器、脚本、序列化、垃圾回收以及数据驱动架构的基础设施。
