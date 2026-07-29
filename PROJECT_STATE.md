# Writing Project State

Last updated: 2026-07-29

## Current Objective

完成第一篇文章的作者理解校验和第二轮技术校验，将修订稿推进到可发布状态：

> WOT-001｜重新理解 WoT | W3C WoT 到底解决什么问题？

## Repository Status

- GitHub repository: `https://github.com/yushun1990/clinkz-wot-notes`
- Default branch: `main`
- 专栏导读和第一季文章地图已经按新的 WoT 叙事更新。
- WOT-001 修订稿位于：
  `articles/01-wot-foundations/001-what-does-wot-solve.md`
- WOT-001 当前状态：`DRAFTING`。
- 尚未登记知乎 canonical URL。

## Stable Decisions

- 文章仓库与实现仓库分离。
- 专栏名称为 **从零开发一个 Rust WoT 引擎**。
- 文章系列按“系列号.系列内序号”组织，第一季当前收录 12 篇文章。
- 第一篇正式标题为：
  **重新理解 WoT | W3C WoT 到底解决什么问题？**
- 第一篇不得从 MQTT、HTTP、CoAP、Zenoh 或任何单一协议建立 WoT 的问题空间。
- MQTT、HTTP、CoAP、Modbus、OPC UA、BACnet、Zenoh 与 WoT 不在同一抽象层级；它们只在解释现实接入边界、Form 与 Protocol Binding 时作为具体协议示例出现。
- 第一篇必须先证明现实中的多协议和多接口碎片化为什么会自然存在，再引入 WoT。
- 不再使用“HTTP 接空调、MQTT 接传感器”这类为协议强行分配设备的开场，也不使用“厂商 A/B/C 分别采用三种协议”的虚构配对作为核心论据。
- 现实主线采用长期演进的系统：存量设备、现场协议、厂商网关、消息平台、Web API 与新应用在不同阶段进入同一个系统。
- 文章必须区分“统一传输协议”和“统一 Thing 接口”；全部转成 MQTT 或 HTTP 并不会自动统一能力、数据 Schema、操作语义和错误模型。
- 泵站场景只作为基于公开标准与 W3C use cases 的组合说明，不声称对应某个具体生产项目。
- 系列一前三篇顺序固定为：
  1. W3C WoT 到底解决什么问题；
  2. Thing Description 是语义契约，而不是设备配置文件；
  3. 同一个 Thing 如何通过不同协议交互。
- 实现仓库 `https://github.com/yushun1990/clinkz-wot` 仍是源码、测试和架构事实的权威来源。
- 文章必须绑定一个明确的主项目 commit，并区分实现、已接受设计、计划和探索内容。
- 写作同时服务于项目传播和作者真正掌握 ClinkZ-WoT，不用流畅文案掩盖理解缺口。

## Why the Framing Changed

最初的 WOT-001 草稿虽然没有再把 WoT 写成 MQTT 的替代品，但仍从以下虚构配对开始：

```text
HTTP 空调
MQTT 传感器
CoAP 或 Zenoh 边缘控制器
```

后文又使用“厂商 A 走 HTTP、厂商 B 走 MQTT、厂商 C 走 Zenoh”的水泵示例。

这种写法先假设协议碎片化成立，再用人为分配的设备和协议解释 WoT，缺少现实因果，也容易让读者追问“为什么空调一定使用 HTTP，传感器一定使用 MQTT”。

修订后的因果链为：

```text
真实系统不是一次建成
  -> 设备寿命、网络约束、采购和厂商生态不同
  -> 存量协议、网关、消息平台和 Web API 长期共存
  -> 应用不仅面对协议差异，还面对能力、Schema 和行为差异
  -> 统一 MQTT/HTTP 只能统一部分传输
  -> 需要共同、机器可读的 Thing 接口
  -> TD + Interaction Affordance + Form + Protocol Binding
```

## WOT-001 Research Baseline

### Main-project snapshot

- Repository: `https://github.com/yushun1990/clinkz-wot`
- Branch: `master`
- Commit: `f453f165c2ea775e5f0d10c36f1e419fcc1d79f3`
- Inspected at: `2026-07-29`
- Active design revision: `v5.0 bounded-core authority`

Read or rechecked during this revision:

- `AGENTS.md`
- `PROJECT_STATE.md`
- `docs/architecture/10-primary-data-flows.md`
- the current WOT-001 draft

Current implementation boundary recorded in the article:

- WP-100 includes completed Foundation and Core slices.
- M1 and M2 remain in progress.
- M3 Planning and Compilation Pipeline remains open.
- The planned `clinkz-wot-planning` crate and the v5 Logical Plan, Binding Artifact and Binding Compiler product types are not implemented.
- Existing Protocol Binding paths still reflect the legacy direct execution boundary.

### External standards checked

- W3C Web of Things Architecture 1.1
- W3C Web of Things Thing Description 1.1
- W3C Web of Things Binding Templates
- W3C Web of Things Discovery
- W3C Web of Things: Use Cases and Requirements, 2026-02-05

The key external evidence is:

- W3C WoT Architecture states that IoT uses multiple protocols because no single protocol is appropriate in all contexts.
- The 2026 use-case document explicitly includes cross-protocol Industry 4.0 interaction, brownfield and constrained devices, multi-vendor integration, and protocol abstraction from business applications.

## Core Argument

W3C WoT 不发明新的通信协议，也不与 MQTT、HTTP、CoAP、Modbus、OPC UA、BACnet 或 Zenoh 竞争。

它处理的是更上一层的问题：

```text
communication is possible
  -> real systems retain heterogeneous interfaces
  -> unified transport does not create unified semantics
  -> applications need a shared, machine-readable Thing interface
  -> TD describes capabilities and network-facing forms
  -> Protocol Binding maps interactions to real protocol behavior
  -> WoT Runtime turns description into governed execution
```

## Draft Status

2026-07-29 修订已经完成以下调整：

- 删除“HTTP 空调、MQTT 传感器、CoAP/Zenoh 控制器”的开场；
- 删除厂商 A/B/C 按协议分配水泵接口的核心示例；
- 从系统长期演进、设备寿命、通信约束和厂商生态解释协议碎片化；
- 明确多协议共存不必然是架构失败；
- 增加“统一为 MQTT 或 HTTP 为什么仍然不够”的直觉方案与失败边界；
- 使用同一条泵站能力主线解释 Property、Action、Event、Form 与 Protocol Binding；
- 补充存量设备可以由网关或代理提供 TD，不要求设备原生支持 TD；
- 明确 Thing 可以是聚合多个设备和服务的虚拟对象；
- 更新 ClinkZ-WoT 基线到 commit
  `f453f165c2ea775e5f0d10c36f1e419fcc1d79f3`；
- 将项目状态从旧的 v4.9 描述更新为 active v5.0 authority；
- 明确 Planning、Binding Compiler/Artifact 和完整 Property Read 计划链尚未进入产品实现；
- 将 W3C 2026 use-case 文档加入外部资料和事实校验记录。

## Current Writing Queue

1. WOT-001 — 作者确认新的现实问题主线是否符合预期；
2. WOT-001 — 第二轮标准术语和技术事实校验；
3. WOT-001 — 检查知乎版本篇幅，压缩重复解释；
4. WOT-001 — 决定是否绘制“真实系统碎片化 → TD/Binding”数据流图；
5. WOT-001 — 校验完成后将状态从 `DRAFTING` 改为 `REVIEWING`；
6. WOT-002 — 在 WOT-001 方向稳定后开始提纲。

## Next Safe Actions

1. 检查 Property、Action、Event、Form、Protocol Binding 和 brownfield 的措辞是否严格对应 W3C WoT 1.1。
2. 检查泵站组合场景是否足够具体，但不会被误读为某个真实项目的案例报告。
3. 压缩 ClinkZ-WoT 内部架构部分，避免第一篇过早进入 v5 细节。
4. 评估知乎版本保留约 4000–5000 字完整稿，还是进一步压缩。
5. 完成技术校验后，更新状态和发布检查项。

## Open Editorial Questions

- 第一篇是否需要单独绘制：
  `存量设备/厂商平台 -> 网关与接口 -> TD -> WoT Consumer`。
- 知乎版本是否保留 Front Matter 之外的项目 commit 基线提示。
- 外部平台标题是否始终使用系列前缀，仓库文件内标题保持一致。
- 正文和图表最终采用哪一种内容许可证。

## Continuation Rule

新会话应依次读取：

1. `AGENTS.md`；
2. 本文件；
3. `CONTENT_PLAN.md`；
4. `EDITORIAL_GUIDE.md`；
5. `SOURCE_POLICY.md`；
6. WOT-001 修订稿；
7. ClinkZ-WoT 主仓库最新状态。
