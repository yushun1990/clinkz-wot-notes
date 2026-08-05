---
id: "ARCH-001"
title: "ClinkZ-WoT 架构｜我心目中的完整 WoT 引擎"
subtitle: "从 W3C WoT 规范出发，梳理一个 Runtime 应具备的能力、边界与生命周期"
series: "ClinkZ-WoT 架构设计"
series_order: 1
status: "DRAFTING"
author: "yushun1990"
created: "2026-08-05"
updated: "2026-08-05"
summary: "W3C WoT 规范定义了 Thing Description、交互模型、发现和协议绑定等外部语义，但不会替实现者完成内部 Runtime 架构。本文结合 ClinkZ-WoT 的设计过程，给出我心目中一个完整 WoT 引擎的能力地图，并说明声明式描述、协议中立、服务暴露、异步生命周期和受限资源为何构成真正的实现难点。"

clinkz_wot:
  repository: "https://github.com/yushun1990/clinkz-wot"
  branch: "master"
  commit: "30485b1a51470f328e79453ba0e82e3358c14f79"
  inspected_at: "2026-08-05"

publication:
  platform: "zhihu"
  published_at: null
  canonical_url: null

related:
  previous: "WOT-002"
  next: "ARCH-002"
  articles:
    - "WOT-001"
    - "WOT-002"
    - "ARCH-002"
    - "ARCH-003"
    - "ARCH-004"
  docs:
    - "docs/spec/planning.md"
    - "docs/spec/binding-spi.md"
    - "docs/architecture/30-compiled-plan-lifecycle.md"
    - "docs/architecture/50-servient-runtime-lifecycle.md"
    - "docs/work-packages/WP-500-discovery.md"
    - "docs/work-packages/WP-600-protocol-bindings.md"
  source: []
  tests: []
---

# ClinkZ-WoT 架构｜我心目中的完整 WoT 引擎

> 本文基于 ClinkZ-WoT commit `30485b1a51470f328e79453ba0e82e3358c14f79`。
>
> 本文讨论的是我在设计 ClinkZ-WoT 过程中逐渐形成的总体架构认识，不是对 W3C 规范目录的逐项翻译，也不声称这是唯一正确的 WoT Runtime 实现。文中会明确区分 W3C WoT 定义的外部语义与 ClinkZ-WoT 为实现这些语义而选择的内部结构。

前两篇文章分别讨论了 W3C WoT 为什么存在，以及现场设备不原生支持 WoT 时，Thing Description 如何由业务模型、设备映射、网关和平台共同形成。

理解这些概念以后，接下来终于可以回到这个专栏真正想做的事情：

> 如果要从零实现一个 WoT 引擎，它到底应该是什么样的？

从规范走向实现时，首先需要确定的并不是模块数量，而是规范定义的外部语义如何在 Runtime 内部获得稳定、一致的解释。

TD 数据模型、Property、Action、Event API 和 Protocol Binding 都是必要组成部分，但它们还不能自动构成一套完整的运行时。真正运行起来以后，引擎还必须处理：一份声明式 TD 怎样形成稳定的执行结构；多个协议组件怎样共享统一语义，又保留各自真实的通信特性；本地 Thing 对外暴露失败时怎样回滚；调用取消以后谁继续持有协议资源；TD 更新以后旧请求和旧订阅怎样退出；同一套行为怎样同时运行在普通服务器和受限设备上。

这些问题通常不会出现在最初的演示代码中，却会决定一个 Runtime 能否长期演进。

这篇文章想先给出一张总体地图。后续文章再逐项展开其中的 Planning、Logical Plan、Binding Artifact、Servient、Protocol Binding 和生命周期设计。

## W3C WoT 给出的不是一套内部模块图

W3C WoT Architecture 1.1 描述了 Thing、Consumer、Producer、Servient、Protocol Binding、Intermediary 和 Discovery 等抽象角色；Thing Description 1.1 定义了 Thing 的元数据、Property、Action、Event、Data Schema、Security Definition 和 Form；WoT Discovery 定义了怎样发现并取得 TD。

这些规范共同建立了 WoT 的外部世界：

```text
Thing 可以提供什么能力
Consumer 可以进行什么交互
这些交互通过什么 Form 暴露
使用什么数据结构和安全声明
TD 如何被发现和取得
```

但规范不会替一个 Rust 项目决定：

- 是否应在热路径中直接遍历 TD；
- Form 选择结果由谁冻结；
- Protocol Binding 能否访问完整 TD；
- Handler 注册表由哪个对象持有；
- 一次取消由 Future Drop 还是显式状态机处理；
- ProducedThing 的多个协议入口怎样一致地发布；
- 旧 generation 的迟到结果怎样与新状态隔离；
- `no_std + alloc` 环境怎样推进同样的交互语义。

这些都属于实现者必须完成的架构工作。

因此，这篇文章所说的“完整”，不是规范条目的覆盖率，也不是协议数量，而是运行时责任是否闭合：

> 在遵守 WoT 外部语义的前提下，描述、发现、消费、暴露、协议执行、安全、数据处理和异步生命周期必须被组织在同一套可验证的运行时规则中；每个状态、决策和外部副作用都应有明确的解释者、拥有者和终止条件。

这种完整性只能相对于引擎已经声明支持的能力来讨论。一个当前只支持 Zenoh 的 Runtime，只要调用、订阅、暴露、取消和清理能够形成完整闭环，它的边界就是清楚的；相反，接入更多协议却让每个 Binding 各自维护选择、回调、重试和后台状态，并不会自动形成一个统一的 WoT Runtime。

## 我心目中的总体能力地图

从外部看，一个 WoT Runtime 应该连接三类对象：

```text
                 Thing Description / Discovery
                              |
                              v
Application <-> WoT Runtime <-> Protocol Bindings <-> Remote or Local Things
                              |
                              v
                    Handlers and Produced Things
```

从内部责任看，它至少需要下面这些彼此独立、又能组成闭环的部分。

## 一、完整保存和理解 Thing Description

TD 是 WoT 的语义入口。对 Runtime 而言，关键不在于能否反序列化文档，而在于如何在强类型访问、规范默认规则与开放扩展之间保持无损、可验证的表示。

引擎需要处理：

- Thing 与 Thing Model；
- Property、Action 和 Event；
- Data Schema、单位、范围和枚举；
- Form、operation、`contentType` 和响应描述；
- Security Definition 与安全引用；
- `base`、URI Template、继承和默认规则；
- JSON-LD 上下文、语义标注和扩展成员；
- 校验、诊断以及未知信息的无损保留。

这里存在一个容易被忽略的冲突。

Rust 强类型希望尽可能早地把数据收敛成明确结构；WoT TD 又允许扩展词汇和不同协议的附加成员。如果解析层为了让类型更“干净”而丢弃未知字段，后面的 Binding 就再也无法看见它真正需要的协议信息。

因此，TD 层既要提供强类型访问，也要保留规范允许的开放性。它负责忠实表达文档，而不是提前替 Runtime 做执行决策。

## 二、消费远程 Thing

Consumer 获取 TD 后，需要得到一个可以交互的对象。无论最终 API 是否完全仿照 WoT Scripting API，它至少要表达这些能力：

```text
read / write / observe Property
invoke / query / cancel Action
subscribe / unsubscribe Event
执行 Thing 级批量或集合操作
```

应用面对的应该是 WoT 操作，而不是某种具体协议：

```text
read_property("outletPressure")
```

而不是：

```text
发送 HTTP GET
发布 MQTT 请求 Topic
执行 Zenoh Query
```

但“隐藏协议”并不代表协议消失了。Runtime 仍然要处理：

- 操作对应哪些有效 Form；
- 数据怎样编码和解码；
- 输入、输出和响应是否符合 Schema；
- 使用哪一个安全分支和凭证；
- 协议状态怎样映射为应用可理解的结果；
- 调用、观察和订阅怎样取消并完成清理。

ConsumedThing 提供统一的应用语义，真正的协议工作则必须交给对应 Binding。

## 三、对外暴露本地 Thing

一个完整的 WoT 引擎不能只作为客户端消费别人的 Thing，还要能够把本地能力暴露为 ProducedThing。

应用可能先注册 Handler：

```text
read outletPressure -> local handler
write targetPressure -> local handler
invoke start         -> local handler
emit overload        -> local event source
```

Runtime 再通过一个或多个 Binding 将它们暴露出去。

这看起来像“服务端版本的 ConsumedThing”，但两者的生命周期并不对称。

消费一次远程属性失败，通常只影响一次调用。暴露本地 Thing 时，Runtime 可能已经：

- 绑定了端口；
- 注册了路由；
- 建立了 Broker 订阅；
- 发布了可被外部发现的 TD；
- 接收了部分请求；
- 创建了需要显式关闭的协议资源。

如果一个 Thing 同时通过多个 Binding 暴露，失败还可能发生在中间：

```text
HTTP route prepared
Zenoh route prepared
MQTT route failed
```

此时不能简单返回一个错误，然后假设一切都恢复到了调用前。

ProducedThing 的暴露必须是一套可验证的生命周期：准备、就绪、激活、发布、停止、回滚和最终清理都要有明确的拥有者。它不一定能够让整个网络世界获得真正的分布式原子性，但至少必须让一个 Servient 内部的发布状态保持一致，并诚实记录残留的外部副作用。

## 四、发现 Thing，但不把 Directory 塞进 Runtime

WoT Discovery 解决的是怎样找到和取得 TD。一个 Runtime 应该能够：

```text
通过直接地址获取 TD
通过 well-known 或其他介绍机制发现 TD
查询 Thing Description Directory
验证并导入发现结果
创建 ConsumedThing
```

但 Discovery 客户端和 Directory 服务端不是同一项责任。

Directory 服务还涉及：

- TD 存储和索引；
- 查询语言与分页；
- 身份认证和授权；
- 多租户隔离；
- revision、watch 和 publication；
- 数据库、集群和运维策略。

这些是平台服务的责任，不应该因为 Runtime 需要“发现 Thing”，就全部进入引擎内核。

所以我更倾向于让 ClinkZ-WoT 拥有 Discovery 和 Directory 客户端边界，并把获取到的 TD 送入与普通 `consume` 相同的验证和 Planning 路径；完整 Directory 服务则由独立平台组件实现。

## 五、在执行前形成稳定的计划

TD 是声明式描述，不是可以直接交给协议栈执行的指令。

真正执行前，Runtime 必须把分散在 TD 各处的信息组合起来：

- 目标是 Thing、Property、Action 还是 Event；
- 当前操作是 `readproperty`、`invokeaction` 还是 `subscribeevent`；
- `base` 与 `href` 组合后的真实目标是什么；
- operation 默认规则如何生效；
- 哪些安全声明和 scope 有效；
- `contentType`、响应和 Schema 如何处理；
- 哪些已注册 Binding 能够支持这个候选；
- 协议专属的地址、Topic、Key Expression 或选项怎样编译；
- 计划、候选和临时工作是否超过资源上限。

最直觉的实现，是在每次调用时重新遍历 TD，再让某个 Binding 自己完成剩余解释。

这种方式可以很快做出原型，但它会逐渐产生三类问题：

第一，不同 Binding 可能重复实现 `base`、默认 operation、安全继承和 Form 解析，并得出不一致结果。

第二，调用热路径不断处理本可提前完成的工作，很多错误只能在协议副作用已经开始后才暴露。

第三，Binding 如果拿到完整 TD，就同时获得了重新选 Form、重新解释安全和绕过统一策略的机会。

因此，ClinkZ-WoT 选择把执行前的结果分为两层：

```text
Logical Plan
  协议中立、可共享的交互事实

Binding Artifact
  某个具体 Binding 编译出的协议专属数据
```

再由一个 Servient 拥有的 Compiled Plan Set 管理它们的发布、引用、draining 和回收。

这套设计并不是 W3C 规定的内部名词，而是 ClinkZ-WoT 为了让执行决定可验证、可复用并且具有明确生命周期而选择的实现结构。

## 六、让 Protocol Binding 专注协议，而不是成为另一个 Runtime

Protocol Binding 是协议中立架构里最容易失控的边界。

它必须拥有足够多的能力，才能真实实现 HTTP、MQTT、CoAP 或 Zenoh：

- 协议地址和语法；
- 报文、Topic、Key Expression 与 framing；
- 编码和协议状态映射；
- correlation；
- client、listener、session 和 route；
- 原生订阅、流控、重试和协议本地缓存；
- transport authentication material；
- 协议资源的创建和清理。

但 Binding 不应该拥有：

- Servient 的 Thing 和 Handler 注册表；
- 重新解释整份 TD 的权力；
- 全局 Form 和安全策略选择权；
- 任意调用应用 Handler 的 dispatch capability；
- 跨 Binding 的调度、公平性和资源治理；
- 旧 generation 是否仍然有效的最终判断权。

否则每个 Binding 都会逐渐发展出自己的计划、路由、取消和订阅模型，所谓“协议中立 Runtime”最终只是多个协议子系统外面套了一层统一 API。

我心目中的 Binding 更像一个受约束的运行时组件：它参与协议专属计划编译，也负责真实 I/O 和协议资源，但它消费的是 Runtime 已经确定的交互事实，并通过受控状态转换把结果交还给引擎。

## 七、安全与数据处理必须跨越多层，但不能无人负责

WoT TD 可以描述安全方案、数据结构、媒体类型和响应信息，但这些声明最终要穿过多个组件才能产生实际效果。

一次安全交互大致包含：

```text
TD 声明安全需求
        |
Runtime 选择有效安全分支
        |
凭证提供者提交具体材料
        |
Binding 映射到协议 Header、握手或传输认证
        |
响应返回后重新验证身份、状态和数据
```

如果把全部安全责任都交给 Binding，它可能无法理解应用级 scope 和 TD 安全组合；如果 Core 直接处理所有协议凭证，又会把 HTTP Header、TLS 参数或协议握手细节带入协议中立层。

数据处理也类似：

- TD 和 Planning 确定 Data Schema、media type 和响应约束；
- codec 负责具体编码与解码；
- Binding 负责协议承载；
- Runtime 在正确的阶段完成输入、输出和响应验证。

完整引擎需要明确每一层拥有什么信息、在哪个阶段失败，以及失败以后是否已经产生外部副作用。

## 八、Servient 必须成为运行时权威

当 TD、Planning、Binding、Handler、Discovery、安全和生命周期都进入同一系统后，必须有一个明确的整体拥有者。

这里的“权威”不是指 Servient 亲自实现所有工作，而是所有影响全局语义和生命周期的决定必须由同一个运行时边界编排和验证。

Servient 至少需要拥有或协调：

- startup-only 的 Binding 组合与注册快照；
- ConsumedThing 与 ProducedThing generation；
- Handler 和 route 的权威映射；
- Compiled Plan Set 的构建、发布和回收；
- 请求、订阅和暴露操作的 admission；
- deadline、取消原因和 late completion；
- 新旧 generation 的隔离；
- draining、cleanup 和残留状态记录；
- 跨 Binding 的资源预算和公平推进。

没有这个权威，系统很容易变成：Planning 选择一个候选，Binding 又重新选择另一个；Servient 删除了 Thing，后台任务仍继续向旧 Handler 回调；调用方丢弃 Future 后，协议资源没有任何对象负责完成清理。

一个漂亮的 trait 并不能自动解决这些问题。真正重要的是每个状态在任何时刻都能回答：

```text
谁拥有它？
谁可以推动它？
谁可以取消它？
失败后由谁继续清理？
何时才允许真正回收？
```

## 九、相同语义必须能够运行在不同资源环境中

ClinkZ-WoT 不只希望运行在有 Tokio、动态分配和后台任务的 Host 环境，也希望核心语义能够进入 `no_std + alloc` 或更受限的运行环境。

这不能只靠删除 `std` 依赖完成。

Host 环境可以方便地使用：

- 动态分配；
- trait object；
- executor task；
- channel 和 waker；
- 连接池和后台 reactor。

受限环境可能只能使用：

- 调用方提供的固定 slot；
- 手工 poll；
- 明确的 WorkBudget；
- 预先声明的最大对象尺寸；
- 固定数量的请求、订阅和路由状态。

如果两种环境各自实现一套业务状态机，它们最终会形成两套语义：Host 版本支持完整取消，受限版本直接丢弃；Host 版本保存迟到响应，受限版本覆盖 slot；Host 版本隐式重试，受限版本没有对应行为。

因此我的目标不是让二者拥有相同的存储方式，而是：

```text
相同的状态转换和可观察结果
不同的分配、调度和推进机制
```

这也是为什么资源上限不能只作为性能优化。队列长度、并发调用、订阅 buffer、correlation 表、计划大小、Binding Artifact、临时解析内存和每次 poll 的工作量，都必须成为可以声明、验证和测试的运行时契约。

## 真正困难的不是“调用成功”，而是生命周期闭合

把前面的能力放在一起后，我认为实现 WoT Runtime 最困难的并不是让一次 Property Read 返回正确值，而是下面五组矛盾。

### 声明式描述与可执行结构

TD 必须保持开放、可扩展和可交换；执行计划则必须稳定、受限并适合热路径。Runtime 需要在二者之间建立明确的编译边界。

### 协议中立与协议真实性

应用不应该绑定 HTTP、MQTT 或 Zenoh，但 Runtime 也不能假装这些协议拥有相同的请求数量、流控、取消、缓存和订阅语义。抽象必须统一 WoT 意图，而不是抹平协议事实。

### Consumer 与 Producer 的生命周期不对称

消费远程 Thing 主要面对调用和订阅；暴露本地 Thing 会创建外部可见路由、监听和发布副作用，需要更严格的准备、激活、回滚和清理协议。

### 异步取消与资源所有权

调用方离开不代表协议工作已经停止。取消、超时、迟到结果、重试、cleanup failure 和旧 generation 都需要保留完整拥有者，不能用“Drop Future”代替状态机。

### Host 与受限环境的语义一致性

两种环境可以有完全不同的内存和推进方式，但必须共享同一组状态转换、终态分类和资源责任。

这五组矛盾也是 ClinkZ-WoT 架构不断变复杂的主要原因。

复杂并不总是好事，很多抽象也可能设计过度。但如果某项复杂度对应一个真实的生命周期、资源或权责问题，删除类型并不会删除问题，只会让它重新变成隐式约定。

## ClinkZ-WoT 如何组织这些责任

基于当前 active v5.0 设计，ClinkZ-WoT 大致把责任划分为：

| 边界 | 主要责任 |
|---|---|
| `clinkz-wot-td` | TD/TM 数据结构、校验、默认规则和扩展信息保留 |
| `clinkz-wot-core` | 协议中立身份、请求、结果、状态和 Binding SPI 契约 |
| `clinkz-wot-planning` | 有效 Form、Logical Plan、候选和编译协调 |
| 具体 Protocol Binding crate | 协议能力、Binding Compiler、Artifact 与协议执行 |
| `clinkz-wot-servient` | 注册快照、计划发布、admission、路由、generation 和生命周期权威 |
| `clinkz-wot-discovery` | Discovery 与 Directory 客户端边界 |

这张表不是完整依赖图，也不代表每个包已经完成。它表达的是一个更重要的原则：

> 一个事实只能有一个最终解释者，一项生命周期只能有一个明确拥有者。

例如：

- W3C 默认规则由 TD/Planning 统一处理，Binding 不重复解释；
- 协议专属地址和操作数据由具体 Binding 编译，但它不能重新选择 Form；
- Handler 由 Servient 路由，Binding 不获得隐藏 dispatch 权限；
- 计划由 Servient 发布和回收，调用和路由通过 generation-bearing lease 保持有效性；
- Directory 服务留在平台层，Runtime 只消费发现结果。

## 当前实现到了哪里

截至本文使用的项目基线，ClinkZ-WoT 已经拥有 TD/TM 数据结构、序列化与校验基础，以及 Planning、Binding Compiler 和 Property Read 方向的一部分可构造源码切片。Producer 路由的计划投影也已经进入 WP-200 的已完成范围。

但这不能表述为“完整 WoT 引擎已经完成”。

当前更准确的状态是：

- **已经实现并有源码证据**：TD 基础能力，以及部分 Property Read Planning 与 Binding 编译契约；
- **已经成为 active v5.0 设计权威**：Compiled Plan Set、完整 Binding registration、Servient authority、generation、取消和有界资源等架构边界；
- **仍在后续 Work Package 中推进**：完整客户端和服务端执行链、广泛交互能力、Discovery 客户端、生产级 Zenoh/zenoh-pico Binding 与端到端集成。

因此，这篇文章描述的是 ClinkZ-WoT 正在实现的总体架构，而不是一份已经全部通过产品代码证明的功能清单。

## 这张地图会继续变化

“我心目中的完整 WoT 引擎”不是一句品牌口号，也不是提前宣布项目已经完成。

它更像一组当前必须守住的设计判断：

1. **WoT 语义必须独立于具体协议，但不能否认协议真实差异。**
2. **所有异步工作和外部副作用都必须拥有可追踪的生命周期与清理责任。**
3. **Host 与受限环境可以采用不同机制，但不能形成两套行为不同的 Runtime。**

后续实现可能继续推翻某些 Rust 类型、模块边界和 API 形式，但如果新的设计无法回答“谁拥有、谁推进、谁取消、谁清理”，它就还没有真正替代旧方案。

下一篇将从这张总体地图中选择第一个核心问题继续展开：

> 为什么 WoT Runtime 不应该在每次调用时重新解释 TD，而需要提前生成稳定的执行计划？

---

## 延伸阅读

### W3C WoT

- [Web of Things (WoT) Architecture 1.1](https://www.w3.org/TR/wot-architecture11/)
- [Web of Things (WoT) Thing Description 1.1](https://www.w3.org/TR/wot-thing-description11/)
- [Web of Things (WoT) Discovery](https://www.w3.org/TR/wot-discovery/)
- [Web of Things (WoT) Scripting API](https://www.w3.org/TR/wot-scripting-api/)
- [Web of Things (WoT) Security and Privacy Guidelines](https://www.w3.org/TR/wot-security/)

### ClinkZ-WoT

- [ClinkZ-WoT repository](https://github.com/yushun1990/clinkz-wot)
- `docs/spec/planning.md`
- `docs/spec/binding-spi.md`
- `docs/architecture/30-compiled-plan-lifecycle.md`
- `docs/architecture/50-servient-runtime-lifecycle.md`
- `docs/work-packages/WP-500-discovery.md`
- `docs/work-packages/WP-600-protocol-bindings.md`

---

## 内部事实分类

- **规范事实**：W3C WoT Architecture 1.1、Thing Description 1.1 和 Discovery 定义的角色、信息模型与发现语义。
- **作者观点**：“完整”应以权责和生命周期闭合衡量，而不是仅以协议数量或 API 数量衡量。
- **已接受设计**：ClinkZ-WoT active v5.0 中的 Planning、Binding、Servient authority、generation 与有界资源边界。
- **已实现事实**：TD 基础能力，以及当前 Property Read Planning/Binding 的阶段性源码切片。
- **计划中能力**：完整 Discovery、广泛运行时执行、生产 Zenoh/zenoh-pico Binding 与端到端集成。
