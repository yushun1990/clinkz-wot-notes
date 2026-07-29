# Writing Project State

Last updated: 2026-07-29

## Current Objective

完成第二篇文章的作者理解校验与技术校验：

> WOT-002｜重新理解 WoT | TD、网关与 Directory 在真实系统中如何协作

同时保留 WOT-001 的发布前校验任务，不把开始第二篇误写成第一篇已经完成。

## Repository Status

- GitHub repository: `https://github.com/yushun1990/clinkz-wot-notes`
- Default branch: `main`
- WOT-001 当前状态：`DRAFTING`。
- WOT-002 当前状态：`DRAFTING`。
- WOT-002 新稿位于：
  `articles/01-wot-foundations/002-td-gateway-directory-in-practice.md`
- 原稿 `002-thing-description-is-semantic-contract.md` 已删除。
- 专栏导读和 README 已更新为新的 1.2 标题、链接与叙事。
- 两篇文章均尚未登记知乎 canonical URL。

## Why WOT-002 Was Rewritten

第一版 WOT-002 试图通过“不要把线程数、连接池、缓存和运行时句柄放进 TD”说明 TD 与运行时配置的边界。

这个反例虽然在抽象上可以成立，但不符合文章需要面对的真实读者和工作场景：

- 实际编写业务模型的人往往是工艺、设备或业务技术人员；
- 他们关心设备能力、工艺含义、数据单位、操作和告警；
- 他们通常不会尝试把 CPU、线程、连接池或 Rust 对象写进 TD；
- 因此原稿先制造了一个很少发生的错误，再花大量篇幅反驳它，论证显得脱离实际。

新的 WOT-002 改为回答真正影响 WoT 落地的问题：

```text
业务和工艺知识
  -> Thing Model / 能力模板
  -> 现场协议与点位映射
  -> 网关或平台生成具体 TD
  -> TD Server 托管或 Directory 注册
  -> Consumer 发现并获取 TD
  -> WoT Runtime 根据 Form 执行交互
```

## Stable Editorial Decisions for WOT-002

- 标题改为：
  **重新理解 WoT | TD、网关与 Directory 在真实系统中如何协作**。
- 第二篇不再以荒诞的运行时配置字段作为主要冲突。
- 文章必须解释 TD 在真实系统中的生成、托管、注册、发现、消费和更新流程。
- 业务/工艺人员负责定义 Thing 的业务能力，不要求他们理解现场协议或手写完整 TD。
- 设备与集成人员负责将寄存器、点位和私有协议映射为 Property、Action 和 Event。
- 网关可以作为 Intermediary：代理存量设备、跨协议转换、聚合多个设备、暴露虚拟 Thing、生成或托管 TD。
- 存量设备不需要原生支持 WoT；TD 可以由设备自身或外部服务托管。
- 必须区分：
  - TD Server：根据已知 URL 提供一份 TD；
  - Thing Description Directory：管理和查找一组 TD。
- Directory 保存和检索接口元数据，不等于消息总线、时序数据库或每次交互的中转节点。
- Consumer 获得 TD 后，实际调用是否经过网关，取决于 TD 中的 Form。
- Directory 是可选系统组件，构造 Servient 不依赖进程内 Directory。
- ClinkZ-WoT 只拥有 Directory/Discovery 客户端边界，不实现 Directory 存储和服务端查询。

## WOT-002 Research Baseline

### Main-project snapshot

- Repository: `https://github.com/yushun1990/clinkz-wot`
- Branch: `master`
- Commit: `f453f165c2ea775e5f0d10c36f1e419fcc1d79f3`
- Inspected at: `2026-07-29`
- Active design revision: `v5.0 bounded-core authority`

Read or rechecked:

- `PROJECT_STATE.md`
- `docs/architecture/10-primary-data-flows.md`
- `docs/work-packages/WP-500-discovery.md`
- `td/src/lib.rs`
- `td/src/thing.rs`
- `td/src/thing_model.rs`

### External standards checked

- W3C Web of Things Architecture 1.1
- W3C Web of Things Thing Description 1.1
- W3C Web of Things Discovery

Key standard facts used:

- A TD may be hosted by the Thing itself or externally for constrained and legacy devices.
- Edge gateways commonly provide local compute and bridge protocols.
- A Thing Description Server provides a TD at a known URL.
- A Thing Description Directory manages a collection of TDs and supports registration and lookup.
- A Consumer can obtain a TD through Directory, direct retrieval, static assignment or another discovery mechanism.
- Obtaining a TD and performing the interactions described by its Forms are separate steps.

### ClinkZ-WoT fact boundary

- TD/TM structures, serialization, validation and extension preservation: `IMPLEMENTED`.
- Discovery results entering the same consume path as application-supplied TDs: `ACCEPTED_DESIGN`.
- No implicit in-process Directory and client-only Directory scope: `ACCEPTED_DESIGN`.
- WP-500 Directory and Discovery Client Runtime: `PLANNED`.
- Full new Directory client API and state machines: not yet claimed as implemented.

## Draft Structure

The rewritten WOT-002 currently covers:

1. real project roles: process experts, device integrators and platform/runtime developers;
2. Thing Model as a reusable capability template;
3. concrete TD generation after device deployment;
4. legacy devices represented through gateways or external TD hosting;
5. gateway as protocol bridge, proxy, aggregator and virtual-Thing host;
6. three TD hosting patterns: device, gateway and platform/Directory;
7. TD Server versus Thing Description Directory;
8. why Directory is not a data bus or mandatory runtime hop;
9. the end-to-end path from business modeling to ConsumedThing interaction;
10. when Directory is unnecessary and when it becomes valuable;
11. TD updates and ownership;
12. ClinkZ-WoT's Directory client-only boundary.

## Current Writing Queue

1. WOT-002 — 作者确认新的实际应用主线是否符合预期；
2. WOT-002 — 检查 Thing、Intermediary、TD Server、Directory 与 Discovery 用词；
3. WOT-002 — 检查泵站组合场景是否清楚且不过度假定具体平台；
4. WOT-002 — 压缩重复内容，控制知乎版本篇幅；
5. WOT-002 — 确认后将状态推进到 `REVIEWING`；
6. WOT-001 — 完成作者理解校验、第二轮技术事实校验和篇幅压缩；
7. WOT-003 — 在前两篇边界稳定后开始提纲。

## Open Editorial Questions

- 1.2 是否需要增加正式架构图：
  `Thing Model -> Gateway -> TD Server/Directory -> Consumer`。
- Directory 的服务端职责是否只在 1.2 做必要解释，把存储、查询、多租户和授权边界留到 WOT-006 深挖。
- 是否把“谁编写 TD”单独整理成角色责任表。
- WOT-001 与 WOT-002 的泵站例子是否需要进一步统一名称和字段。

## Continuation Rule

新会话应依次读取：

1. `AGENTS.md`；
2. 本文件；
3. `CONTENT_PLAN.md`；
4. `EDITORIAL_GUIDE.md`；
5. `SOURCE_POLICY.md`；
6. WOT-001；
7. WOT-002；
8. ClinkZ-WoT 主项目最新状态。
