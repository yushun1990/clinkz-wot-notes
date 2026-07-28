---
id: "WOT-001"
title: "重新理解 WoT | W3C WoT 到底解决什么问题？"
subtitle: "它不发明新的通信协议，而是让应用以一致方式理解和操作不同的 Thing"
series: "重新理解 WoT"
series_order: 1
status: "DRAFTING"
author: "yushun1990"
created: "2026-07-28"
updated: "2026-07-28"
summary: "W3C Web of Things 不试图替代 MQTT、HTTP、CoAP 或 Zenoh。它解决的是更上一层的问题：如何用机器可读的方式描述不同物理设备和软件服务提供的能力，让应用围绕 Property、Action 和 Event 交互，再由 Protocol Binding 将这些交互映射到具体协议。"

clinkz_wot:
  repository: "https://github.com/yushun1990/clinkz-wot"
  branch: "master"
  commit: "6c01e07a446f51d413618474554b5eedcf5de23e"
  inspected_at: "2026-07-28"

publication:
  platform: "zhihu"
  published_at: null
  canonical_url: null

related:
  previous: "00-column-guide.md"
  next: "WOT-002"
  articles:
    - "WOT-002"
    - "WOT-003"
  docs:
    - "docs/architecture/10-primary-data-flows.md"
  source: []
  tests: []
---

# 重新理解 WoT | W3C WoT 到底解决什么问题？

> 本文基于 ClinkZ-WoT commit `6c01e07a446f51d413618474554b5eedcf5de23e`。
>
> ClinkZ-WoT 仍处于架构闭合和分阶段实现过程中。本文首先解释 W3C Web of Things 的问题空间，再说明 ClinkZ-WoT 为什么选择从 Thing Description 开始构建运行时；文中的完整执行链并不代表当前已经全部实现。

假设一个应用需要操作三台设备：

- 一台通过 HTTP 暴露接口的空调；
- 一台通过 MQTT 收发消息的传感器；
- 一台通过 CoAP 或 Zenoh 接入的边缘控制器。

它们可能都提供非常相似的能力：读取当前状态、修改一个设定值、启动某项操作、订阅异常通知。

真正困难的地方并不是这些协议无法通信。HTTP、MQTT、CoAP 和 Zenoh 都能很好地完成各自擅长的通信任务。困难在于，应用通常必须分别理解每一种接口：

```text
HTTP
  -> URL、method、status code、header、response body

MQTT
  -> topic、payload、QoS、订阅、请求与响应关联

CoAP
  -> resource、method、option、observe

Zenoh
  -> key expression、query、put、subscriber
```

如果每接入一种设备或协议，应用就要增加一套专用代码，那么系统虽然“连通”了很多设备，却没有形成统一的交互模型。

**W3C Web of Things（WoT）试图解决的，正是这个位于协议之上的问题。**

## WoT 不是另一种通信协议

理解 WoT，首先要把几个层次分开。

| 层次 | 主要问题 | 典型内容 |
|---|---|---|
| 领域与能力 | 这个对象是什么，能做什么 | 温度、开关、启动、告警 |
| 交互模型 | 应用可以怎样操作它 | Property、Action、Event |
| 接口描述 | 数据、安全和访问入口是什么 | Data Schema、Security、Form |
| 协议通信 | 请求和消息怎样传输 | HTTP、MQTT、CoAP、Zenoh |
| 运行时执行 | 如何选择、校验、调用和清理 | planning、Binding、lifecycle |

MQTT、HTTP、CoAP 和 Zenoh 主要位于协议通信层。它们规定消息、请求或数据如何抵达另一端。

WoT 位于更上一层。它不要求所有设备改用某一种统一协议，而是尝试建立一种共同语言，用来描述：

- 这是一个什么 Thing；
- 它暴露了哪些状态；
- 可以对它触发哪些操作；
- 它会产生哪些通知；
- 输入和输出数据满足什么约束；
- 需要什么安全配置；
- 可以通过哪些网络接口完成交互。

因此，WoT 与 MQTT 不是竞争者，WoT 与 HTTP 也不是竞争者。

更准确的关系是：

```text
Thing 的能力与交互语义
          |
          v
Thing Description
          |
          v
Form + Protocol Binding
          |
          v
HTTP / MQTT / CoAP / Zenoh / ...
```

下层协议仍然负责通信，只是不再要求上层应用把某个协议的地址、消息格式和调用习惯当成 Thing 本身。

## WoT 面对的是“接口异构”，不只是“协议异构”

把问题简单说成“协议太多”仍然不够准确。

即使两个设备都使用 MQTT，它们也可能拥有完全不同的 Topic 约定、Payload 格式、命令响应和错误表达。反过来，同一个 Thing 的同一项能力，也可能同时通过 HTTP 和 MQTT 暴露。

例如，一台水泵提供“出口压力”与“启动”两项能力：

```text
出口压力：可以读取，也可以观察变化
启动：接收启动模式，触发一个可能产生副作用的过程
```

不同厂商可能把它们实现成：

```text
厂商 A：
GET /api/pumps/17/pressure
POST /api/pumps/17/start

厂商 B：
订阅 factory/pump/17/telemetry
发布 factory/pump/17/command/start

厂商 C：
对 factory/pumps/17/pressure 发起 query
向 factory/pumps/17/start 执行 put
```

从通信角度看，这是三套不同接口。

从应用角度看，它们表达的却可能是同一组能力：

```text
Property: pressure
Action: start
```

如果没有位于协议之上的共同描述，应用只能为厂商 A、B、C 分别编写适配代码。随着设备、协议和平台增加，这些适配会分散到业务服务、规则引擎、前端组件和自动化流程中。

WoT 并不能让差异自动消失，但它要求这些差异被放到清楚的边界里：

- 共同的能力由 Interaction Affordance 描述；
- 数据约束由 Data Schema 描述；
- 具体访问入口由 Form 描述；
- 协议消息如何构造和解释，由 Protocol Binding 负责。

## Thing：物理设备，也可以是虚拟对象

WoT 中的 Thing 不是“某种联网硬件”的同义词。

W3C WoT Architecture 把 Thing 看作物理实体或虚拟实体的抽象。它可以是：

- 真实传感器；
- 工业控制器；
- 网关代理的传统设备；
- 数字孪生；
- 聚合多个设备能力的虚拟 Thing；
- 通过网络接口提供交互的软件服务。

这很重要，因为应用真正希望复用的通常不是“如何连接某块硬件”，而是“如何操作一个具有明确能力的对象”。

例如，一个模拟水泵和一台真实水泵可以拥有相同的 `pressure` Property 与 `start` Action。二者的底层实现完全不同，但上层测试工具、监控界面和编排逻辑可以围绕同一组交互语义工作。

## WoT 的三个基本交互类型

W3C WoT 用三类 Interaction Affordance 描述 Consumer 可以如何与 Thing 交互。

### Property：Thing 的状态

Property 表达可以读取、写入或观察的状态。

例如：

- 当前温度；
- 电机转速；
- 门锁状态；
- 目标压力；
- 设备运行模式。

Property 不等于某条遥测消息。它强调的是状态及其读、写、观察能力。具体实现可以是一次 HTTP GET、一次 CoAP Read，也可以由某个 MQTT 或 Zenoh Binding 通过消息交互完成。

### Action：触发一个操作

Action 表达调用者可以触发的功能或过程。

例如：

- 启动水泵；
- 重启控制器；
- 执行校准；
- 开始固件升级；
- 生成一次诊断报告。

Action 可以定义输入与输出，也可以声明安全性、幂等性或同步性等信息。它表达的是“调用某项能力”，而不是绑定到某个固定的 HTTP method 或消息 Topic。

### Event：Thing 主动产生的通知

Event 表达 Thing 产生的异步通知源。

例如：

- 过热告警；
- 门被打开；
- 电机堵转；
- 作业完成；
- 连接状态异常。

Event 的具体传输可能使用 MQTT 订阅、Server-Sent Events、WebSocket、CoAP Observe 或 Zenoh Subscriber。WoT 关心的是这些通知在 Thing 的接口中代表什么。

## Thing Description 是 WoT 的中心入口

Thing Description（TD）是 W3C WoT 的核心构件。它使用机器可读的数据描述 Thing 的元数据和面向网络的接口。

一个 TD 通常会包含：

```text
Thing metadata
  |
  +-- Properties
  +-- Actions
  +-- Events
  +-- Data Schemas
  +-- Security Definitions
  +-- Forms
  +-- Links and semantic annotations
```

下面是一段只用于说明层次的概念示意，并非完整的生产 TD：

```json
{
  "title": "Pump 17",
  "properties": {
    "pressure": {
      "type": "number",
      "unit": "kPa",
      "readOnly": true,
      "forms": [
        {
          "href": "https://example.com/pumps/17/pressure",
          "op": ["readproperty"]
        }
      ]
    }
  },
  "actions": {
    "start": {
      "input": {
        "type": "object"
      },
      "forms": [
        {
          "href": "https://example.com/pumps/17/start",
          "op": ["invokeaction"]
        }
      ]
    }
  }
}
```

这里最重要的不是 JSON 语法，而是依赖方向：

```text
应用要读取 pressure
        |
        v
找到 pressure Property
        |
        v
选择支持 readproperty 的 Form
        |
        v
由对应 Protocol Binding 执行
```

应用表达的是交互意图，Form 和 Binding 负责把意图落实为具体网络通信。

## Protocol Binding 负责把语义变成协议消息

W3C WoT Architecture 将 Protocol Binding 描述为从 Interaction Affordance 到特定协议具体消息的映射。

假设应用执行：

```text
read Property "pressure"
```

不同 Binding 可能产生完全不同的协议行为：

```text
HTTP Binding
  -> GET https://example.com/pumps/17/pressure

CoAP Binding
  -> GET coap://example.com/pumps/17/pressure

MQTT Binding
  -> 根据约定发布请求、订阅响应并进行 correlation

Zenoh Binding
  -> 对某个 key expression 发起 query
```

WoT 不要求这些协议表现得一模一样，也不会凭空抹平 QoS、请求响应、发布订阅和流式通信之间的差异。

它提供的是一个共同的上层接口，并要求具体 Binding 明确承担协议适配责任。

这意味着：

- MQTT 仍然可以使用 Topic、QoS 和会话机制；
- HTTP 仍然使用 URL、method、header 和 status code；
- Zenoh 仍然使用 key expression、query、put 和 subscriber；
- 应用不必把这些协议概念作为操作 Thing 的唯一入口。

## WoT 不是什么

理解一个技术，知道它不负责什么同样重要。

### WoT 不是统一传输协议

WoT 不要求设备把已有协议替换成“WoT 协议”。它建立在现有协议和生态之上。

### WoT 不是完整物联网平台

TD、Interaction Model 和 Protocol Binding 不会自动提供设备资产管理、时序数据存储、规则引擎、告警中心、多租户、计费、OTA 或可视化组态。

这些能力可以围绕 WoT 模型构建，但不属于 WoT Runtime 的全部责任。

### WoT 不是自动兼容所有设备的魔法层

一个设备没有可靠的接口文档、数据含义不明确，或者不同协议下的行为根本不等价，仅仅增加 TD 并不能消除这些问题。

TD 必须准确描述真实能力，Binding 也必须正确实现协议语义。错误的模型只会把错误变成机器可读格式。

### WoT 不保证应用完全忽略协议差异

某些场景确实需要感知协议能力，例如实时性、离线行为、QoS、流控或特定安全机制。协议中立不等于协议能力被抹平，而是这些差异通过明确的能力、策略和 Binding 边界暴露，而不是散落在所有业务代码中。

## 只有 TD 还不够，为什么需要 WoT Runtime

TD 是描述，不会自行完成一次属性读取。

一个可执行的 WoT Runtime 至少需要处理：

- 解析并校验 TD；
- 识别 Property、Action、Event 和操作类型；
- 解析安全要求和数据 Schema；
- 判断哪些 Form 可以使用；
- 选择对应 Protocol Binding；
- 构造并执行协议请求；
- 校验和转换返回结果；
- 管理调用、订阅、取消、资源和清理生命周期。

因此，WoT 的价值不仅是给设备增加一份 JSON 文档。真正困难的工程问题，是如何把这份描述稳定地变成可执行交互。

这也是 ClinkZ-WoT 关注的核心。

## ClinkZ-WoT 为什么从 TD 开始

ClinkZ-WoT 的目标，是实现一个协议中立的 Rust WoT Runtime。

主项目当前已经接受的架构方向大致是：

```text
TD document or ProducedThing draft
        |
        v
parse + preserve extensions + validate
        |
        v
capture immutable planning context
        |
        v
logical plans
        |
        v
binding-owned protocol artifacts
        |
        v
admitted immutable plan set
        |
        v
runtime interaction
```

这条链路试图保持两个边界。

第一，运行时的共同部分处理协议中立事实：

- 操作哪个 Thing；
- 目标是哪个 Property、Action 或 Event；
- 执行 read、write、invoke、observe 还是 subscribe；
- 使用什么 Schema、安全分支和候选 Form；
- 谁拥有计划、生命周期和清理责任。

第二，具体 Binding 拥有协议专属知识：

- HTTP method、URL、header 与状态映射；
- MQTT Topic、QoS、correlation 和会话行为；
- Zenoh key expression、query、subscriber 和编码设置；
- 其他协议自己的通信与流控机制。

主项目还规定：一旦某个决定已经进入不可变计划，执行阶段不应回到完整 TD 中重新发现同一个决定。这样做不仅考虑热路径开销，更重要的是让选择结果、所有权、资源预算和错误边界可以被验证。

必须说明当前状态：截至本文基线，ClinkZ-WoT 仍在进行 v4.9 架构闭合，M1 和 M2 处于进行中。部分 Foundation 和 Core 切片已经完成，但完整的 Property Read 纵向链路、Planning、Protocol Binding SPI 与 Servient 集成尚未全部完成。

因此，这里描述的是已经接受的架构方向，不是完整运行时已经发布的声明。

## 这如何落到 Rust 边界

下面不是 ClinkZ-WoT 当前公开 API，而是概念上的模块责任：

```text
Application
  -> ReadProperty(ThingId, PropertyName)

Core / Planning
  -> validate interaction
  -> select admitted candidate
  -> produce protocol-neutral execution facts

Protocol Binding
  -> own protocol artifact
  -> execute protocol I/O
  -> return transport result

Servient
  -> own orchestration, lifecycle and cleanup
```

Rust 类型边界需要防止职责重新混在一起：

- 协议中立请求不应携带某个具体 MQTT 或 HTTP 客户端对象；
- Binding Artifact 应由对应 Binding 拥有，Core 不应猜测其内部结构；
- Servient 应拥有运行时编排权，而不是让 Binding 在隐藏任务中直接调用应用 Handler；
- 调用和订阅需要明确的 owned value、generation、取消和终结清理路径；
- Host 与 `no_std + alloc` 可以拥有不同推进机制，但应保持相同的交互语义。

语义层没有消除复杂度。它把原本散落在各种接入代码中的复杂度，集中到了可以被检查、测试和治理的边界中。

## 采用 WoT 需要付出什么

WoT 并不是免费抽象。

首先，团队必须认真建模。Property、Action 和 Event 的划分如果不准确，后续运行时再完善也无法弥补错误语义。

其次，每种协议或生态仍然需要可靠的 Protocol Binding。Binding 不只是把字符串地址换一种格式，它还要处理协议消息、序列化、安全、关联、订阅、流控和错误映射。

再次，运行时必须处理选择与生命周期。当一个 Interaction Affordance 存在多个 Form 时，哪个可用、哪个安全、是否允许回退、失败后谁清理，都需要明确规则。

最后，并非所有项目都需要完整 WoT Runtime。一个范围固定、接口稳定、只服务单一应用的设备系统，直接使用已有协议 API 可能更简单。WoT 的价值主要出现在需要跨设备、跨协议、跨厂商或长期演进的系统中。

## 总结

1. W3C WoT 不发明新的通信协议。它在现有协议之上描述 Thing 的能力、数据和可执行交互。
2. WoT 解决的不只是协议数量问题，而是不同设备和服务缺少共同、机器可读的接口模型。
3. Thing Description 描述 Property、Action、Event、Schema、安全和 Form；Protocol Binding 再把这些交互映射到 HTTP、MQTT、CoAP、Zenoh 等具体协议。

理解这一点之后，才能继续讨论下一层问题：Thing Description 究竟应该描述什么，又为什么不能把它写成一份运行时配置文件。

## 延伸阅读

- 上一篇：[专栏导读：从零开发一个 Rust WoT 引擎](../00-column-guide.md)
- 下一篇：重新理解 WoT | Thing Description 是语义契约，而不是设备配置文件（计划中）
- 相关项目规范：ClinkZ-WoT `docs/architecture/10-primary-data-flows.md`
- 相关源码与测试：本文不对尚未完成的完整执行链作实现声明

## 项目资料

- ClinkZ-WoT commit: `6c01e07a446f51d413618474554b5eedcf5de23e`
- `README.md`
- `PROJECT_STATE.md`
- `PLAN.md`
- `docs/architecture/10-primary-data-flows.md`

## 外部资料

- W3C Web of Things (WoT) Architecture 1.1
- W3C Web of Things (WoT) Thing Description 1.1
- W3C Web of Things (WoT) Binding Templates
- W3C Web of Things (WoT) Discovery

<!--
内部事实分类：

- W3C WoT 的 Thing、TD、Interaction Affordance 和 Protocol Binding 定义：外部标准事实。
- ClinkZ-WoT 的协议中立、Servient 编排、不可变 Planning Context、Logical Plan、Binding Artifact 和 Compiled Plan Set：ACCEPTED_DESIGN。
- v4.9 架构闭合、M1/M2 状态和当前阻塞：CURRENT PROJECT STATE。
- 完整 Property Read、Planning、Binding SPI 和 Servient 集成：PLANNED / PARTIALLY IMPLEMENTED。
- 文中示例接口和模块流程：CONCEPTUAL ILLUSTRATION。

发布前检查：

- [x] 已读取主项目 AGENTS.md、PROJECT_STATE.md、PLAN.md
- [x] 已检查主题对应的正式规范
- [x] 已记录完整 commit
- [x] 已区分 implemented / accepted / planned
- [x] 未把 MQTT 或其他协议描述为 WoT 的竞争者
- [x] 未虚构公开 Rust API、性能或规模结论
- [ ] 完成作者理解校验
- [ ] 完成第二轮技术事实校验
- [ ] 回填知乎发布信息
-->
