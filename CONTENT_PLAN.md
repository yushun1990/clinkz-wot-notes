# Content Plan

本文档保存文章系列、推荐顺序和选题状态。它是写作路线，不是
ClinkZ-WoT 的开发计划。

## 专栏

**从零设计一个 Rust WoT Runtime**

副标题：

> ClinkZ-WoT 的架构、实现与 AI 协作开发实录

## 第一季：建议优先完成的 12 篇

| ID | 标题 | 系列 | 状态 |
|---|---|---|---|
| WOT-001 | 为什么物联网平台不应该从 MQTT Topic 开始设计 | WoT 基础 | IDEA |
| WOT-002 | W3C WoT 不是一种新协议 | WoT 基础 | IDEA |
| WOT-003 | Thing Description 是语义契约，而不是设备配置文件 | WoT 基础 | IDEA |
| ARCH-001 | 从 TD 到一次属性读取：ClinkZ-WoT 的完整执行链 | 运行时架构 | IDEA |
| ARCH-002 | 为什么 WoT Runtime 需要预编译执行计划 | 运行时架构 | IDEA |
| ARCH-003 | Logical Plan 与 Binding Artifact：协议中立如何落地 | 运行时架构 | IDEA |
| ARCH-004 | Servient 为什么必须成为运行时权威 | 运行时架构 | IDEA |
| ARCH-005 | Protocol Binding 为什么不能直接调用 Handler | 运行时架构 | IDEA |
| ARCH-006 | 为什么 V1 放弃动态插件，选择 Cargo 静态链接 | 运行时架构 | IDEA |
| RUST-001 | Generation 如何阻止异步旧结果破坏新状态 | Rust 运行时 | IDEA |
| RUST-002 | 为什么所有队列、缓存和后台工作都必须有上限 | Rust 运行时 | IDEA |
| AI-001 | 我如何用 AI 推进一个复杂 Rust 架构项目 | AI 工程 | IDEA |

推荐先从 `WOT-001` 开始。它面向更广泛的物联网开发者，也能自然引出
WoT、TD、Form、Planning 和 Protocol Binding。

---

## 系列一：WoT 基础与语义模型

### WOT-001｜为什么物联网平台不应该从 MQTT Topic 开始设计

核心问题：

- Topic 为什么是通信地址而不是业务语义；
- 协议结构如何渗透应用；
- Property、Action、Event 如何建立协议之上的交互模型；
- ClinkZ-WoT 为什么从 TD 开始规划。

### WOT-002｜W3C WoT 到底解决什么问题？它不是另一种物联网协议

核心问题：

- WoT 与 MQTT、HTTP、CoAP、Zenoh 的关系；
- TD 与 Protocol Binding 的分工；
- 为什么 WoT 是编程模型和互操作层。

### WOT-003｜Thing Description 不是设备配置文件，而是语义契约

核心问题：

- Property、Action、Event；
- DataSchema、Form、Security；
- 扩展成员；
- 为什么运行时连接和缓存不应进入 TD。

### WOT-004｜一个 Thing 同时提供多个 Form，客户端应该选哪个

核心问题：

- operation；
- security；
- binding availability；
- policy；
- candidate fallback；
- 为什么选择结果必须进入 Plan。

### WOT-005｜ConsumedThing 与 ProducedThing：相同语义，不同生命周期

核心问题：

- 消费远程 Thing；
- 对外暴露本地 Thing；
- 为什么 producer 暴露涉及外部副作用和事务性激活。

### WOT-006｜WoT Directory 是什么，为什么 Runtime 不应实现 Directory 服务

核心问题：

- Discovery client 与 Directory service；
- 存储、索引、查询、多租户和权限为何属于平台层。

---

## 系列二：ClinkZ-WoT 运行时架构

### ARCH-001｜从 TD 到一次属性读取：完整执行链

主线：

```text
TD
 -> parse and validate
 -> PlanningContext
 -> Logical Plan
 -> Binding Artifact
 -> Compiled Plan Set
 -> OutboundRequest
 -> Binding
 -> response validation
 -> application
```

### ARCH-002｜为什么 WoT Runtime 需要预编译执行计划

核心问题：

- 热路径解释 TD 的代价；
- form、security、schema 和 media decision；
- 编译期错误与运行期错误；
- immutable plan；
- 这里的“编译”为什么不是 Cargo build。

### ARCH-003｜Logical Plan 与 Binding Artifact：协议中立如何落地

核心问题：

- 哪些决定属于协议中立；
- 哪些产物必须由具体 Binding 拥有；
- HTTP method/URL、Zenoh key expression、MQTT topic 的边界。

### ARCH-004｜Servient 为什么必须成为运行时唯一编排者

核心问题：

- 谁拥有 registry；
- 谁选择 Handler；
- 谁管理 admission、generation、cancellation 和 cleanup；
- 为什么 Binding 不能持有通用 dispatch authority。

### ARCH-005｜Protocol Binding 为什么不能直接调用 Handler

使用一次 inbound Zenoh 请求画完整时序：

```text
protocol frame
 -> Binding
 -> protocol-neutral request
 -> Servient admission and routing
 -> Handler
 -> protocol-neutral response
 -> Binding correlation and emission
```

### ARCH-006｜为什么 V1 放弃动态插件，选择 Cargo 静态链接

记录路线演进：

- `.so` 动态加载愿景；
- Rust ABI、async、trait object、卸载和资源所有权；
- WASM 备选；
- Cargo-linked Binding；
- 重新构建与灰度发布。

### ARCH-007｜build.rs 能不能实现 Rust 插件系统

核心问题：

- Cargo 依赖解析发生在何时；
- build script 的边界；
- 为什么不能在 build.rs 中临时添加 crate dependency；
- 平台管理器如何生成组合工程。

### ARCH-008｜静态链接不等于不能动态扩展

平台级流程：

```text
install binding
 -> update manifest
 -> build new Servient image
 -> verify
 -> canary deployment
 -> drain old instance
 -> switch traffic
```

### ARCH-009｜一个 ProducedThing 的暴露为什么需要事务

核心问题：

- prepare、ready、activate、commit；
- committed-closed；
- atomic publication；
- partial serving 风险；
- rollback 和 cleanup。

### ARCH-010｜Compiled Plan Set 生命周期：删除对象不等于资源已经释放

核心问题：

- Building、Frozen、Published、Draining、Reclaimed；
- call、route、subscription 和 late completion；
- plan pin 与 cleanup owner。

### ARCH-011｜Protocol Binding 是适配器，还是一个受约束的运行时组件

核心问题：

- compiler extension；
- client call；
- server route；
- subscription driver；
- emission、correlation、resource footprint；
- complete registration bundle。

### ARCH-012｜为什么 Core 中不应该有一个万能 EventBroker

核心问题：

- 应用订阅、协议发布、本地 fan-out 和远程事件不是同一个问题；
- Servient 与 Binding 的责任；
- 为什么 Core 不应锁死队列实现。

---

## 系列三：Rust 运行时机制

### RUST-001｜Generation 如何阻止异步旧结果破坏新状态

用旧 generation 的 late callback 与新 generation 的资源重建说明问题。

### RUST-002｜为什么所有队列、缓存和后台工作都必须有上限

覆盖：

- unbounded channel；
- correlation map；
- subscription buffer；
- retry queue；
- ingress buffer；
- diagnostics；
- host 与 constrained profile。

### RUST-003｜异步取消不是 Drop Future

核心问题：

- caller drop；
- transport side effect；
- late response；
- cancellation cause；
- cleanup completion。

### RUST-004｜为什么外部回调必须在引擎锁外执行

两阶段或 lease 模型：

```text
under lock: validate, reserve, snapshot, acquire lease
outside lock: invoke user/binding/codec code
under lock: revalidate generation and commit
```

### RUST-005｜HandlerContext 应该包含什么，不应该包含什么

核心问题：

- deadline；
- cancellation；
- identity；
- security principal；
- plan/route context；
- resource view；
- 为什么不能暴露整个 Servient。

### RUST-006｜Deadline 为什么不能只是一个 u64

核心问题：

- clock domain；
- ClockId；
- wrap；
- extended logical tick；
- incomparable clocks；
- deadline 与 wall clock。

### RUST-007｜no_std 不只是移除标准库

核心问题：

- host async 与 caller-driven progress；
- static/dynamic bounded resources；
- 相同语义、不同 progress mechanism；
- async 不等于 Tokio。

### RUST-008｜Pull-based Subscription 为什么比回调更适合运行时边界

核心问题：

- 执行权；
- 背压；
- receive cursor；
- cancellation；
- application executor；
- protocol-native flow control。

### RUST-009｜Subscription 为什么不应该 Clone

核心问题：

- broadcast 还是 competing consumer；
- 谁拥有 stop；
- 谁拥有 cursor；
- 谁负责 cleanup；
- 多个消费者为何需要多个 SubscriptionId。

### RUST-010｜subscribe_all_events 为什么不能循环订阅每个 Event

核心问题：

- N 个资源；
- 协议原生多主题能力；
- Thing-level collection operation；
- native/coalesced driver。

### RUST-011｜WorkBudget：没有统一 Executor 时如何表达公平性

核心问题：

- bounded step；
- ready queue；
- round-robin cursor；
- slow binding isolation；
- host/manual runtime 一致性。

### RUST-012｜Fallible Cleanup：为什么 Drop 不能成为唯一清理协议

核心问题：

- 网络关闭可能失败；
- route shutdown；
- cleanup record；
- terminal disposition；
- Drop 只能是兜底。

---

## 系列四：人与 AI 的工程协作

### AI-001｜我如何让 AI 主导一个 Rust 架构项目，而不是只生成代码

核心问题：

- Owner 与 AI 的责任；
- 技术判断；
- 项目约束；
- evidence；
- 不把技术决策重新甩给 Owner。

### AI-002｜聊天记录不是项目记忆

核心问题：

- 会话中断；
- 模型额度；
- repository as durable carrier；
- PROJECT_STATE 应保存什么。

### AI-003｜AGENTS、PLAN 和 PROJECT_STATE 分别应该写什么

比较：

- 稳定规则；
- 路线与里程碑；
- 当前连续状态；
- workspace；
- docs；
- source/tests。

### AI-004｜如何避免 AI 在大型项目中不断重新设计架构

核心问题：

- authority hierarchy；
- architecture backbone；
- ADR；
- normative owner；
- requirement index；
- acceptance gate。

### AI-005｜Workspace 不是垃圾场：问题如何迁移成规范

生命周期：

```text
OPEN -> DISCUSSING -> DECIDED -> MIGRATED
```

### AI-006｜设计文档越写越长，可能意味着权责尚未收敛

核心问题：

- 巨型 design 文档的早期价值；
- 后期冲突；
- decomposition；
- single owner；
- 渐进迁移。

### AI-007｜为什么代码正确也不能直接合并

核心问题：

- architecture gate；
- lifecycle；
- ownership；
- resource contract；
- evidence；
- tests 只是证据之一。

### AI-008｜不是所有改动都需要相同的架构审查

核心问题：

- Category A/B/C；
- 风险与审查深度；
- 小改动不过度治理；
- 大改动不能悄悄进入。

### AI-009｜如何把庞大重构拆成 Work Package DAG

核心问题：

- dependency；
- tranche；
- admission；
- exact scope；
- evidence；
- independent attestation。

### AI-010｜用一个 Property Read 证明整个架构

主线：

```text
Planner
 -> Binding
 -> Servient
 -> Handler
 -> Response
 -> Generation
 -> Cleanup
```

强调纵向组合证明比并行创建大量孤立 trait 更早暴露架构错误。

---

## 暂不写成“最终方案”的主题

在主项目对应 gate 和实现尚未完成前，以下内容只能写设计探索或阶段性结论：

- 最终公开 Handler API；
- 最终 Protocol Binding Rust trait；
- 最终 Servient 应用 API；
- Zenoh Binding 完整使用教程；
- 百万设备容量结论；
- 性能 benchmark 结论；
- 完整 ESP32/no_std 示例；
- 动态升级和灰度发布的已实现教程。

AI 每次写作前必须重新读取主项目状态，不能依赖本列表判断项目是否已经完成。
