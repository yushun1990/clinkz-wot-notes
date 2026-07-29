---
id: "INDEX-001"
title: "专栏导读｜从零开发一个 Rust WoT 引擎"
subtitle: "ClinkZ-WoT 的设计、实现与 AI 协作开发记录"
series: "专栏导读"
series_order: 0
status: "DRAFTING"
author: "yushun1990"
created: "2026-07-28"
updated: "2026-07-29"
summary: "介绍《从零开发一个 Rust WoT 引擎》专栏的写作目标、内容结构、推荐阅读路线和文章更新规则。"

clinkz_wot:
  repository: "https://github.com/yushun1990/clinkz-wot"
  branch: "master"
  commit: "f453f165c2ea775e5f0d10c36f1e419fcc1d79f3"
  inspected_at: "2026-07-29"

publication:
  platform: "zhihu"
  published_at: null
  canonical_url: null

related:
  previous: null
  next: "WOT-001"
  articles:
    - "WOT-001"
    - "ARCH-001"
    - "RUST-001"
    - "AI-001"
  docs:
    - "README.md"
    - "CONTENT_PLAN.md"
  source: []
  tests: []
---

# 专栏导读｜从零开发一个 Rust WoT 引擎

> 这是《从零开发一个 Rust WoT 引擎》的长期索引。
>
> 专栏中的文章会随着 ClinkZ-WoT 的开发持续增加和修订。每发布一篇新文章，我都会在这里补充链接、阅读顺序和对应的项目状态。

## 我为什么要写这个专栏

我正在开发一个名为 [ClinkZ-WoT](https://github.com/yushun1990/clinkz-wot) 的 Rust Web of Things Runtime。

它不是一个已经完成后再回头包装的“成功项目”，而是一个仍在真实演进中的工程：架构会发生冲突，接口会被推翻，计划会重新拆分，一些看起来聪明的方案也会在继续分析后被放弃。

我想把这个过程记录下来，主要有两个原因。

第一，是向更多物联网开发者介绍 W3C Web of Things。WoT 并不是一种替代 MQTT、HTTP、CoAP 或 Zenoh 的新协议，而是一套建立在具体协议之上的 Thing 描述、交互模型与运行时编程方式。

第二，也是更重要的原因，是逼迫自己真正理解这个项目。能让 AI 生成一段代码，不等于掌握了这段代码；能接受一份架构设计，也不等于理解了设计背后的状态、生命周期、失败路径和取舍。

所以这个专栏不会只展示结论。我会尽量说明：

- 最初遇到了什么现实问题；
- 最直觉的方案为什么有吸引力；
- 它会在哪些场景中失败；
- 不同方案的责任和复杂度分别落在哪里；
- ClinkZ-WoT 当前选择了什么边界；
- 这些边界最终如何落实为 Rust 类型、所有权、状态机和模块结构。

## ClinkZ-WoT 是什么

ClinkZ-WoT 希望实现一个以 [W3C Thing Description](https://www.w3.org/TR/wot-thing-description11/) 为语义入口、协议中立的 Rust WoT Runtime。

真实的物联网系统通常不是一次建成的。存量控制器、现场协议、厂商网关、消息平台、Web API 和后来增加的边缘服务，可能在不同阶段进入同一个系统。多协议、多厂商和多接口长期共存，往往是设备寿命、网络条件、采购边界和系统演进共同造成的结果，而不是为了说明 WoT 人为拼出的场景。

即使系统通过网关把通信统一为 MQTT 或 HTTP，应用仍然需要理解不同的 Topic、Payload、URL、数据含义、操作方式、错误模型和订阅语义。统一传输只能解决“消息怎样到达”的一部分问题，不能自动形成共同的设备能力模型。

真正缺少的不是又一种通信协议，而是位于这些接口之上的、机器可读的 Thing 交互模型。

WoT 尝试把几个层次分开：

```text
Thing Description
        |
        v
Property / Action / Event
        |
        v
Form + Protocol Binding
        |
        v
HTTP / MQTT / CoAP / Modbus / OPC UA / Zenoh / ...
```

Thing Description 描述 Thing 能做什么，Protocol Binding 负责如何通过某种协议完成交互，而 Servient Runtime 负责把语义、策略、生命周期和协议执行组织起来。

ClinkZ-WoT 的目标，就是探索这套模型如何在 Rust 中落地，并同时兼顾：

- 协议中立；
- 明确的资源所有权；
- 可验证的异步生命周期；
- 有界队列与资源预算；
- Host 与受限环境的共同语义；
- 可以长期演进的 Protocol Binding SPI。

## 这个专栏会写什么

专栏暂时分为四条主线。它们并不是彼此独立的教程，而是从不同角度解释同一个运行时。

### 系列一：重新理解 WoT

这一部分不会先给空调、传感器或控制器强行分配一种协议，再用虚构示例证明 WoT 的必要性。它会从真实系统为什么自然形成多协议、多厂商和多接口边界开始，逐步追问：通信已经打通以后，应用为什么仍然难以复用设备能力；把所有接口统一成 MQTT 或 HTTP 为什么仍然不够；WoT 又在哪一层介入。

会涉及：

- 存量设备、网络约束、采购边界和长期演进如何形成协议与接口碎片化；
- 统一传输协议为什么不等于统一设备能力、数据含义和交互语义；
- W3C WoT 为什么不是一种新协议，它试图建立什么共同接口；
- Thing Description 为什么是语义契约，而不是设备或运行时配置文件；
- Property、Action、Event、Data Schema 和 Form 如何描述 Thing 的能力与访问方式；
- 同一个 Thing 如何通过不同协议或多个 Form 交互；
- ConsumedThing、ProducedThing、Protocol Binding 与 Servient 分别承担什么责任；
- 多个 Form 的选择、回退，以及 WoT Runtime 与 Directory、平台服务之间的边界。

### 系列二：ClinkZ-WoT 架构设计

这一部分关注 ClinkZ-WoT 最核心的架构问题：一个 TD 如何变成真正可执行的交互。

主线大致如下：

```text
Thing Description
  -> parse and validate
  -> PlanningContext
  -> Logical Plan
  -> Binding Artifact
  -> Compiled Plan Set
  -> runtime interaction
```

会重点讨论：

- 为什么运行时需要预编译执行计划；
- Logical Plan 与 Binding Artifact 如何划分协议中立边界；
- Servient 为什么必须成为运行时编排权威；
- Protocol Binding 为什么不能绕过 Servient 直接调用 Handler；
- ProducedThing 的暴露为什么需要事务性激活；
- Compiled Plan Set 如何进入 draining 和 reclaim；
- 为什么 V1 选择 Cargo 静态链接，而不是直接实现动态插件。

### 系列三：Rust 运行时机制

这一部分把架构结论进一步落实到 Rust。

会讨论：

- generation 如何隔离异步旧结果；
- 异步取消为什么不等于丢弃 Future；
- 为什么外部代码必须在引擎锁外执行；
- Subscription 为什么不应该随意 Clone；
- deadline 为什么不能只是一个整数；
- 为什么队列、缓存、重试和后台任务都必须有上限；
- 没有统一 Executor 时，如何表达公平调度和执行预算；
- 为什么 Drop 不能成为唯一的资源清理协议。

### 系列四：人与 AI 共同开发大型 Rust 项目

ClinkZ-WoT 的开发大量使用了 AI，但这个系列不会把“使用 AI”简单理解为让模型生成代码。

这一部分会记录：

- Owner 与 AI 应该如何划分责任；
- 为什么聊天记录不能充当项目记忆；
- AGENTS、PLAN、PROJECT_STATE、ADR 和 Proposal 各自解决什么问题；
- 如何把复杂重构拆成可验证的 Work Package；
- 如何避免 AI 在长周期项目中重复推翻已经确认的设计；
- 如何审查 AI 给出的结论，而不是被术语和完整文档迷惑；
- 如何在 AI 主导推进的同时，保证开发者自己真正获得项目能力。

## 第一季文章地图

第一季先收录当前已经规划的 12 篇文章。文章按系列分组，编号采用“系列号.系列内序号”；知乎专栏没有二层目录，因此系列名称会直接作为正式文章标题的前缀。后续文章将在写作推进过程中逐步加入新的季度或扩展现有系列。

### 系列一：重新理解 WoT

**1.1**　[**重新理解 WoT | W3C WoT 到底解决什么问题？**](./01-wot-foundations/001-what-does-wot-solve.md)（撰写中）  
真实系统为什么会自然形成多协议、多接口边界，统一 MQTT 或 HTTP 为什么仍然不够，以及 WoT 在哪一层介入。

**1.2**　**重新理解 WoT | Thing Description 是语义契约，而不是设备配置文件**（计划中）  
TD 应该描述什么、不应该描述什么。

**1.3**　**重新理解 WoT | 同一个 Thing 如何通过不同协议交互**（计划中）  
Interaction Affordance、Form 与 Protocol Binding 如何分工。

### 系列二：ClinkZ-WoT 架构设计

**2.1**　**ClinkZ-WoT 架构设计 | 从 TD 到一次属性读取：ClinkZ-WoT 的完整执行链**（计划中）  
一次调用如何穿过规划、Binding 和运行时。

**2.2**　**ClinkZ-WoT 架构设计 | 为什么 WoT Runtime 需要预编译执行计划**（计划中）  
为什么不能在每次调用时重新解释 TD。

**2.3**　**ClinkZ-WoT 架构设计 | Logical Plan 与 Binding Artifact：协议中立如何落地**（计划中）  
哪些决定属于 Core，哪些属于 Binding。

**2.4**　**ClinkZ-WoT 架构设计 | Servient 为什么必须成为运行时权威**（计划中）  
谁拥有路由、生命周期和状态迁移权。

**2.5**　**ClinkZ-WoT 架构设计 | Protocol Binding 为什么不能直接调用 Handler**（计划中）  
如何避免协议组件获得隐藏的执行权。

**2.6**　**ClinkZ-WoT 架构设计 | 为什么 V1 放弃动态插件，选择 Cargo 静态链接**（计划中）  
Rust ABI、异步边界和插件卸载有什么代价。

### 系列三：Rust 运行时机制

**3.1**　**Rust 运行时机制 | Generation 如何阻止异步旧结果破坏新状态**（计划中）  
late completion 如何污染新一代资源。

**3.2**　**Rust 运行时机制 | 为什么所有队列、缓存和后台工作都必须有上限**（计划中）  
如何避免隐藏的无界资源增长。

### 系列四：人与 AI 共同开发大型 Rust 项目

**4.1**　**人与 AI 共同开发大型 Rust 项目 | 我如何用 AI 推进一个复杂 Rust 架构项目**（计划中）  
如何让 AI 参与工程，而不是只生成代码。

## 推荐阅读路线

第一次接触 W3C WoT，可以从第一篇开始顺序阅读：

```text
真实系统的协议与接口碎片化
  -> WoT 的问题空间
  -> Thing Description 与交互语义
  -> Form 与 Protocol Binding
  -> ClinkZ-WoT 架构设计
  -> Rust 运行时机制
  -> 人与 AI 共同开发大型 Rust 项目
```

已经熟悉 WoT、主要关注运行时架构，可以从“ClinkZ-WoT 架构设计 | 从 TD 到一次属性读取：ClinkZ-WoT 的完整执行链”开始，再阅读 Logical Plan、Binding Artifact、Servient 和 Protocol Binding 相关内容。

主要关注 Rust 异步设计，可以直接阅读 generation、取消、锁外执行、Subscription 和资源预算等文章，但建议先了解 ClinkZ-WoT 的基本执行链，否则其中很多类型约束会显得没有必要。

正在尝试使用 AI 推进长期项目，可以先阅读“人与 AI 共同开发大型 Rust 项目”系列。这个部分不会讨论“哪个提示词最厉害”，而会关注项目记忆、证据、决策边界、计划维护和验证机制。

## 如何理解文章中的项目状态

ClinkZ-WoT 仍在开发中。为了避免把设计愿景误写成已经实现的功能，文章会尽量区分以下几种状态：

| 状态 | 含义 |
|---|---|
| **已实现** | 已经可以在源码和测试中找到证据 |
| **已接受设计** | 架构方向已经确认，但实现可能尚未完成 |
| **计划中** | 已进入执行计划，细节仍可能调整 |
| **探索中** | 仍在比较方案，尚未形成正式结论 |
| **历史方案** | 曾经采用或讨论过，但已经被后续设计取代 |

涉及具体实现的文章会记录对应的 ClinkZ-WoT commit。项目继续演进后，如果文章结论发生变化，我会修订文章，并说明变化发生的原因，而不是悄悄把旧结论覆盖掉。

## 这不只是一套教程

我希望这个专栏最终既能成为一套 WoT Runtime 的中文技术资料，也能保留一个复杂系统从模糊想法逐渐收敛为工程实现的过程。

因此，这里不仅会有“正确答案”，还会有：

- 被推翻的接口；
- 暴露责任混乱的架构审查；
- 设计与源码不一致时的修正；
- 看起来通用、实际上无法治理资源的抽象；
- 为了 V1 可交付性主动放弃的能力；
- AI 与开发者意见不一致时，最终如何通过证据收敛。

这些过程有时比最终代码更有价值。因为真正决定一个运行时是否可靠的，往往不是某个漂亮的 trait，而是谁拥有状态、谁推动生命周期、失败后谁负责清理，以及这些规则能否被测试和文档共同证明。

## 项目与文章仓库

- ClinkZ-WoT 主项目：<https://github.com/yushun1990/clinkz-wot>
- 专栏文章源稿：<https://github.com/yushun1990/clinkz-wot-notes>

主项目保存源码、测试、正式架构、ADR、规范和执行计划；文章仓库保存面向读者的解释、图解和开发复盘。

当文章与主项目发生冲突时，以主项目当前源码、测试和正式规范为准。

## 写在开始之前

从零开发一个 Rust WoT 引擎，对我来说并不是“先把代码写完，再把结果整理出来”。

写作本身就是开发过程的一部分：只有能够把一个设计讲清楚，能够指出它的失败路径，能够解释为什么没有选择另一个方案，才能确认自己并非只是记住了结论。

这篇导读会长期更新，也会成为整个专栏的入口。

下一篇将从最基础的问题开始：

> 重新理解 WoT | W3C WoT 到底解决什么问题？