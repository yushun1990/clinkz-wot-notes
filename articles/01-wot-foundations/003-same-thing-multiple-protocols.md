---
id: "WOT-003"
title: "重新理解 WoT | 同一个 Thing 为什么可以同时使用 HTTP、MQTT 和 Zenoh？"
subtitle: "Interaction Affordance 保持业务语义，Form 与 Protocol Binding 负责协议映射"
series: "重新理解 WoT"
series_order: 3
status: "DRAFTING"
author: "yushun1990"
created: "2026-08-04"
updated: "2026-08-04"
summary: "同一个 Thing 的同一项 Property、Action 或 Event 可以包含多个 Form，分别描述不同操作和协议入口。应用表达 readproperty、invokeaction 或 subscribeevent 等交互意图，WoT Runtime 负责解析共同语义，Protocol Binding 再将已经确定的交互映射为 HTTP 请求、MQTT 消息、CoAP 资源访问或 Zenoh query、put 与 subscriber。"

clinkz_wot:
  repository: "https://github.com/yushun1990/clinkz-wot"
  branch: "master"
  commit: "30485b1a51470f328e79453ba0e82e3358c14f79"
  inspected_at: "2026-08-04"

publication:
  platform: "zhihu"
  published_at: null
  canonical_url: null

related:
  previous: "WOT-002"
  next: "WOT-004"
  articles:
    - "WOT-001"
    - "WOT-002"
    - "WOT-004"
  docs:
    - "docs/spec/planning.md"
    - "docs/spec/binding-spi.md"
    - "docs/work-packages/WP-600-protocol-bindings.md"
  source:
    - "protocol-bindings/protocols/zenoh/src/form.rs"
  tests: []
---

# 重新理解 WoT | 同一个 Thing 为什么可以同时使用 HTTP、MQTT 和 Zenoh？

> 本文基于 ClinkZ-WoT commit `30485b1a51470f328e79453ba0e82e3358c14f79`。
>
> ClinkZ-WoT 当前已经具有 TD 数据结构、通用 Form 处理以及一部分 Zenoh Form 解析与操作映射代码。active v5 架构进一步规定由共享 Planning 完成有效 Form 解析和候选构造，再由 Binding Compiler 生成协议专属 Artifact。完整 Zenoh/zenoh-pico 生产 Binding 的迁移属于 WP-600，当前仍为 Planned。

上一篇文章说明了 Thing Description 可以由设备、网关或平台提供。现场 PLC 和仪表即使不认识 WoT，也可以由网关映射为一个可被 Consumer 使用的 Thing。

接下来会出现一个更具体的问题。

假设泵站 Thing 暴露一个出口压力属性：

```text
Property: outletPressure
```

局域网中的控制程序希望通过 Zenoh 读取它；云端管理平台只能访问 HTTPS；另一个监测服务希望通过 MQTT 接收压力变化。

那么，这究竟是三个 Thing，还是同一个 Thing？

WoT 的回答是：它可以仍然是同一个 Thing，甚至仍然是同一个 Property，只是提供了不同的网络交互入口。

这些入口由 **Form** 描述，具体协议行为则由 **Protocol Binding** 执行。

## 业务看到的是出口压力，协议看到的是 URL、Topic 和 Key Expression

对于业务应用来说，目标很简单：

```text
读取出口压力
观察出口压力变化
```

但不同协议看到的接口完全不同：

```text
HTTP
  GET /things/pump-17/properties/outletPressure

MQTT
  subscribe pump/17/outlet-pressure

Zenoh
  query pump/17/outlet-pressure
```

HTTP 关心 URL、method、header 和 status code。

MQTT 关心 Broker、Topic、QoS、retain、subscription 和消息关联方式。

Zenoh 关心 locator、key expression、query、put、subscriber、优先级和拥塞控制。

如果应用直接依赖这些细节，业务代码会再次被协议拆开：

```text
HTTP 应用知道 URL
MQTT 应用知道 Topic
Zenoh 应用知道 Key Expression
```

WoT 并不删除这些概念，而是把它们放到协议边界中。

业务接口继续表达 `outletPressure`；Form 描述网络入口；Binding 负责理解入口并执行协议通信。

## 一个 Affordance 可以拥有多个 Form

Thing Description 中的 Property、Action 和 Event 都属于 Interaction Affordance。

Affordance 描述 Consumer 可以做什么，Form 描述某项操作怎样通过网络完成。

下面是一段概念性的 TD 片段。它用于说明结构，不是一份可直接部署的完整生产 TD：

```json
{
  "title": "Pump Station 17",
  "properties": {
    "outletPressure": {
      "type": "number",
      "unit": "kPa",
      "readOnly": true,
      "observable": true,
      "forms": [
        {
          "href": "https://edge.example.com/things/pump-17/properties/outletPressure",
          "op": "readproperty",
          "contentType": "application/json"
        },
        {
          "href": "zenoh://edge-03:7447/pump/17/outlet-pressure",
          "op": ["readproperty", "observeproperty"],
          "contentType": "application/json"
        },
        {
          "href": "mqtt://broker.example.com/pump/17/outlet-pressure",
          "op": "observeproperty",
          "contentType": "application/json"
        }
      ]
    }
  }
}
```

这份描述表达了三件事：

1. `outletPressure` 是同一个 Property；
2. 读取和观察是不同的 WoT operation；
3. 不同 operation 可以由不同 Form 和不同协议实现。

HTTPS Form 可以用于读取当前值。

Zenoh Form 同时声明可以读取和观察。

MQTT Form 只声明观察能力。

因此，“一个 Affordance 有多个 Form”不等于“所有 Form 都能完成所有操作”。

Runtime 必须先确定调用者需要什么 operation，再寻找声明支持该 operation 的 Form。

## `op` 表达交互意图，不是协议方法

TD 中的 `op` 很容易被误解成 HTTP method。

实际上：

```text
readproperty
writeproperty
observeproperty
invokeaction
subscribeevent
```

这些是 WoT Interaction Model 中的操作类型，表达的是调用者的语义意图。

它们不是：

```text
GET
POST
SUBSCRIBE
QUERY
PUT
```

后面这些才是具体协议中的操作形式。

例如，同一个 `readproperty` 可以被映射为：

```text
HTTP Binding
  -> GET 一个资源

CoAP Binding
  -> GET 一个 CoAP resource

Zenoh Binding
  -> 对 key expression 发起 query

MQTT Binding
  -> 按该 Binding 约定完成请求、响应或状态读取
```

因此，两层之间的关系是：

```text
WoT operation
  readproperty
       |
       v
Protocol Binding
       |
       +--> HTTP GET
       +--> CoAP GET
       +--> Zenoh Query
       +--> MQTT-specific interaction pattern
```

WoT 定义共同的交互意图，但不会假装所有协议都具有相同的消息模型。

尤其是 MQTT 这类发布订阅协议，并不存在一个天然等价于 HTTP GET 的通用操作。具体 Binding 必须规定 Topic、消息方向、响应关联、超时和订阅行为，TD 也可能需要对应的 Binding 扩展词汇。

## Form 不是一个可以执行协议的驱动

Form 本身只是机器可读的网络交互描述。

它可以包含或继承：

- `href`；
- `op`；
- `contentType`；
- `subprotocol`；
- 预期响应；
- 协议或平台扩展字段；
- 与安全配置相关的信息。

但一段 JSON 不会自己建立连接、发送请求或解析报文。

真正执行通信的是 Protocol Binding。

以 Zenoh 为例，一个 Binding 可能需要完成：

```text
解析 zenoh:// URI
  -> 获得 router/peer endpoint
  -> 获得 key expression
  -> 读取 Zenoh 扩展元数据
  -> 将 WoT operation 映射为 Query / Put / Subscribe
  -> 构造协议调用
  -> 接收并解码结果
  -> 转换为 WoT 层结果
```

HTTP Binding 会完成另一套工作：

```text
解析 URL
  -> 确定 method
  -> 构造 header 和 body
  -> 执行请求
  -> 解释 status code
  -> 解码 response body
```

MQTT Binding 则可能需要管理：

```text
Broker connection
Topic
QoS
request/response correlation
subscription lifecycle
retained message semantics
payload encoding
```

Form 提供执行所需的描述，Binding 才是理解并落实该描述的协议组件。

## 同一个 operation，通过不同协议执行

假设应用调用一个概念性的 ConsumedThing API：

```rust
let pressure = thing.read_property("outletPressure").await?;
```

从应用角度看，它只表达了：

```text
Thing: Pump Station 17
Target: outletPressure
Operation: readproperty
```

运行时内部则需要经历更多步骤：

```text
Application
  -> request readproperty("outletPressure")
  -> locate PropertyAffordance
  -> find forms supporting readproperty
  -> resolve target and effective metadata
  -> choose one usable candidate
  -> hand selected plan to matching Binding
  -> Binding performs protocol I/O
  -> decode and validate result
  -> return Property value
```

如果最终使用 HTTPS：

```text
readproperty
  -> HTTP Binding
  -> GET /things/pump-17/properties/outletPressure
  -> 200 OK + payload
```

如果最终使用 Zenoh：

```text
readproperty
  -> Zenoh Binding
  -> query pump/17/outlet-pressure
  -> reply payload
```

应用看到的是同一种 Property Read。

协议层执行的却是完全不同的通信过程。

## Producer 也可以用多个协议暴露同一个能力

多协议不只存在于 Consumer 一侧。

一个 ProducedThing 也可能同时通过 HTTP 和 Zenoh 暴露 `outletPressure`：

```text
HTTP request --------+
                     |
Zenoh query ----------+--> PumpStation ProducedThing
                     |        |
MQTT subscription ----+        v
                         outletPressure Handler
```

这里最危险的直觉实现是：

```text
HTTP Binding 直接调用 Handler
Zenoh Binding 直接调用 Handler
MQTT Binding 再实现一套处理路径
```

这样很容易形成三套略有差异的行为：

- HTTP 路径做了权限检查，Zenoh 路径忘了；
- 一条路径校验 Schema，另一条直接传递原始 Payload；
- 一条路径尊重 generation，另一条仍把旧响应发送出去；
- 取消、超时和错误映射各自实现；
- Handler 的并发和生命周期无法统一治理。

更清楚的边界是：

```text
protocol message
  -> Protocol Binding 解码并形成协议中立请求
  -> Servient 负责 admission、路由和 Handler 编排
  -> Handler 返回协议中立结果
  -> Protocol Binding 编码并发送协议响应
```

这样，多种协议共享同一项 Thing 语义，但仍由各自 Binding 负责协议行为。

Binding 不应该因为接收到了网络消息，就获得绕过 Runtime 直接控制应用 Handler 的权力。

## 协议中立不等于抹平协议差异

WoT 的目标不是让所有协议看起来完全一样。

HTTP、MQTT、CoAP 和 Zenoh 在很多方面具有真实差异：

- 请求响应还是发布订阅；
- 是否具有原生观察能力；
- 连接与会话如何保持；
- QoS 和可靠性怎样表达；
- 数据是否可能 retained 或 cached；
- 是否返回一个值、多个值或持续流；
- 取消和超时如何传播；
- 背压和流控在哪里发生；
- 错误由状态码、消息还是连接状态表达。

这些差异不能靠给函数统一命名就消失。

更准确的责任划分是：

| 共同语义层 | Protocol Binding |
|---|---|
| Thing 与 Affordance 身份 | URL、Topic、Key Expression、Resource |
| `readproperty`、`invokeaction` 等 operation | GET、POST、QUERY、PUT、SUBSCRIBE 等协议操作 |
| Data Schema 与媒体要求 | Payload 编码和协议 framing |
| 安全需求与已选择的安全分支 | 传输认证材料和协议安全机制 |
| plan、generation、deadline、cancellation | correlation、session、QoS、flow control |
| 应用 Handler 和结果语义 | status、ack、reply、publication |

协议中立的真正含义是：

> 共同语义不依赖某一个协议，但协议专属事实仍然由明确的 Binding 拥有。

它不是把所有协议细节塞进一个巨大的通用请求结构，也不是让 Core 猜测 MQTT Topic 或 Zenoh key expression 应该怎样工作。

## 多个 Form 不代表它们完全等价

同一个 operation 拥有多个 Form，只能说明 TD 提供了多个候选入口。

它并不自动保证：

- 每个入口当前都可达；
- 每个入口具有相同延迟；
- 每个入口使用相同安全机制；
- 每个入口具有相同一致性和缓存行为；
- 每个入口都适合当前网络环境；
- 一个入口失败后可以安全切换到另一个；
- 重试不会重复执行非幂等 Action。

例如，局域网 Zenoh Form 可能延迟更低，但远程应用无法访问；HTTPS Form 可以跨公网使用，但需要不同凭据；MQTT Form 适合持续观察，却不一定适合读取一次即时状态。

因此，Form 描述“有哪些交互入口”，但不会单独完成最终选择。

选择还需要考虑：

```text
operation compatibility
security compatibility
binding availability
application policy
network environment
candidate order
failure and fallback semantics
```

这些问题属于下一篇文章，而不是 Protocol Binding 可以私下决定的事情。

## ClinkZ-WoT 当前如何划分这些责任

ClinkZ-WoT 的 active v5 架构把 Form 处理拆成共享 Planning 和具体 Binding 两部分。

共享 Planning 负责：

- 应用 W3C 的纯默认规则；
- 解析 `base` 与 `href`；
- 确定 operation；
- 计算有效安全要求；
- 处理媒体、响应和 URI 模板；
- 建立 Form 身份；
- 构造有序且有界的 Binding candidate；
- 生成协议中立的 Logical Plan。

具体 Binding Compiler 接收的不是整棵 TD，也不是一组可以自由重选的 Form，而是一个已经解析并确定的 candidate。

它负责生成不可变的协议专属 Binding Artifact，例如：

```text
HTTP Artifact
  - method
  - resolved URL template
  - protocol metadata

Zenoh Artifact
  - transport
  - authority
  - key expression
  - operation kind
  - QoS metadata
```

执行阶段再由匹配的 Binding 使用 Artifact 完成实际 I/O。

目标依赖方向是：

```text
TD Form
  -> shared effective-form planning
  -> Logical Plan
  -> Binding Candidate
  -> Binding Compiler
  -> immutable Binding Artifact
  -> Binding execution
```

Binding 不能在执行时重新扫描 TD、改选另一个 Form 或改变 operation。

这种限制不是为了削弱 Binding，而是为了让选择结果、安全边界、资源预算和生命周期具有一个可以检查的权威来源。

## 当前实现到了哪里

ClinkZ-WoT 当前的 Zenoh Form 代码已经实现了一部分具体协议知识，包括：

- 识别 `zenoh://`、`zenoh+tcp://` 和 `zenoh+udp://`；
- 解析 authority 与 key expression；
- 读取 content type 和 Zenoh 扩展元数据；
- 将 Property Read 等读取操作映射为 Query；
- 将写入操作映射为 Put；
- 将观察和事件订阅映射为 Subscribe；
- 将 Action Invoke 等操作映射为 Request/Reply。

这些代码证明了 Form 如何被转成 Zenoh 专属执行信息。

但它们不能被理解为 active v5 的完整多协议运行时已经完成。

当前主线已经建立了部分协议中立 Planning、Binding Compiler/Artifact 和 Property Read 投影基础；完整 Servient 编排与 Zenoh/zenoh-pico 生产执行迁移仍需要后续工作。WP-600 当前仍是 Planned。

因此，本文涉及两类不同状态：

```text
Zenoh Form 解析与旧路径操作映射
  -> 已有实现

active v5 完整 shared planning + compiled artifact + production binding execution
  -> 已接受架构，仍在分阶段实现
```

## 总结

第一，同一个 Thing 的同一项 Property、Action 或 Event 可以包含多个 Form。Affordance 保持业务能力身份，`op` 表达 WoT 交互意图，Form 描述具体网络入口。

第二，Protocol Binding 把已经确定的交互映射为 HTTP、MQTT、CoAP 或 Zenoh 的真实消息、会话、订阅和响应行为。协议中立不是删除协议差异，而是把差异收敛到明确的 Binding 边界。

第三，多个 Form 只是提供多个候选入口，并不自动决定当前应该使用哪个，也不保证失败后可以任意切换。Form 选择、安全、策略和 fallback 必须由 Runtime 在执行前明确决定。

下一篇将继续回答这个问题：

> 一个 Property 同时提供 HTTPS、MQTT 和 Zenoh Form 时，Runtime 到底应该选择哪一个？

## 延伸阅读

- 上一篇：[重新理解 WoT | 现场设备不支持 WoT，Thing Description 从哪里来？](./002-td-gateway-directory-in-practice.md)
- 下一篇：重新理解 WoT | 一个 Thing 同时提供多个 Form，客户端应该选哪个
- 相关规范：ClinkZ-WoT `docs/spec/planning.md`
- 相关规范：ClinkZ-WoT `docs/spec/binding-spi.md`
- 相关工作包：ClinkZ-WoT `docs/work-packages/WP-600-protocol-bindings.md`

## 项目资料

- ClinkZ-WoT commit：`30485b1a51470f328e79453ba0e82e3358c14f79`
- `protocol-bindings/protocols/zenoh/src/form.rs`
- `docs/spec/planning.md`
- `docs/spec/binding-spi.md`
- `docs/work-packages/WP-600-protocol-bindings.md`

## 外部资料

- [W3C Web of Things (WoT) Architecture 1.1](https://www.w3.org/TR/wot-architecture11/)
- [W3C Web of Things (WoT) Thing Description 1.1](https://www.w3.org/TR/wot-thing-description11/)
- [W3C Web of Things (WoT) Binding Templates](https://www.w3.org/TR/2023/NOTE-wot-binding-templates-20230928/)

<!--
内部事实分类：

- Interaction Affordance、Form、op、Protocol Binding 与多 Form 结构：外部标准事实。
- 泵站多协议交互示例：CONCEPTUAL COMPOSITE，用于说明常见系统拓扑，不声称对应某个具体项目。
- Zenoh Form 识别、target 提取、metadata 解析与 operation-kind 映射：IMPLEMENTED。
- shared effective-form planning、Logical Plan、Binding Candidate、Binding Artifact 与 Binding 不得重选 Form：ACCEPTED_DESIGN，并有部分 Property Read 基础实现。
- WP-600 Zenoh/zenoh-pico 完整生产 Binding 迁移：PLANNED。

发布前检查：

- [x] 已与 WOT-002 区分内容边界
- [x] 未提前展开 WOT-004 的具体选择算法
- [x] 已区分 WoT operation 与协议 method/message type
- [x] 已说明多个 Form 不保证完全等价
- [x] 已区分 Binding 与 Servient/Handler 权责
- [x] 已区分现有 Zenoh 代码与 active v5 目标架构
- [ ] 完成作者理解校验
- [ ] 完成第二轮技术事实校验
- [ ] 制作 Affordance/Form/Binding 分层图
- [ ] 回填知乎发布信息
-->
