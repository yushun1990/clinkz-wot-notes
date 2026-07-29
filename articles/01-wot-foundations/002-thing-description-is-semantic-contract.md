---
id: "WOT-002"
title: "重新理解 WoT | Thing Description 是语义契约，而不是设备配置文件"
subtitle: "TD 描述 Consumer 可以理解和使用的 Thing 接口，而不是某个运行时实例的连接池、凭据、缓存和执行状态"
series: "重新理解 WoT"
series_order: 2
status: "DRAFTING"
author: "yushun1990"
created: "2026-07-29"
updated: "2026-07-29"
summary: "Thing Description 可以包含 Property、Action、Event、Data Schema、Form 和安全要求，但这不意味着它应该容纳运行时的全部配置。本文从消费者可见契约与运行时私有状态的边界出发，解释 TD 应该描述什么、扩展成员能做什么、Thing Model 与 TD 有何区别，以及 ClinkZ-WoT 为什么在解析 TD 之后才捕获策略、Binding 注册和 generation 等执行上下文。"

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
  previous: "WOT-001"
  next: "WOT-003"
  articles:
    - "WOT-001"
    - "WOT-003"
  docs:
    - "docs/architecture/10-primary-data-flows.md"
  source:
    - "td/src/lib.rs"
    - "td/src/thing.rs"
    - "td/src/thing_model.rs"
    - "td/src/components/form.rs"
    - "td/src/components/data_schema.rs"
  tests: []
---

# 重新理解 WoT | Thing Description 是语义契约，而不是设备配置文件

> 本文基于 ClinkZ-WoT commit `f453f165c2ea775e5f0d10c36f1e419fcc1d79f3`。
>
> ClinkZ-WoT 当前的 `td` crate 已经实现 TD/TM 数据结构、序列化、反序列化、校验和扩展成员保留。本文后半部分提到的 Planning Context、Logical Plan 与 Binding Artifact 属于已经接受但尚未完整进入产品源码的 v5 目标架构。

上一篇文章讨论了 WoT 面对的问题：真实系统即使已经打通通信，上层应用仍然缺少共同、机器可读的 Thing 接口。

于是一个新的问题出现了：

> 这份接口描述究竟应该包含多少内容？

假设团队正在为一个泵站编写 Thing Description。最初的内容很清楚：出口压力、目标压力、启动操作和过载告警。

随着项目推进，越来越多参数看起来都可以放进同一份 JSON：

```text
MQTT broker 地址
连接池大小
请求超时时间
失败重试次数
订阅缓冲区容量
缓存过期时间
TLS 证书文件路径
访问令牌
Rust Handler 名称
当前 generation
已经建立的会话
正在执行的调用
```

这样做很有诱惑力。一份文档似乎就能同时描述设备、网络和运行时，部署系统只要读取它便可以完成全部工作。

但这份文件很快会失去清楚的身份。

Consumer 为什么需要知道连接池大小？同一个 Thing 从一个 Servient 实例迁移到另一个实例后，是否要因为线程数变化而发布新的 TD？访问令牌能否安全地交给所有读取 TD 的组件？一个订阅正在运行，是否意味着订阅句柄也要写回 TD？

这些问题说明：

**Thing Description 可以包含通信元数据，但它不是某个运行时实例的完整配置快照。**

本文把 TD 称为“语义契约”，强调的正是这条边界。

## “语义契约”究竟是什么意思

W3C Thing Description 1.1 将 TD 定义为描述 Thing 元数据和接口的形式化信息模型与通用表示。一个 TD 的主要组成包括：

- Thing 自身的标识与可读元数据；
- Consumer 可以使用的 Interaction Affordance；
- 交互数据的 Data Schema；
- 访问这些交互所需的 Form；
- 安全机制的描述；
- 与其他资源之间的 Link；
- 必要的语义、协议或安全扩展。

“契约”意味着 TD 面向 Thing 与 Consumer 之间可观察、可依赖的边界。

Consumer 读取 TD 后，应该能够回答：

```text
这是什么 Thing？
它提供哪些状态、操作和事件？
输入与输出是什么结构？
哪些操作可读、可写、可观察或可订阅？
通过什么入口完成交互？
需要满足什么安全要求？
```

它不需要回答：

```text
Servient 使用几个工作线程？
Binding 内部维护多少连接？
缓存采用 LRU 还是其他算法？
当前有多少请求正在执行？
哪个 Rust 对象拥有 socket？
一次调用属于哪个 generation？
```

前一组是网络接口事实，后一组是实现和运行时状态。

还需要澄清一点：这里的“语义”并不意味着每个 TD 都自动拥有完整的领域本体。

一个名为 `outletPressure` 的 Property，对当前系统的开发者很容易理解；但仅凭这个局部名称，外部系统并不一定知道它与另一个厂商的 `dischargePressure` 是否代表同一概念。JSON-LD 的 `@context`、`@type` 和 TD Context Extension 可以进一步增加机器可理解的领域含义，但这需要明确的词汇表和共同约定。

因此，TD 至少提供机器可读的交互语义；是否进一步达到跨领域的知识语义，取决于项目采用的扩展和 ontology。

## Property、Action 和 Event 描述的是能力

Thing Description 使用三类 Interaction Affordance 表达 Consumer 可以怎样使用一个 Thing。

### Property：可读取、写入或观察的状态

例如泵站的：

- 出口压力；
- 目标压力；
- 运行状态；
- 当前工作模式。

Property 不是“数据库字段”的同义词，也不等于某一条 MQTT 遥测消息。

`targetPressure` 可以成为 Property，是因为它是 Thing 对外暴露、可以被 Consumer 理解和操作的状态。至于这个值在内部保存在 PLC 寄存器、内存、数据库还是远程控制器中，不属于 Property 的契约。

### Action：可以触发的过程

例如：

- 启动水泵；
- 执行校准；
- 切换控制模式；
- 生成诊断报告。

Action 描述可以调用什么、输入和输出满足什么约束，以及调用具有哪些行为特征。它不应该直接绑定某个 Rust 函数指针、闭包地址或 Handler 实例。

### Event：异步产生的通知

例如：

- 过载告警；
- 作业完成；
- 传感器故障；
- 连接状态变化。

Event 描述通知的数据、订阅和取消所需的信息。至于运行时使用 channel、ring buffer、协议原生订阅还是轮询适配器推进事件，不属于 TD 的公共接口。

这三类 Affordance 共同描述的是 Thing 的能力，而不是能力背后的实现对象。

## Data Schema 描述数据约束，不描述内存布局

只有 Property、Action 和 Event 的名字仍然不够。

Consumer 需要知道：

- 数据是数字、字符串、数组还是对象；
- 单位是什么；
- 是否只读或只写；
- 允许哪些枚举值；
- 数值范围是什么；
- 对象包含哪些字段；
- Action 的输入和输出是什么；
- Event 推送的数据是什么。

这些内容由 Data Schema 描述。

例如：

```json
{
  "type": "number",
  "unit": "kPa",
  "minimum": 0,
  "maximum": 1600,
  "readOnly": true
}
```

这段 Schema 约束 Consumer 可以观察到的数据。

它不应该描述 Rust 中采用 `f32` 还是 `f64`，也不应该携带结构体内存布局、序列化缓冲区地址、缓存对象或数据库列名。协议编码和语言内部表示可以变化，只要对外数据仍然满足契约。

这条边界允许同一份 TD 被不同语言、不同运行时和不同部署方式消费。

## Form 为什么属于 TD，却不等于运行时配置

这里最容易产生疑问。

既然 Form 中可以出现 `href`、`contentType`、`op`、`subprotocol` 和安全引用，为什么还说 TD 不是配置文件？

因为一份可执行的网络接口契约必须告诉 Consumer **如何访问该接口**。

W3C 将 Form 定义为 Hypermedia Control。它表达的内容可以概括为：

> 要在当前上下文中执行某个 operation，应向哪个目标提交怎样的请求，并遵循哪些通信元数据。

例如：

```json
{
  "href": "https://edge.example.com/pumps/17/properties/outletPressure",
  "op": "readproperty",
  "contentType": "application/json"
}
```

这些信息是 Consumer 完成交互所需的公开事实。

但是 Form 不包含：

- 已建立的 TCP、QUIC、MQTT 或 Zenoh session；
- 客户端连接池对象；
- DNS 缓存；
- socket 文件描述符；
- correlation map 当前内容；
- retry queue；
- 协议任务的 Future 或 Waker；
- Binding 内部的可变状态。

可以把两者的差异概括为：

```text
Form
  = 如何找到并使用一个网络交互入口

Binding / Runtime state
  = 当前进程如何实现、复用和推进这次交互
```

前者可以跨实现交换，后者由具体运行时拥有生命周期。

## Security 描述要求，不保存真实凭据

安全字段也容易被误解成“连接配置”。

TD 中的 `securityDefinitions` 和 `security` 用于描述交互要求采用什么安全机制，以及凭据信息应出现在哪里。

例如，它可以说明：

- 使用 Basic、Bearer、API Key、PSK 或 OAuth2；
- token 服务或授权服务在哪里；
- API Key 位于 header、query 或 cookie；
- 某个 Form 需要哪些 OAuth scope；
- 多种安全机制是任选其一还是必须组合。

但 TD 不应该保存实际密码、私钥、长期访问令牌或设备密钥。

安全契约与凭据材料应该分开：

```text
TD
  -> 声明需要 bearer token

Credential Provider / Secret Store
  -> 为当前调用者提供可以使用的 token
```

同一份 TD 可以被多个 Consumer 使用，但每个 Consumer 的身份、权限和凭据都可能不同。把真实凭据写进 TD，不仅破坏复用，还会扩大秘密泄露范围。

ClinkZ-WoT 的 v5 目标架构也把 TD 中的安全候选与运行时提交的 credential/provider identity 分开处理：TD 提供安全要求，具体凭据和适用性在规划与调用上下文中决定。

## 一个更完整的 TD 示例

下面用一个概念化的泵站 TD 展示公共接口中应该出现的内容。示例用于说明边界，并非 ClinkZ-WoT 当前稳定公开 API 的输出。

```json
{
  "@context": "https://www.w3.org/2022/wot/td/v1.1",
  "id": "urn:example:pump-station:17",
  "title": "Pump Station 17",
  "base": "https://edge.example.com/pump-stations/17/",
  "securityDefinitions": {
    "operator": {
      "scheme": "bearer",
      "format": "jwt",
      "in": "header"
    }
  },
  "security": "operator",
  "properties": {
    "outletPressure": {
      "title": "Outlet pressure",
      "type": "number",
      "unit": "kPa",
      "minimum": 0,
      "maximum": 1600,
      "readOnly": true,
      "forms": [
        {
          "href": "properties/outletPressure",
          "op": "readproperty"
        }
      ]
    },
    "targetPressure": {
      "type": "number",
      "unit": "kPa",
      "minimum": 100,
      "maximum": 1200,
      "forms": [
        {
          "href": "properties/targetPressure",
          "op": ["readproperty", "writeproperty"]
        }
      ]
    }
  },
  "actions": {
    "start": {
      "input": {
        "type": "object",
        "properties": {
          "mode": {
            "type": "string",
            "enum": ["manual", "automatic"]
          }
        },
        "required": ["mode"]
      },
      "forms": [
        {
          "href": "actions/start",
          "op": "invokeaction"
        }
      ]
    }
  },
  "events": {
    "overload": {
      "data": {
        "type": "object",
        "properties": {
          "motorCurrent": {"type": "number", "unit": "A"},
          "threshold": {"type": "number", "unit": "A"}
        }
      },
      "forms": [
        {
          "href": "events/overload",
          "op": ["subscribeevent", "unsubscribeevent"]
        }
      ]
    }
  }
}
```

这份 TD 描述了：

- 泵站提供什么能力；
- 每项能力交换什么数据；
- 数据必须满足什么约束；
- Consumer 可以执行什么 operation；
- 网络入口在哪里；
- 使用什么安全机制。

它没有描述当前 Servient 如何分配资源。

同一个部署可能还需要一份内部配置：

```yaml
servient:
  max_in_flight_calls: 128
  subscription_buffer_items: 64
  cleanup_budget: 32

bindings:
  http:
    connection_pool_size: 16
    request_timeout_ms: 3000

credentials:
  operator:
    provider_ref: secret-store/operator-token
```

这份配置可能对部署非常重要，但它不是 Thing 接口契约。

TD 可以被另一个实现复用；上述内部配置则可能只对某个 Servient、某个 Binding 甚至某个进程版本有效。

## “设备配置”中的哪些内容可以成为 Property

标题中的“不是设备配置文件”并不意味着设备的所有配置状态都不能进入 TD。

需要区分两类“配置”。

| 内容 | 是否适合进入 TD | 原因 |
|---|---|---|
| 目标压力 | 可以，作为可读写 Property | 是 Thing 对外暴露的状态 |
| 工作模式 | 可以，作为 Property 或 Action 输入 | Consumer 需要理解和操作 |
| 告警阈值 | 视接口设计而定，可以是 Property | 若它是公开、受支持的能力 |
| 固件版本 | 可以作为元数据、Property 或扩展 | 是 Consumer 可观察的 Thing 事实 |
| MQTT 连接池大小 | 不应进入 TD | 是运行时资源策略 |
| 订阅队列容量 | 不应进入 TD | 属于实现与资源治理 |
| Handler 函数名 | 不应进入 TD | 是语言和进程内部实现 |
| 当前 session ID | 不应进入 TD | 是短生命周期运行时状态 |
| API Token 的真实值 | 不应进入 TD | 是调用者私有凭据 |
| 当前 generation | 不应进入 TD | 是运行时生命周期身份 |

判断关键不是某个值是否被称为“配置”，而是它是否构成 Thing 对 Consumer 的稳定、可解释承诺。

## 扩展成员不是把任意配置塞进 TD 的许可证

W3C TD 允许使用 TD Context Extension。

扩展机制非常重要，因为核心词汇不可能覆盖所有领域、协议和安全方案。扩展可以用于：

- 增加领域语义；
- 描述新的 Protocol Binding 元数据；
- 引入新的安全机制；
- 增补 Data Schema 词汇；
- 添加具有明确命名空间的业务信息。

但“TD 允许扩展”不等于“任何内部字段都适合放进去”。

例如：

```json
{
  "x-clinkz-thread-count": 8,
  "x-clinkz-cache-ttl": 30,
  "x-clinkz-handler-pointer": "0x7f0a..."
}
```

即使解析器愿意保留这些字段，它们仍然缺少跨实现语义，也会把 TD 绑定到某个运行时实例。

一个有价值的扩展通常至少应回答：

1. 这个字段属于哪个公开词汇或命名空间？
2. 谁需要理解它？
3. 它对 Thing 的接口或交互有什么可观察影响？
4. 另一个实现是否有机会遵守同一含义？
5. 不理解该扩展时，处理器应该保留、忽略还是拒绝？

ClinkZ-WoT 的 `td` crate 在 `Thing`、`Form`、`DataSchema` 和其他结构中保存 `ExtensionMap`，反序列化时收集未被核心字段消费的成员，重新序列化时再写回。

这体现了一个重要区别：

```text
preserve
  != understand
  != validate extension semantics
  != execute
```

保留扩展成员可以避免 TD 经过一个暂时不理解该词汇的处理器后丢失信息；但是否能够使用这些扩展完成协议交互，仍需要相应的 Binding、扩展处理器或策略支持。

## Thing Description 与 Thing Model 不是同一份文档

另一个常见混淆，是把 Thing Model 当成 TD 的别名，或者把两者都当成设备模板配置。

Thing Description 面向一个可以交互的具体 Thing 实例。它的目标是给 Consumer 提供成功交互所需的具体信息，因此通常会包含实例标识、Form 和安全元数据。

Thing Model 描述一类 Thing 的可复用模型。它更接近能力模板，可以在设备尚未部署、地址尚未确定，或者通信由外部生态统一处理时使用。

例如，Thing Model 可以先规定：

```text
所有这一型号的泵站都必须具有：
- outletPressure Property
- targetPressure Property
- start Action
- overload Event
```

等实例部署后，再为 `Pump Station 17` 补充具体的 `id`、地址、安全要求和 Form，生成可消费的 TD。

W3C TD 1.1 明确允许 Thing Model 缺少或弱化具体通信元数据，并提供扩展、组合、可选 affordance 和 placeholder 等机制。

ClinkZ-WoT 的 `td` crate 也将二者实现为不同类型：

- `Thing` 表示具体 Thing Description；
- `ThingModel` 表示可复用模型；
- `ThingModelForm.href` 可以为空，因为具体地址可能在实例化时才知道；
- 当前 crate 保存和校验 TM 结构，但不负责获取被引用的远程模型，也不自动生成协议专属 Form。

因此：

```text
Thing Model
  = 一类 Thing 应该具有什么能力

Thing Description
  = 某个具体 Thing 现在通过什么接口提供这些能力

Runtime configuration
  = 当前部署如何实现、执行和治理这些交互
```

三者相关，但不能合并成一个没有边界的配置对象。

## 如果把运行时状态写进 TD，会发生什么

把内部配置和执行状态写进 TD，短期看似方便，长期会产生一系列耦合。

### TD 会随着部署细节频繁变化

调整线程数、队列容量或连接池，本应是运行时运维动作。如果这些字段位于 TD 中，每次资源调优都可能被误认为 Thing 接口发生变化。

### Consumer 会依赖不应该知道的实现

业务应用一旦读取 `cacheTtl`、`handlerName` 或 `sessionPoolSize` 来决定行为，就不再只依赖 Thing 契约，而是绑定到某个实现。

### 同一个 Thing 难以迁移和复制

两个 Servient 实例可以提供相同 Thing 接口，却使用完全不同的 executor、连接池和缓存策略。若这些细节进入 TD，就很难判断两份 TD 是否代表同一个接口。

### 生命周期所有权变得模糊

session、subscription cursor、in-flight call 和 generation 都是带生命周期的运行时对象。把它们序列化成字段并不会解决谁拥有、谁取消、谁清理的问题，反而会制造一个看似持久、实际上已经失效的状态副本。

### 凭据和内部拓扑可能泄露

TD 往往会被发现、缓存、分发或存储。把 secret、内网文件路径和内部服务拓扑写进去，会扩大攻击面。

语义契约的价值之一，就是防止实现细节沿着“方便”这条路径不断泄漏到所有消费者中。

## ClinkZ-WoT 如何落实这条边界

ClinkZ-WoT 当前已经实现的 `td` crate 负责：

- TD 与 TM 的协议中立数据结构；
- 构建、序列化和反序列化；
- W3C 结构与本地约束校验；
- 默认值处理；
- 未知扩展成员的 round-trip preservation；
- `no_std + alloc` 下可用的数据模型。

`Thing` 类型直接拥有：

```text
metadata
properties / actions / events
forms
security / securityDefinitions
schemaDefinitions
uriVariables
links
extension fields
```

它没有拥有：

```text
Binding session
connection pool
runtime registry
Handler object
Logical Plan
Binding Artifact
generation
in-flight call
subscription driver
```

这不是遗漏，而是模块边界。

主项目当前 active v5 架构的数据流进一步把 TD 和执行上下文分开：

```text
TD document or ProducedThing draft
        |
        v
parse + preserve extensions + validate
        |
        v
capture immutable planning context
  (policy + registration view + dependency generations)
        |
        v
logical plans + binding-owned artifacts
        |
        v
admitted immutable plan set
        |
        v
runtime interaction
```

TD 提供接口事实；Planning Context 再捕获当前运行时的策略、已注册 Binding、依赖 generation 和其他执行条件。

这种顺序非常重要。

如果策略、注册表和当前 generation 本来就写在 TD 中，运行时便无法清楚地区分：

- 哪些内容是 Thing 自己的接口；
- 哪些内容是当前 Servient 的部署选择；
- 哪些内容是一次规划的不可变快照；
- 哪些内容是执行期间变化的状态。

截至本文基线，`td` crate 的数据模型和校验已经存在；v5 Planning、Logical Plan、Binding Artifact 与完整 Property Read 计划链仍是已接受但尚未完整进入产品源码的架构方向。

## 一个实用的字段归属检查表

准备向 TD 增加一个字段时，可以依次询问：

1. **Consumer 是否需要它才能理解或操作 Thing？**  
   如果只有当前进程的实现代码需要，通常不属于 TD。

2. **它描述的是稳定接口，还是某一时刻的执行状态？**  
   session、in-flight call、重试计数和 generation 属于执行状态。

3. **它是否具有标准词汇或清楚定义的扩展语义？**  
   只有字段名，没有公共含义，不会自动形成互操作契约。

4. **更换 Servient、Binding 或部署拓扑后，它仍然成立吗？**  
   如果更换实现就失效，更可能属于运行时配置。

5. **它是否包含凭据、私钥或其他秘密？**  
   TD 可以描述安全要求，但不应该分发真实秘密。

6. **它是否代表一个需要所有权、取消和清理的活对象？**  
   活对象不能因为被序列化成 JSON 就失去生命周期问题。

可以把最终判断简化为：

```text
Thing 对 Consumer 的可观察承诺
  -> TD

一类 Thing 的可复用能力模板
  -> Thing Model

当前部署如何实现和治理
  -> Runtime / Binding configuration

一次编译或执行产生的状态
  -> Plan / Handle / Session / Registry
```

## 采用语义契约边界需要付出什么

保持 TD 纯粹并不意味着工程会自动变简单。

团队仍然需要：

- 认真区分 Property、Action 和 Event；
- 为数据定义准确的 Schema；
- 维护 Form 与 Protocol Binding 元数据；
- 管理 TD 与 Thing Model 的版本关系；
- 设计独立的运行时配置、凭据和策略系统；
- 为扩展词汇建立命名空间、校验和兼容规则；
- 在 TD 更新后重新规划并处理旧执行状态的生命周期。

把运行时配置排除在 TD 外，并没有消灭这些问题。

它只是让每一类问题拥有正确的 owner：TD 负责接口事实，Thing Model 负责复用模型，Binding 负责协议行为，Servient 负责运行时编排，配置和 secret 系统负责部署材料。

清楚的边界比“一份文件控制一切”多了几个对象，却减少了长期耦合和隐式责任。

## 总结

1. Thing Description 是具体 Thing 面向 Consumer 的机器可读接口契约，主要描述 Property、Action、Event、Data Schema、Form、安全要求、链接和扩展语义。
2. Form 和 Security 属于 TD，是因为 Consumer 完成交互需要访问入口和安全要求；连接池、缓存、真实凭据、Handler、session、generation 和执行状态仍然属于运行时。
3. Thing Model、Thing Description 与 Runtime Configuration 分别描述能力模板、具体接口和部署实现。把三者合并，会让接口随实现细节变化，并破坏复用、互操作和生命周期所有权。

下一篇将继续讨论：同一个 Thing 的同一项能力可以拥有多个 Form 时，Protocol Binding 如何把相同交互意图映射为 HTTP、MQTT、CoAP 或 Zenoh 的具体行为。

## 延伸阅读

- 上一篇：[重新理解 WoT | W3C WoT 到底解决什么问题？](./001-what-does-wot-solve.md)
- 下一篇：重新理解 WoT | 同一个 Thing 如何通过不同协议交互（计划中）
- 相关项目规范：ClinkZ-WoT `docs/architecture/10-primary-data-flows.md`
- 相关源码：ClinkZ-WoT `td` crate

## 项目资料

- ClinkZ-WoT commit: `f453f165c2ea775e5f0d10c36f1e419fcc1d79f3`
- `td/src/lib.rs`
- `td/src/thing.rs`
- `td/src/thing_model.rs`
- `td/src/components/form.rs`
- `td/src/components/data_schema.rs`
- `docs/architecture/10-primary-data-flows.md`

## 外部资料

- [W3C Web of Things (WoT) Thing Description 1.1](https://www.w3.org/TR/wot-thing-description11/)
- [W3C Web of Things (WoT) Architecture 1.1](https://www.w3.org/TR/wot-architecture11/)
- [W3C Web of Things (WoT) Binding Templates](https://www.w3.org/TR/wot-binding-templates/)

<!--
内部事实分类：

- TD、Interaction Affordance、Data Schema、Form、Security、TD Context Extension 和 Thing Model 的定义：外部标准事实。
- “TD 是语义契约”的说法：基于标准信息模型与 Consumer 交互目标的解释性框架，不是 W3C 原文术语。
- ClinkZ-WoT td crate 的 TD/TM 结构、校验、no_std 和扩展成员保留：IMPLEMENTED，依据当前源码。
- Thing、Form、DataSchema 中的 ExtensionMap 与序列化回写：IMPLEMENTED，依据当前源码。
- TD 之后捕获 immutable planning context，再生成 logical plan 和 binding artifact：ACCEPTED_DESIGN。
- v5 Planning、Logical Plan、Binding Artifact 与完整 Property Read 计划链：PLANNED / NOT YET IMPLEMENTED。
- 泵站 TD 与 runtime YAML：CONCEPTUAL ILLUSTRATION，不代表当前稳定 API 或配置格式。

发布前检查：

- [x] 已读取主项目 PROJECT_STATE.md 和当前架构数据流
- [x] 已检查 W3C TD 1.1、Architecture 1.1 与 Binding Templates
- [x] 已检查 td crate 的 Thing、ThingModel、Form、DataSchema 与 ExtensionMap 边界
- [x] 已记录完整 commit
- [x] 已区分 implemented / accepted / planned
- [x] 未将真实凭据建议写入 TD
- [x] 未把扩展成员等同于运行时自动理解
- [x] 未虚构当前稳定 Rust API 或运行时配置格式
- [ ] 完成作者理解校验
- [ ] 完成第二轮技术事实校验
- [ ] 压缩知乎版本篇幅和重复解释
- [ ] 回填知乎发布信息
-->
