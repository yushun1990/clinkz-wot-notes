---
id: "WOT-001"
title: "重新理解 WoT | 为什么物联网平台不应该从 MQTT Topic 开始设计"
subtitle: "Topic 适合路由消息，但不应该成为平台的业务模型"
series: "重新理解 WoT"
series_order: 1
status: "DRAFTING"
author: "yushun1990"
created: "2026-07-28"
updated: "2026-07-28"
summary: "MQTT Topic 可以承载业务约定，但协议只把它定义为消息路由与订阅匹配标识。物联网平台一旦直接围绕 Topic 组织业务，协议结构就会逐步渗透到规则、权限、存储和应用 API。更稳定的做法，是先定义 Thing 的 Property、Action 和 Event，再把 MQTT Topic 作为具体 Protocol Binding 的通信地址。"

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
  next: null
  articles:
    - "WOT-002"
    - "WOT-003"
  docs:
    - "docs/architecture/10-primary-data-flows.md"
  source: []
  tests: []
---

# 重新理解 WoT | 为什么物联网平台不应该从 MQTT Topic 开始设计

> 本文基于 ClinkZ-WoT commit `6c01e07a446f51d413618474554b5eedcf5de23e`。
>
> ClinkZ-WoT 仍处于架构闭合和分阶段实现过程中。本文介绍的是已经明确的语义优先方向，以及它希望解决的普遍工程问题；文中涉及的完整执行链并不代表当前已经全部实现。

很多物联网平台的第一张设计图，不是设备模型，也不是业务对象，而是一棵 MQTT Topic 树。

```text
/{tenant}/{product}/{device}/telemetry
/{tenant}/{product}/{device}/property/set
/{tenant}/{product}/{device}/property/reply
/{tenant}/{product}/{device}/event/alarm
/{tenant}/{product}/{device}/command/reboot
```

这很自然。

设备已经通过 MQTT 接入，消息必须发布到某个 Topic；服务端需要订阅，权限系统需要配置 ACL，规则引擎也需要知道从哪里接收数据。既然所有数据最终都经过 Topic，那么直接围绕 Topic 设计平台，似乎是最短的路径。

问题是：**Topic 能告诉消息系统把数据送到哪里，却不能完整说明这次交互在业务上意味着什么。**

当平台把 Topic 结构直接当成业务模型时，协议地址会逐步变成规则语言、权限模型、存储索引、应用 API，最后甚至变成整个系统唯一的“设备语义”。最初省掉的建模工作，会在后续每一次协议扩展、设备升级和业务变化中重新出现。

## Topic 首先是通信地址

MQTT 规范把 Topic Name 定义为附着在应用消息上的标签，用于与服务端保存的订阅进行匹配；Topic Filter 则表达客户端对一个或多个 Topic 的订阅兴趣。

这套机制非常适合消息路由：

- Topic 可以分层组织；
- Topic Filter 支持通配符；
- 发布者和订阅者不需要直接认识彼此；
- Broker 可以围绕 Topic 做转发和访问控制；
- 大量设备可以共享统一的接入模式。

但 MQTT 并没有规定下面这些问题：

- `temperature` 是一个可以读取的 Property，还是周期上报的普通消息？
- `reboot` 是一次 Action，还是向某个 Topic 写入字符串？
- `alarm` 是 Event，还是状态 Property 的一次变化？
- 某个值的数据类型、单位、范围和只读约束是什么？
- 一次写入是否需要响应、如何关联响应、失败意味着什么？
- 同一能力通过 HTTP、MQTT 或其他协议暴露时，它们是否代表同一个交互？

开发团队当然可以为 Topic 赋予这些含义。例如约定：

```text
.../property/post      表示属性上报
.../property/set       表示属性写入
.../event/post         表示事件通知
.../service/invoke     表示服务调用
```

这些约定并没有错。真正的问题是：**它们是平台自己定义的业务协议，不是 MQTT Topic 本身提供的语义。**

一旦团队忘记了这层区别，就很容易把“当前 MQTT 接入约定”误认为“设备本身的能力模型”。

## 为什么从 Topic 开始很有吸引力

Topic-first 设计在项目早期通常非常高效。

假设一台泵站设备上报出口压力：

```text
factory-a/pump/17/telemetry/pressure
```

平台只需要：

1. 订阅 `factory-a/pump/+/telemetry/#`；
2. 从 Topic 中解析设备编号和指标名；
3. 解析 Payload；
4. 写入时序数据库；
5. 触发告警规则。

对于只有一种协议、少量设备类型和固定消息格式的系统，这种方案完全可以工作。Topic 同时承担路由键和一部分上下文，开发者不必先建设完整的模型层。

因此，“不要从 Topic 开始设计”并不是说 Topic 命名不应该清晰，也不是说所有平台都必须立刻引入 WoT。它针对的是另一种情况：**当系统准备成为一个长期演进的平台时，不要让 Topic 成为所有上层模块理解设备的唯一入口。**

## Topic 结构如何渗透整个平台

最初，只有接入服务知道 Topic。

随后，业务需求增加：

```text
MQTT Topic
   -> 设备识别
   -> 消息类型判断
   -> Payload 解析
   -> 规则匹配
   -> 权限校验
   -> 存储分区
   -> API 返回结构
   -> 前端组件绑定
```

几个月后，系统中可能到处都是这样的判断：

```text
如果 Topic 以 /telemetry/pressure 结尾，按压力数据处理；
如果 Topic 包含 /event/alarm，进入告警规则链；
如果 Topic 包含 /property/set，检查写权限；
如果 Topic 的第 4 段是 deviceId，就用它查询设备档案。
```

此时 Topic 已经不只是消息地址，而是隐藏的领域模型。

这个模型通常具有三个问题。

### 1. 业务语义依赖字符串位置

`factory-a/pump/17/telemetry/pressure` 中每一段的含义，只有平台约定知道。

把 `pump` 改成 `pumps`，增加一个区域层级，或者调整设备标识位置，都可能影响规则、ACL、数据库导入任务和外部应用。字符串结构的变化被放大成系统级迁移。

### 2. 同一个能力被协议重复定义

假设“读取泵站出口压力”后来同时支持：

```text
MQTT: factory-a/pump/17/telemetry/pressure
HTTP: GET /api/pumps/17/properties/pressure
Zenoh: factory-a/pumps/17/properties/pressure
```

如果平台从地址出发，那么三个地址很容易变成三个彼此独立的接口。应用需要分别理解 MQTT、HTTP 和 Zenoh 的命名方式、错误处理和数据格式。

但从业务上看，它们可能只是同一个操作：

```text
读取 Thing "pump-17" 的 Property "pressure"
```

协议地址不同，不等于业务能力不同。

### 3. Topic 无法独立表达完整交互契约

例如：

```text
factory-a/pump/17/command/start
```

仅看 Topic，仍然不知道：

- Payload 是空、布尔值，还是包含启动模式的对象；
- 操作是否幂等；
- 是否返回执行结果；
- 超时后设备是否可能已经启动；
- 哪些身份有权调用；
- 能否在设备离线时排队；
- `start` 应被理解为 Action，还是某个可写状态 Property。

这些信息最终总要存在。Topic-first 方案只是把它们分散到了文档、代码、规则配置和团队经验中。

## 一个典型的失败场景

继续使用泵站设备的例子。

第一版系统只有 MQTT：

```text
factory-a/pump/17/property/pressure/post
factory-a/pump/17/service/start/invoke
factory-a/pump/17/event/fault/post
```

后来出现三个新需求：

1. 运维系统希望通过 HTTP 调用同一设备能力；
2. 边缘节点希望通过 Zenoh 低延迟访问；
3. 平台需要把真实设备和模拟设备统一展示。

如果业务层直接依赖 MQTT Topic，常见演化路径是：

```text
MQTT Topic 规则
   -> 增加一套 HTTP URL 规则
   -> 增加一套 Zenoh key expression 规则
   -> 模拟设备再增加内部事件名称
   -> 每套规则分别维护权限、Schema、错误和重试
```

系统看起来支持了更多协议，实际上只是复制了更多接口。

更危险的是，四套接口可能逐渐不一致：

- MQTT 的 `pressure` 使用 kPa，HTTP 返回 Pa；
- MQTT 的 `start` 无响应，HTTP 返回任务 ID；
- 模拟设备把 `fault` 表示成字符串，真实设备使用整数错误码；
- 某个协议新增了参数，但其他协议没有同步。

最终，应用不能再操作“泵站”，只能操作“通过某种协议暴露的泵站接口”。

这正是协议细节反向控制业务模型的结果。

## 先定义 Thing，再决定如何通信

W3C Web of Things（WoT）提供的核心思路，不是发明一种新的传输协议，而是在现有协议之上建立统一的交互模型。

一个 Thing 可以提供三类主要 Interaction Affordance：

- **Property**：可读取、可写入或可观察的状态；
- **Action**：由调用者触发、可能产生过程或副作用的操作；
- **Event**：Thing 主动产生、由消费者订阅的通知。

Thing Description（TD）描述这些能力，同时记录数据 Schema、安全要求和用于实际通信的 Form。

下面是一个刻意简化的概念示意。为了突出语义结构，它省略了完整上下文、安全定义和具体 MQTT Binding 扩展字段，不能直接作为生产 TD 使用。

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
          "href": "mqtt://broker.example/factory-a/pump/17/telemetry/pressure",
          "op": "observeproperty",
          "contentType": "application/json"
        },
        {
          "href": "https://api.example/pumps/17/properties/pressure",
          "op": "readproperty",
          "contentType": "application/json"
        }
      ]
    }
  },
  "actions": {
    "start": {
      "input": {
        "type": "object",
        "properties": {
          "mode": { "type": "string" }
        }
      },
      "forms": [
        {
          "href": "mqtt://broker.example/factory-a/pump/17/command/start",
          "op": "invokeaction",
          "contentType": "application/json"
        }
      ]
    }
  },
  "events": {
    "fault": {
      "data": {
        "type": "object",
        "properties": {
          "code": { "type": "integer" },
          "message": { "type": "string" }
        }
      },
      "forms": [
        {
          "href": "mqtt://broker.example/factory-a/pump/17/event/fault",
          "op": "subscribeevent",
          "contentType": "application/json"
        }
      ]
    }
  }
}
```

在这个模型中：

```text
pressure 是 Property
start 是 Action
fault 是 Event
```

MQTT Topic 仍然存在，而且仍然重要；但它被放进了 Form，成为激活某个交互的具体通信信息。

这带来了一个关键的依赖方向：

```text
Application Intent
      |
      v
Property / Action / Event
      |
      v
Protocol Binding
      |
      v
MQTT Topic / HTTP URL / Zenoh key expression
```

应用先表达“我要读取 pressure”，运行时再决定使用哪个 Form、哪个 Protocol Binding 和哪个地址。

地址服务于语义，而不是语义从地址中被临时猜出来。

## 三种设计方式的区别

| 设计方式 | 平台的核心对象 | 优点 | 主要代价 |
|---|---|---|---|
| Topic-first | Topic、Payload、订阅规则 | 上手快，适合单协议和固定场景 | 协议结构容易渗透业务层，跨协议复用困难 |
| Message-envelope-first | 统一消息头、消息类型、设备 ID | 比裸 Topic 更稳定，便于接入治理 | 仍可能把所有交互压成“收发消息”，缺少 Property/Action/Event 区分 |
| Semantic-first / TD-first | Thing、Property、Action、Event、Schema、Form | 应用意图与协议解耦，可统一多协议能力 | 需要规划、Binding、校验和生命周期等额外运行时机制 |

这三种方式不是简单的“落后、过渡、先进”。

如果系统只是一个边界明确的 MQTT 应用，Topic-first 可能已经足够。真正需要 TD-first 的场景，通常具有以下特征：

- 需要支持多种协议或未来可能迁移协议；
- 同一能力需要被设备、边缘、云服务和应用共同理解；
- 需要统一描述读、写、调用和订阅；
- 需要机器可读的 Schema、安全和交互元数据；
- 希望业务应用不直接依赖具体通信地址。

## ClinkZ-WoT 为什么从 TD 开始

ClinkZ-WoT 的目标，是实现一个协议中立的 Rust WoT Runtime。

当前已经明确的架构方向是：

```text
TD document or ProducedThing draft
        |
        v
parse + validate
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
runtime execution
```

这条执行链刻意把两类信息分开：

```text
协议中立的决定：
- 操作哪个 Thing；
- 访问哪个 Property、Action 或 Event；
- 执行 read、write、invoke、observe 还是 subscribe；
- 使用什么 Schema、安全分支和候选 Form。

协议专属的产物：
- MQTT Topic、QoS 和关联方式；
- HTTP method、URL 和 Header；
- Zenoh key expression 与协议侧编码设置。
```

因此，ClinkZ-WoT 不希望应用层提交一个裸 Topic，再由某个全局分发器猜测应该调用什么业务逻辑。应用和 Servient 处理的是协议中立交互；Protocol Binding 负责把交互落实为具体协议通信。

主项目当前规范还要求：一旦某个决定已经进入不可变计划，执行阶段不应重新回到完整 TD 中再次解释。这个约束不仅是为了性能，也为了让选择结果、资源所有权和错误边界可以被验证。

不过，必须明确当前状态：截至本文基线，ClinkZ-WoT 仍在进行 v4.9 架构闭合，M1 和 M2 处于进行中。部分 Foundation 和 Core 切片已经完成，但完整的 Property Read 纵向链路、规划层、Protocol Binding SPI 与 Servient 集成仍未全部完成。本文描述的是项目已经接受的方向，不是对完整运行时能力的发布声明。

## 这如何落实到 Rust 边界

下面不是 ClinkZ-WoT 当前公开 API，而是概念上的责任划分：

```text
应用请求：
ReadProperty(thing = "pump-17", property = "pressure")

逻辑计划：
- interaction = Property("pressure")
- operation = Read
- schema = number, unit kPa
- selected candidate = form #2
- security = selected security branch

HTTP Binding Artifact：
- method = GET
- url = https://api.example/pumps/17/properties/pressure
- response codec = application/json

或者 MQTT Binding Artifact：
- topic = factory-a/pump/17/telemetry/pressure
- subscription/correlation policy = ...
- payload codec = application/json
```

核心层不需要理解 Topic 的层级规则，HTTP Binding 也不需要理解 Zenoh key expression。每个 Binding 拥有协议知识，但不拥有 Thing 的业务定义。

这类边界在 Rust 中通常意味着：

- 协议中立类型不携带某个具体 MQTT 客户端对象；
- Binding 产物由对应 Binding 拥有，不能被 Core 随意解释；
- 运行时通过 owned value、不可变计划和明确的生命周期传递执行权；
- 应用调用面向 Property、Action 和 Event，而不是面向字符串路由规则。

真正困难的部分也由此出现：Form 选择、安全提交、请求关联、取消、资源上限、订阅生命周期和错误映射都必须有明确所有者。语义优先没有消除复杂度，而是把复杂度从分散的业务代码收回到可审查的运行时边界中。

## 不从 Topic 开始，不等于抛弃 MQTT

这篇文章最容易被误解成“MQTT 不适合物联网平台”。结论正好相反。

MQTT 在设备接入、发布订阅、弱网络通信和大规模消息分发中仍然非常有价值。Topic 也完全可以使用清晰、稳定、可治理的命名规则。

需要避免的是下面这种依赖关系：

```text
业务能力 = Topic 字符串结构
```

更合适的关系是：

```text
业务能力
   -> 交互语义
   -> 协议映射
   -> Topic
```

Topic 可以是某个能力的地址，但不应该成为能力本身。

## 已有 MQTT 平台可以怎样逐步调整

引入语义层不一定要求推翻现有系统。一个现实的迁移顺序可以是：

1. 先列出平台真正关心的设备能力，而不是现有 Topic；
2. 把每项能力归类为 Property、Action 或 Event；
3. 为数据补充类型、单位、范围和读写约束；
4. 建立“交互语义到 Topic”的显式映射；
5. 让规则、API 和 UI 优先引用语义标识，而不是解析 Topic；
6. 最后再考虑是否使用完整 TD 和 WoT Runtime。

例如，不再让告警规则直接订阅：

```text
factory-a/pump/+/telemetry/pressure
```

而是表达：

```text
对所有 Pump Thing 的 pressure Property 建立阈值规则
```

底层仍然可以由 MQTT Binding 把这项需求编译成原生 Topic Filter。区别在于，Topic Filter 是执行产物，不再是业务规则的唯一源代码。

## 总结

1. MQTT Topic 是优秀的消息路由和订阅匹配机制，但它的业务含义来自平台约定，而不是 MQTT 协议本身。
2. 当 Topic 结构直接进入规则、权限、存储和应用 API 时，平台就会被具体协议反向塑形；多协议支持往往退化成多套接口复制。
3. WoT 的语义优先思路是先定义 Thing 的 Property、Action 和 Event，再通过 Form 和 Protocol Binding 映射到 MQTT Topic、HTTP URL 或其他协议地址。

真正应该稳定下来的，不是某一棵 Topic 树，而是设备和服务向外提供的能力。

## 延伸阅读

- 上一篇：[专栏导读：从零开发一个 Rust WoT 引擎](../00-column-guide.md)
- 下一篇：重新理解 WoT | W3C WoT 不是一种新协议（计划中）
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
- W3C Web of Things (WoT) Binding Registry
- OASIS MQTT Version 5.0

<!--
内部事实分类：

- MQTT Topic 的协议定义：外部标准事实。
- WoT Property/Action/Event、TD、Form、Protocol Binding：外部标准事实。
- ClinkZ-WoT semantic-first、immutable planning context、logical plan、binding artifact、plan set：ACCEPTED_DESIGN。
- M1/M2 状态和完整纵向链路尚未完成：PLANNED / current project state。
- 文中 Rust 结构：概念示意，不是当前公开 API。

发布前检查：

- [x] 已重新读取主项目 AGENTS.md、PROJECT_STATE.md 和 PLAN.md
- [x] 已检查主题对应的正式规范
- [x] 已记录完整 commit
- [x] 已区分 accepted / planned / conceptual illustration
- [x] 没有虚构 benchmark、容量或完整实现状态
- [ ] 完成作者理解校验
- [ ] 完成第二轮技术事实校验
- [ ] 回填知乎发布信息
-->
