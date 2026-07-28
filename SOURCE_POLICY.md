# Source and Fact Policy

本文档规定文章如何使用 ClinkZ-WoT 主项目资料，并区分实现、设计、计划和
探索性内容。

## 1. 主项目地址

唯一默认主项目：

- <https://github.com/yushun1990/clinkz-wot>

每次写作都应直接读取该地址的最新内容。不要先在 GitHub 中搜索仓库名称，
也不要用同名 fork 或缓存页面代替主仓库。

## 2. 权威来源

### 实现事实

优先依据：

1. 源码；
2. 测试和编译 fixture；
3. 已登记的实现与完成证据；
4. 对应 commit。

仅有 README、计划或设计文档，不足以证明功能已经实现。

### 架构事实

优先依据：

1. 主项目 `docs/` 中的正式规范；
2. 已接受 ADR；
3. 规范索引、需求索引和架构治理所指向的 owner；
4. 当前 `PROJECT_STATE.md` 对迁移和冲突的说明。

### 路线和状态

优先依据：

- `PLAN.md`：里程碑、依赖、目标和状态；
- `PROJECT_STATE.md`：当前停止点、已验证事实、阻塞和下一安全动作；
- work package 与 evidence：具体 tranche 的准入和完成状态。

### 讨论和历史

- `workspace/` 保存问题、提案、替代方案和推理过程；
- 未达到 `MIGRATED` 的 workspace 结论不能冒充正式规范；
- 历史设计文档只能用于解释演进过程，不能默认代表当前架构。

## 3. 文章事实标签

研究阶段为每个重要结论标注以下内部标签：

| 标签 | 证据要求 | 正文推荐措辞 |
|---|---|---|
| `IMPLEMENTED` | 源码、测试、完成证据 | “当前实现已经……” |
| `ACCEPTED_DESIGN` | 正式规范或 ADR | “v1 架构规定……” |
| `PLANNED` | PLAN/work package | “项目计划在……中实现” |
| `EXPLORATORY` | workspace/讨论 | “目前仍在讨论……” |
| `HISTORICAL` | 历史文档/旧 commit | “早期方案曾经……” |
| `INFERENCE` | 多个证据支持的推导 | “由此可以推断……” |

一段话中混合多个状态时，应拆开说明。

## 4. 项目基线

每篇文章必须记录：

```yaml
clinkz_wot:
  repository: "https://github.com/yushun1990/clinkz-wot"
  branch: "master"
  commit: "<full commit sha>"
  inspected_at: "YYYY-MM-DD"
```

文章发布后，commit 不应随项目更新自动改变。它代表文章事实核验时的基线。

如果文章后来按新版本修订，应同时更新：

- `updated`；
- `clinkz_wot.commit`；
- 修订说明；
- 受影响的技术结论。

## 5. 引用方式

### 引用仓库文件

正文中优先使用稳定、可定位的表述：

> ClinkZ-WoT 的 Protocol Binding SPI 规范将 Binding 限定为协议适配、
> I/O、correlation 和 binding-local state 的拥有者。

文末关联：

```markdown
## 项目资料

- `docs/architecture/40-protocol-binding-spi-and-deployment.md`
- `docs/architecture/10-primary-data-flows.md`
- Commit: `<sha>`
```

发布到外部平台时，可使用指向特定 commit 的 GitHub permalink，避免未来
文件移动后引用失真。

### 引用源码

说明：

- crate/module；
- 类型或函数名；
- 对应 commit；
- 必要时给出短代码片段。

不要复制大段源码。文章目标是解释设计，而不是镜像仓库。

## 6. 跨仓库链接

文章仓库只解释主项目，不拥有主项目规范。

正确关系：

```text
clinkz-wot-notes article
        |
        | explains and links
        v
clinkz-wot docs / source / tests
```

禁止在主项目正式规范中写：

> 详细设计见知乎文章。

外部文章可以失效、简化或滞后，不能成为运行时行为的权威 owner。

## 7. 事实校验清单

发布前逐项确认：

- [ ] 已读取主项目最新 `AGENTS.md`、`PROJECT_STATE.md` 和 `PLAN.md`；
- [ ] 已检查文章主题对应的正式规范；
- [ ] 涉及当前实现时已检查源码和测试；
- [ ] 已记录完整 commit SHA；
- [ ] 没有把 workspace 讨论写成正式结论；
- [ ] 没有把 PLAN 中的目标写成已实现能力；
- [ ] 没有把 README 的概述当成完整契约；
- [ ] 已说明 API 或架构仍可能变化的部分；
- [ ] 推断已经明确标记为推断；
- [ ] 没有虚构性能、规模、兼容性和测试结果；
- [ ] 文章中的图与正文使用相同架构版本；
- [ ] 文末列出了主要项目资料。

## 8. 文章过时处理

项目演进导致文章部分过时时，按影响选择：

### 小修订

适用于术语、链接和局部细节变化：

- 更新正文；
- 更新基线 commit；
- 增加修订记录。

### 保留历史版本

适用于文章核心仍有演进价值，但不再代表当前设计：

- 状态改为 `ARCHIVED`；
- 文章顶部加入醒目标记；
- 链接到替代文章。

### 重写

适用于核心结论已经改变：

- 保留旧文作为历史；
- 创建新文章；
- 在两篇文章之间建立“旧设计/新设计”导航。
