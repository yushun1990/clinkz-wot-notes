# Writing Project State

Last updated: 2026-07-29

## Current Objective

完成第二篇文章的作者理解校验和第二轮技术校验：

> WOT-002｜重新理解 WoT | Thing Description 是语义契约，而不是设备配置文件

同时保留 WOT-001 的发布前校验队列，不因开始第二篇而把第一篇误标为完成。

## Repository Status

- GitHub repository: `https://github.com/yushun1990/clinkz-wot-notes`
- Default branch: `main`
- 专栏导读和系列一叙事已经从真实系统的协议与接口碎片化出发。
- WOT-001 修订稿位于：
  `articles/01-wot-foundations/001-what-does-wot-solve.md`
- WOT-001 当前状态：`DRAFTING`。
- WOT-002 首稿已经创建：
  `articles/01-wot-foundations/002-thing-description-is-semantic-contract.md`
- WOT-002 当前状态：`DRAFTING`。
- 两篇文章均尚未登记知乎 canonical URL。

## Stable Editorial Decisions

- 文章仓库与实现仓库分离。
- 专栏名称为 **从零开发一个 Rust WoT 引擎**。
- 文章系列按“系列号.系列内序号”组织，第一季当前收录 12 篇文章。
- 系列一前三篇顺序固定为：
  1. W3C WoT 到底解决什么问题；
  2. Thing Description 是语义契约，而不是设备配置文件；
  3. 同一个 Thing 如何通过不同协议交互。
- 实现仓库 `https://github.com/yushun1990/clinkz-wot` 是源码、测试和架构事实的权威来源。
- 文章必须绑定明确的主项目 commit，并区分实现、已接受设计、计划和探索内容。
- 写作同时服务于项目传播和作者真正掌握 ClinkZ-WoT，不用流畅文案掩盖理解缺口。

## WOT-001 Stable Argument

第一篇的因果链保持为：

```text
真实系统不是一次建成
  -> 设备寿命、网络约束、采购和厂商生态不同
  -> 存量协议、网关、消息平台和 Web API 长期共存
  -> 应用不仅面对协议差异，还面对能力、Schema 和行为差异
  -> 统一 MQTT/HTTP 只能统一部分传输
  -> 需要共同、机器可读的 Thing 接口
  -> TD + Interaction Affordance + Form + Protocol Binding
```

第一篇不得重新退回以下叙事：

- 为 HTTP、MQTT、CoAP 或 Zenoh 强行分配一种设备；
- 用厂商 A/B/C 的虚构协议配对证明 WoT 必要性；
- 把 WoT 写成 MQTT、HTTP 或其他通信协议的替代者。

## WOT-002 Research Baseline

### Main-project snapshot

- Repository: `https://github.com/yushun1990/clinkz-wot`
- Branch: `master`
- Commit: `f453f165c2ea775e5f0d10c36f1e419fcc1d79f3`
- Inspected at: `2026-07-29`
- Active design revision: `v5.0 bounded-core authority`

The baseline commit is still the exact `master` head at the time of this draft.

Read or rechecked for WOT-002:

- `PROJECT_STATE.md`
- `docs/architecture/10-primary-data-flows.md`
- `README.md`
- `td/src/lib.rs`
- `td/src/thing.rs`
- `td/src/thing_model.rs`
- `td/src/components/form.rs`
- `td/src/components/data_schema.rs`

External standards checked:

- W3C Web of Things Thing Description 1.1
- W3C Web of Things Architecture 1.1
- W3C Web of Things Binding Templates

### WOT-002 core argument

“TD 是语义契约”是本文使用的解释框架，不是声称 W3C 使用了完全相同的原句。

核心边界为：

```text
Thing 对 Consumer 的可观察承诺
  -> Thing Description

一类 Thing 的可复用能力模板
  -> Thing Model

当前部署如何实现和治理交互
  -> Runtime / Binding configuration

一次规划或执行产生的状态
  -> Plan / Handle / Session / Registry
```

TD 可以包含通信元数据，但不能因此变成某个运行时实例的完整配置快照。

### What belongs in a TD

- Thing identity and human-readable metadata;
- Property, Action and Event interaction affordances;
- input, output and event Data Schemas;
- Forms and operation metadata;
- security mechanism requirements and references;
- links, semantic annotations and governed extensions.

### What does not belong in a TD

- live protocol sessions and connection pools;
- cache and retry implementation objects;
- actual passwords, private keys and access tokens;
- Rust handlers, closures or function pointers;
- runtime registry entries;
- Logical Plans and Binding Artifacts;
- generation, in-flight calls and subscription drivers;
- queue contents and current execution state.

A device setting can still be a Property when it is an intentional, supported Thing capability. For example, `targetPressure` may be a Property, while `mqtt_connection_pool_size` remains runtime configuration.

### Extension boundary

W3C TD Context Extension supports domain semantics, additional protocol metadata, new security schemes and additional vocabulary terms.

The article must retain this distinction:

```text
preserve an extension
  != understand its semantics
  != validate the extension contract
  != execute it
```

The ClinkZ-WoT `td` crate currently preserves extension members in structures such as `Thing`, `Form` and `DataSchema` using `ExtensionMap`, and emits them again during serialization. This is implementation fact. Execution support still requires the relevant Binding, extension processor or policy.

### Thing Description and Thing Model boundary

- TD describes a concrete Thing instance and aims to provide details needed for interaction.
- Thing Model describes a reusable class or capability template and may omit concrete communication metadata.
- ClinkZ-WoT implements `Thing` and `ThingModel` as separate types.
- `ThingModelForm.href` is optional because a concrete endpoint may be assigned during instantiation.
- The current crate stores and validates TM structure but does not fetch referenced models or automatically generate protocol-specific Forms.

### ClinkZ-WoT fact-state boundary

Implemented in the current `td` crate:

- protocol-neutral TD/TM structures;
- builders;
- serialization and deserialization;
- validation;
- default handling;
- extension-member round-trip preservation;
- `no_std + alloc` data-model support.

Accepted design but not fully implemented:

```text
TD document or ProducedThing draft
  -> parse + preserve extensions + validate
  -> immutable planning context
  -> logical plans + binding-owned artifacts
  -> admitted immutable plan set
  -> runtime interaction
```

The planned v5 Planning crate, Logical Plan, Binding Artifact and complete Property Read plan chain are not yet product implementation facts.

## WOT-002 Draft Status

The first draft currently includes:

- a realistic opening showing how one JSON document gradually becomes an ungoverned runtime configuration file;
- the meaning and limit of the “semantic contract” framing;
- Property, Action and Event responsibility;
- Data Schema versus language memory layout;
- why Form belongs in TD without containing live transport state;
- security requirements versus real credential material;
- one conceptual pump-station TD and a separate runtime YAML example;
- a table distinguishing public Thing configuration from private runtime tuning;
- TD Context Extension governance and round-trip preservation;
- Thing Description versus Thing Model;
- failure modes caused by putting runtime state into TD;
- mapping to the current ClinkZ-WoT `td` crate and accepted v5 planning boundary;
- a practical field-ownership checklist;
- explicit implementation/design/planning status notes.

## Current Writing Queue

1. WOT-002 — author understanding review: confirm the semantic-contract/runtime-configuration boundary;
2. WOT-002 — verify whether the pump-station TD example is valid enough for publication while remaining conceptual;
3. WOT-002 — second-pass W3C terminology review for Form, Security, TD Context Extension and Thing Model;
4. WOT-002 — inspect or add source/test evidence for extension round-trip behavior before changing status to `REVIEWING`;
5. WOT-002 — reduce repeated explanations for the Zhihu version;
6. WOT-001 — retain its pending author review and second technical review;
7. WOT-003 — begin only after the TD/Form boundary from WOT-002 is stable.

## Open Editorial Questions

- Should the title retain “设备配置文件”, or use the more precise “运行时配置文件” in the subtitle only?
- Should the conceptual runtime YAML remain in the Zhihu version, or be replaced by one side-by-side responsibility diagram?
- Should the article draw a four-layer diagram: `Thing Model -> TD -> Planning Context -> Runtime State`?
- Should WOT-002 include a short Rust builder example from `td/src/lib.rs`, or keep code for a later implementation-focused article?
- Should the article explain TD document version versus device firmware version in this piece or defer it?

## Next Safe Actions

1. Ask the author to challenge the current field-ownership examples rather than only review prose style.
2. Validate the conceptual TD example against the W3C TD 1.1 requirements and ClinkZ-WoT parser.
3. Locate and inspect tests for unknown extension member round-trip behavior.
4. Update `CONTENT_PLAN.md` and the column guide status/link after the draft direction is accepted.
5. Change WOT-002 from `DRAFTING` to `REVIEWING` only after author and technical review.

## Continuation Rule

A new writing session should read, in order:

1. `AGENTS.md`;
2. this file;
3. `CONTENT_PLAN.md`;
4. `EDITORIAL_GUIDE.md`;
5. `SOURCE_POLICY.md`;
6. WOT-001;
7. WOT-002;
8. the latest ClinkZ-WoT project state and relevant `td` sources.
