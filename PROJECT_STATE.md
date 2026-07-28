# Writing Project State

Last updated: 2026-07-28

## Current Objective

完成第一篇文章的作者理解校验和第二轮技术校验，将新稿推进到可发布状态：

> WOT-001｜重新理解 WoT | W3C WoT 到底解决什么问题？

## Repository Status

- GitHub repository: `https://github.com/yushun1990/clinkz-wot-notes`
- Default branch: `main`
- 专栏导读和第一季文章地图已经按新的 WoT 叙事更新。
- WOT-001 新稿已经写入：
  `articles/01-wot-foundations/001-what-does-wot-solve.md`
- WOT-001 当前状态：`DRAFTING`。
- 原“为什么物联网平台不应该从 MQTT Topic 开始设计”不再作为第一篇正式选题；其内容保留在 Git 历史中，可在后续讨论 Protocol Binding 或平台自定义应用协议时复用。
- 尚未登记知乎 canonical URL。

## Stable Decisions

- 文章仓库与实现仓库分离。
- 专栏名称为 **从零开发一个 Rust WoT 引擎**。
- 文章系列按“系列号.系列内序号”组织，第一季当前收录 12 篇文章。
- 第一篇正式标题为：
  **重新理解 WoT | W3C WoT 到底解决什么问题？**
- 第一篇不得从 MQTT、HTTP、CoAP、Zenoh 或任何单一协议建立 WoT 的问题空间。
- MQTT、HTTP、CoAP、Zenoh 与 WoT 不在同一抽象层级；它们只在解释 Form 与 Protocol Binding 时作为具体协议示例出现。
- 系列一前三篇顺序固定为：
  1. W3C WoT 到底解决什么问题；
  2. Thing Description 是语义契约，而不是设备配置文件；
  3. 同一个 Thing 如何通过不同协议交互。
- 实现仓库 `https://github.com/yushun1990/clinkz-wot` 仍是源码、测试和架构事实的权威来源。
- 文章必须绑定一个明确的主项目 commit，并区分实现、已接受设计、计划和探索内容。
- 写作同时服务于项目传播和作者真正掌握 ClinkZ-WoT，不用流畅文案掩盖理解缺口。

## Why the Previous Framing Was Rejected

原稿从“很多物联网平台从 MQTT Topic 开始设计”切入，虽然其中关于 Topic、业务语义和协议边界的部分可以成立，但它存在两个根本问题：

1. 它把部分 MQTT 项目的工程路径扩大成了整个物联网平台领域的普遍问题；
2. 它让读者误以为 WoT 是为了修正或替代 MQTT，而不是处理位于具体协议之上的 Thing 描述与交互问题。

因此，第一篇改为直接回答 WoT 自己的问题空间：

```text
不同设备和服务已经能够通信
        |
        v
接口、能力和交互方式仍然彼此异构
        |
        v
应用缺少共同、机器可读的 Thing 接口模型
        |
        v
TD + Interaction Affordance + Form + Protocol Binding
```

## WOT-001 Research Baseline

### Main-project snapshot

- Repository: `https://github.com/yushun1990/clinkz-wot`
- Branch: `master`
- Commit: `6c01e07a446f51d413618474554b5eedcf5de23e`
- Inspected at: `2026-07-28`

Read during the draft:

- `AGENTS.md`
- `PROJECT_STATE.md`
- `PLAN.md`
- `README.md`
- `docs/architecture/10-primary-data-flows.md`

External standards checked:

- W3C Web of Things Architecture 1.1
- W3C Web of Things Thing Description 1.1
- W3C Web of Things Binding Templates
- W3C Web of Things Discovery

### Core argument

W3C WoT 不发明新的通信协议，也不与 MQTT、HTTP、CoAP 或 Zenoh 竞争。它解决的是更上一层的问题：如何以机器可读的方式描述物理或虚拟 Thing 的元数据和网络接口，让 Consumer 围绕 Property、Action 和 Event 表达交互意图，再通过 Form 与 Protocol Binding 将这些交互落实到具体协议消息。

第一篇的主线是：

```text
communication is possible
  -> interfaces are still heterogeneous
  -> applications need a shared interaction model
  -> TD describes the Thing interface
  -> Protocol Binding maps affordances to protocol messages
  -> WoT Runtime turns description into execution
```

### ClinkZ-WoT relevance

当前已接受的项目方向是：

```text
TD
  -> parse and validate
  -> immutable planning context
  -> logical plans
  -> binding-owned artifacts
  -> admitted immutable plan set
  -> runtime execution
```

协议中立层描述 Thing、Property、Action、Event 和 operation；Protocol Binding 拥有 HTTP URL、MQTT Topic、CoAP resource、Zenoh key expression 等协议专属知识；Servient 拥有运行时编排、生命周期和清理权威。

### Fact-state boundary

- WoT Thing、TD、Interaction Affordance 和 Protocol Binding 分工：外部标准事实。
- ClinkZ-WoT semantic-first、immutable planning context、logical plan、binding-owned artifact 和 immutable plan set：`ACCEPTED_DESIGN`。
- 主项目仍处于 v4.9 架构闭合，M1/M2 进行中：当前项目状态。
- 完整 Property Read 纵向链路、Planning、Binding SPI 和 Servient 集成尚未全部完成：`PLANNED` / partially implemented / blocked work。
- 文章中的接口、TD 和 Rust 模块流程均明确标为概念示意，不冒充当前稳定 API。

## Draft Status

新稿已经包含：

- 从不同设备和接口的异构问题开场；
- WoT 与通信协议所处层级的对照；
- “接口异构”为什么不只是“协议太多”；
- Thing 可以是物理实体或虚拟实体；
- Property、Action、Event 三类 Interaction Affordance；
- TD、Form 与 Protocol Binding 的责任；
- WoT 不是什么及其能力边界；
- 为什么只有 TD 仍然需要 WoT Runtime；
- ClinkZ-WoT 的 TD-to-plan-to-binding 方向；
- 当前项目状态和未完成边界；
- 采用 WoT 所需付出的建模、Binding 和生命周期治理成本。

## Current Writing Queue

1. WOT-001 — 作者理解校验：文章是否准确表达“WoT 解决接口与交互模型问题，而不是协议替代问题”；
2. WOT-001 — 第二轮标准术语和技术事实校验；
3. WOT-001 — 根据作者反馈调整案例、语气和篇幅；
4. WOT-001 — 发布前回填知乎信息和前后导航；
5. WOT-002 — 在 WOT-001 方向稳定后开始提纲。

## Next Safe Actions

1. 检查第一篇是否过早进入 ClinkZ-WoT 内部架构，必要时压缩 Rust 边界章节。
2. 检查 Property、Action、Event、Form 和 Protocol Binding 的措辞是否严格对应 W3C WoT 1.1。
3. 决定知乎版本保留约 4000–5000 字完整稿，还是压缩为更偏传播的版本。
4. 完成第二轮技术校验后，将状态从 `DRAFTING` 改为 `REVIEWING`。
5. 发布后回填 `published_at`、`canonical_url`、文章地图和下一篇链接。

## Open Editorial Questions

- 知乎版本是否保留 Front Matter 之外的项目 commit 基线提示。
- 第一篇是否需要单独绘制“领域能力—交互模型—Form—协议—运行时”分层图。
- 外部平台标题是否始终使用系列前缀，仓库文件内标题保持一致。
- 正文和图表最终采用哪一种内容许可证。

## Continuation Rule

新会话应依次读取：

1. `AGENTS.md`；
2. 本文件；
3. `CONTENT_PLAN.md`；
4. `EDITORIAL_GUIDE.md`；
5. `SOURCE_POLICY.md`；
6. WOT-001 新稿；
7. ClinkZ-WoT 主仓库最新状态。
