# Writing Project State

Last updated: 2026-07-28

## Current Objective

完成第一篇文章的作者理解校验和第二轮技术校验，将初稿推进到可发布状态：

> WOT-001｜重新理解 WoT | 为什么物联网平台不应该从 MQTT Topic 开始设计

## Repository Status

- GitHub repository: `https://github.com/yushun1990/clinkz-wot-notes`
- Default branch: `main`
- 专栏导读和第一季文章地图已经建立。
- WOT-001 初稿已经写入：
  `articles/01-wot-foundations/001-why-not-start-from-mqtt-topic.md`
- WOT-001 当前状态：`DRAFTING`。
- 尚未登记知乎 canonical URL。

## Stable Decisions

- 文章仓库与实现仓库分离。
- 专栏名称为 **从零开发一个 Rust WoT 引擎**。
- 文章系列按“系列号.系列内序号”组织，第一季当前收录 12 篇文章。
- 第一篇正式标题为：
  **重新理解 WoT | 为什么物联网平台不应该从 MQTT Topic 开始设计**。
- 实现仓库 `https://github.com/yushun1990/clinkz-wot` 仍是源码、测试和架构事实的权威来源。
- 文章必须绑定一个明确的主项目 commit，并区分实现、已接受设计、计划和探索内容。
- 写作同时服务于项目传播和作者真正掌握 ClinkZ-WoT，不用流畅文案掩盖理解缺口。

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
- W3C WoT Binding Registry
- OASIS MQTT Version 5.0

### Core argument

MQTT Topic 可以承载平台定义的业务约定，但 MQTT 标准主要把 Topic Name 和
Topic Filter 用于消息标识、路由和订阅匹配。Topic 本身不能完整表达 Property、
Action、Event、Data Schema、安全、操作方向和跨协议等价关系。

文章不主张抛弃 MQTT，也不否认 Topic-first 在单协议、固定场景中的效率。文章的
判断是：长期演进的平台不应把 Topic 字符串结构作为业务能力的唯一权威模型。

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

协议中立层描述 Thing、Property、Action、Event 和 operation；Protocol Binding
拥有 MQTT Topic、HTTP URL、Zenoh key expression 等协议专属知识。

### Fact-state boundary

- WoT Interaction Affordances、TD 和 Protocol Binding 分工：外部标准事实。
- ClinkZ-WoT semantic-first、immutable planning context、logical plan、
  binding-owned artifact 和 immutable plan set：`ACCEPTED_DESIGN`。
- 主项目仍处于 v4.9 架构闭合，M1/M2 进行中：当前项目状态。
- 完整 Property Read 纵向链路、规划层、Binding SPI 和 Servient 集成尚未全部完成：
  `PLANNED` / blocked work。
- 文章中的 Rust 结构和 TD 片段均明确标为概念示意，不冒充当前稳定 API 或完整生产 TD。

## Draft Status

初稿已经包含：

- 具体 MQTT Topic 树开场；
- Topic-first 的合理性；
- Topic 结构渗透规则、权限、存储和 API 的失败路径；
- MQTT、HTTP、Zenoh 多协议重复建模案例；
- Property、Action、Event 和 Form 的语义分层；
- 三种设计方式对照；
- ClinkZ-WoT 的 TD-to-plan-to-binding 方向；
- 当前项目状态和未完成边界；
- 已有 MQTT 平台的渐进迁移建议。

## Current Writing Queue

1. WOT-001 — 作者理解校验和第二轮技术事实校验；
2. WOT-001 — 根据作者反馈调整案例、语气和篇幅；
3. WOT-001 — 发布前回填外部资料链接、知乎信息和前后导航；
4. WOT-002 — 仅在 WOT-001 方向稳定后开始提纲。

## Next Safe Actions

1. 让作者用自己的话确认四个问题：Topic-first 的问题、适用边界、TD-first 的收益和代价。
2. 检查文章中的简化 TD 是否需要换成更短的语义图，避免读者误认为是完整 MQTT Binding 示例。
3. 决定是否在知乎版本保留完整“Rust 边界”章节，或将其压缩为下一篇的引子。
4. 完成第二轮技术校验后，将状态从 `DRAFTING` 改为 `REVIEWING`。
5. 发布后回填 `published_at`、`canonical_url`、文章地图和下一篇链接。

## Open Editorial Questions

- 知乎版本是否保留 Front Matter 之外的项目 commit 基线提示。
- 第一篇是保持当前约 5000 字的完整版本，还是压缩为更偏传播的 3000–4000 字版本。
- 外部平台标题是否始终使用系列前缀，仓库文件内标题保持一致。
- 正文和图表最终采用哪一种内容许可证。

## Continuation Rule

新会话应依次读取：

1. `AGENTS.md`；
2. 本文件；
3. `CONTENT_PLAN.md`；
4. `EDITORIAL_GUIDE.md`；
5. `SOURCE_POLICY.md`；
6. WOT-001 初稿；
7. ClinkZ-WoT 主仓库最新状态。
