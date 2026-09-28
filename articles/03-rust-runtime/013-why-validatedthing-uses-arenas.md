---
id: "RUST-013"
title: "ClinkZ-WoT 设计笔记｜为什么已验证的 TD 不再需要 BTreeMap"
subtitle: "三个 Arena、一次封存，以及可以核算的 Rust 内存所有权"
series: "Rust 运行时机制"
series_order: 13
status: "DRAFTING"
author: "yushun1990"
created: "2026-09-28"
updated: "2026-09-28"
summary: "Thing 的树形对象适合编辑，却很难给出与构造历史无关的精确内存承诺。本文解释 ClinkZ-WoT 为什么为已验证 TD 选择 Node、Edge、Byte 三个 Arena，以及构建峰值、语义等价和资源准入如何约束这一方案。"

clinkz_wot:
  repository: "https://github.com/yushun1990/clinkz-wot"
  branch: "master"
  commit: "3c37240e4b94edc335a46d33a2a81003e4e0fc01"
  inspected_at: "2026-09-28"

publication:
  platform: "zhihu"
  published_at: null
  canonical_url: null

related:
  previous: null
  next: null
  articles: []
  docs:
    - "docs/work-packages/WP-100-consumer-validated-thing-admission.md"
    - "docs/work-packages/index.toml"
  source:
    - "td/src/thing.rs"
  tests:
    - "tools/architecture-fixtures/validated-thing-arena-layout/src/lib.rs"
    - "td/tests/support/normalized_snapshot_probe.rs"
---

# ClinkZ-WoT 设计笔记｜为什么已验证的 TD 不再需要 BTreeMap

> **事实基线**：ClinkZ-WoT `3c37240e4b94edc335a46d33a2a81003e4e0fc01`（2026-09-28）。
>
> **当前状态**：三 Arena 是 WP-100 已接受、但尚未获得生产实现准入的设计。仓库中有非生产的内存布局原型、字段存储测试和局部共享语义原型；本文不把它们描述成已经上线的 `ValidatedThing`。

## 一句话摘要

编辑 TD 时，`BTreeMap` 很好用；运行时长期保存一份已验证 TD 时，却更需要稳定的语义、可观测的分配和可核算的内存峰值。ClinkZ-WoT 的选择是：保留用于编辑和交换的 `Thing`，再将验证通过的内容归一化、封存到 Node、Edge、Byte 三个 Arena 中。

## 一、一个看似简单的问题：这份 TD 到底占多少内存？

假设有一台设备，提供温度、湿度和压力三个 Property。开发者自然会想用 Rust 表达它们：

~~~rust
// 来自当前 Thing 数据模型的字段类型，仅摘取与本例有关的部分。
pub properties: Option<BTreeMap<String, PropertyAffordance>>,
~~~

再往下看，Property 有 Form、Schema、URI 变量；Schema 还可以递归嵌套；扩展字段保存任意 JSON 值。每一层都可能进一步包含 `Vec`、`String`、映射和嵌套对象。

这不是错误设计。`BTreeMap` 支持按键查询和动态修改，Rust 的类型也让编辑器、配置生成器和普通 TD 用户更容易使用。问题出在另一种需求：

> 运行时准备接受这份远程 TD，却必须先承诺：我还要申请多少内存？其中最大的一次连续申请是多少？构建、失败回滚和最终释放会经过什么内存峰值？

简单地累加字符串的 `len()`，或者测量 `size_of::<Thing>()`，都不能回答这些问题。后者主要衡量顶层值的静态布局；前者遗漏容器容量、节点与构建过程中的额外分配。同样的语义内容，也可能因为构造时的容量和修改历史不同而具有不同的内存使用情况。

当然，可以继续为每一种标准库容器和依赖内部实现编写核算公式，但这会使资源契约绑在项目不能稳定控制的内存布局上。ClinkZ-WoT 先前的 WP-100 候选实现就曾因私有 `liballoc` 布局核算依据无法成立而撤回准入；这正是后来选择项目自有归一化存储的重要背景，而不是一次单纯的性能优化。

## 二、直觉方案：保留整棵 Thing，外面包一层 ValidatedThing

第一种想法非常自然：验证成功后，直接将 `Thing` 放进 `ValidatedThing`。这样不用重新组织数据，调用者还能沿用原本的 Rust 字段与容器。

它适合不要求精确内存契约的应用。但在受限资源准入中，验证“语义合法”并不等于证明“额外内存可控”。仅仅用一个新类型包装旧容器，不能消除内部容量、分配次数和递归释放路径的不确定性。

另一个看似简单的方案，是把 `Thing` 序列化成 JSON 文本或二进制 Blob。这虽然减少了长期持有的分配种类，却又带来新问题：Planner 每次读 Property、Form 或安全声明，都需要重解析、维护第二份结构，或者把解析规则分散到消费端。更重要的是，当前 WP-100 明确要求保留 **typed Thing 的字段级语义**：不能因为一个 Basic-valid 的 Rust 值无法通过现有序列化路径，就在兼容入口拒绝它。

因此，真正要替换的不是 BTreeMap 的查找算法，而是 **已验证对象的所有权与内存布局**。

## 三、为什么叫 Arena？

Arena 是一片由同一个所有者管理的内存区域。它可以集中存放一类对象，通常让对象通过索引、偏移或句柄相互关联，而不是要求每个嵌套对象各自拥有一块堆内存。

Arena 不是特定的 Rust 标准容器，也不等于只能顺序追加的 bump allocator。一个系统可以根据用途分别设计不同的 Arena：可增长的构建数组、固定大小的对象池，或者封存后不可变的连续数组。

ClinkZ-WoT 的特殊之处在于：它不是使用一个万能的 Arena，而是 **按照数据的职责拆成三个封存 Arena**。

~~~text
                  ValidatedThing（已接受的目标设计）
                                |
                +---------------+---------------+
                |               |               |
          Node Arena       Edge Arena      Byte Arena
          类型与标量        关系与区间       可变长内容
                |               |               |
              Node ID <---- Edge.target      字节偏移/长度
~~~

*图 1：三 Arena 的职责示意，并非当前生产内存布局图。*

### Node Arena：保存“这里是什么”

Node 记录 Thing、Property、Form、SecurityScheme、DataSchema、扩展 JSON 值等节点的类型，以及固定大小的标量或 Arena 区间。它不直接拥有新的 `String`、`Vec` 或 `BTreeMap`。

### Edge Arena：保存“它们怎样关联”

Edge 保存集合中的成员、映射的键值关联、目标 Node 索引和有意义的原始序列位置。

例如，`properties` 在语义上是一个从名称到 Property 的映射。它不一定需要一棵常驻 B 树，只需要属于该 Map 的一段 Edge 区间，并确保名称与目标节点保持正确关联。

映射按语义键排序；Form、Context、JSON Array 等有顺序意义的数据，仍保留原始顺序和 Form 索引。**映射的插入历史不应成为 TD 语义，而有意义的序列顺序不能被排序抹掉。**

### Byte Arena：保存“真正的内容”

属性名称、标题、URI，以及需要无损保留的 JSON Number 文本等可变长度数据，集中保存在字节区域。Node 和 Edge 只记录用于定位这些内容的标量或区间。

以下内容是帮助理解的概念示意，不是仓库中的真实结构定义：

~~~rust
// 概念示意：具体 Rust 类型及字段请以正式设计为准。
struct Node {
    kind: NodeKind,
    edges: Range,
}

struct Edge {
    key_bytes: Option<Range>,
    target_node: u32,
    original_index: u32,
}
~~~

这里的核心变化是：**每个节点不再需要自己负责一串新的堆对象，而是通过整数关联同一个拥有者持有的 Arena**。

## 四、那 BTreeMap 是否可以彻底删除？

答案分两个阶段。

**编辑与交换阶段：仍然需要。** 当前公开的 `Thing` 继续使用 Rust 的结构化数据模型。开发者可以方便地增删 Property，组合 Form，或者从第三方获得 typed TD。WP-100 不打算破坏这一接口来强制所有使用者手动维护 Arena 索引。

**已验证后的封存阶段：不再保留。** 设计规定，成功构建的 `ValidatedThing` 不拥有原输入 `Thing`、`BTreeMap`、调用者的容器容量或输入借用，而只拥有归一化的不可变快照。

如果某个 Map 的 Edge 区间已按键排序，未来可以在这段连续区间里进行二分查找，再通过 Node 索引读取 Property。二分查找的比较次数是 O(log n)，但字符串比较还有自身成本。**这是排序表示允许采用的实现策略，不应被误写成已经完成并经过性能验证的生产索引算法。**

| 问题 | 编辑阶段的 BTreeMap | 封存后的排序 Arena |
|---|---|---|
| 主要目标 | 易于构造、查找和修改 | 稳定存储、只读访问和资源核算 |
| 修改映射 | 支持动态增删 | 设计为不可变；更新通常需要构建新快照 |
| 关系表示 | 容器自身维护树结构 | 一段排序 Edge 加 Node 索引 |
| 内存核算 | 需考虑容器及其嵌套分配 | 由项目控制的实际 Arena 申请构成 |
| 代价 | 内部布局和容量较复杂 | 构建、排序、封存与索引访问更复杂 |

Arena 本身并不会自动带来性能收益。连续布局往往有利于局部性，但究竟快多少，必须用实际数据和目标硬件测量；本设计首先追求的是可核算的所有权。

## 五、只有三个 Arena，为何还需要四个临时 Arena？

三个 Arena 描述的是**成功完成后长期保留什么**，而不是整个构建过程只会出现三次申请。

已接受设计明确区分两个阶段：

~~~text
输入：调用者已有的 Thing，或借用的 JSON 字节
         |
         v
     验证 / 归一化
         |
         v
  构建期四个临时 Arena
  - 可增长 Node
  - 可增长 Edge
  - 可增长 Byte
  - 遍历帧（处理嵌套结构）
         |
         v
    校验资源并封存
         |
         v
  三个精确长度的不可变 Arena
~~~

*图 2：WP-100 规定的构建与封存生命周期。输入所有者的内存不计作新增的项目自有 Arena。*

为什么还需要单独的遍历帧？因为嵌套 TD、DataSchema 和 JSON 扩展可能很深。生产设计不能把无界的递归调用栈当作免费资源；构建中的遍历状态也需要被管理和计费。现有字段存储测试原型仍有递归遍历和预留空间，因此尚不能声称已经解决完整有界遍历问题。

封存时，构建数组可能留有未使用的 capacity。若直接保留，就会把历史扩容行为带入最终内存占用。因此封存要生成按实际内容精确申请的 Node、Edge、Byte 区域；空区域不产生申请。

但还有一个容易被忽略的峰值：**生成新 Arena 时，旧的构建 Arena 尚未释放。** 增长时也是一样，旧数组与新数组可能短暂共存。如果只计算最终三个 Arena 的大小，就会漏掉真正容易触发资源上限的瞬间。

## 六、资源证明不是“我估计能放得下”

WP-100 为这些内存申请规定了实际可检查的责任：使用项目控制的 `Layout`，检查整数溢出，在分配前通过 Foundation 的 `AdmissionLedger` 做与真实申请一一对应的预留，并记录源对象、临时内存、总峰值与最大单次连续申请。

以下只表达检查关系，变量并非真实公开 API：

~~~text
r = 本次实际申请的 Layout 大小
S = 当前已计入的源对象内存
T = 当前已计入的临时内存

单次连续申请：           r <= contiguous_limit
这次申请后的总峰值： S + T + r <= peak_limit
临时或源对象限制：    对应账户 + r <= 对应 limit
~~~

这些式子里的加法也必须经过溢出检查。申请失败、预算拒绝、数组增长或封存失败，都必须回到明确的所有权状态。

更关键的是：账本里记录的最大连续申请，应该是**实际单次申请中的最大值**，不是把三段 Arena 或所有临时占用加起来冒充一次大申请。

Arena 只给出了容易管理的存储形状；真正的资源保证，还需要正确的计费、预算执行、取消和有界清理共同成立。

此外，兼容入口从 `&Thing` 读取时，只承诺从入口开始由项目新增的资源，不可能倒过来约束调用者在此之前就已分配的对象树。严格 JSON 入口计划提供更完整的引擎自有处理资源界限，但调用者持有的输入字节仍归调用者负责。

## 七、把数据压平，不等于把 TD 语义压没

只保存 Node 和字节还不够。两个语义相同的 TD，可能以不同顺序插入 Map；反过来，同一个 Form 数组改变顺序，却可能真的改变调用者的选择结果。

因此，归一化必须同时满足：

- 对 Map：保留准确的键值对应关系，忽略非语义性的插入历史；
- 对 Form、Context 和 JSON 数组：保留原始顺序和必要的原始索引；
- 对数字和扩展数据：保留规定的 typed 内容与无损 Number 文本，不用一次 `f64` 转换冒充原始数值；
- 对 URI、默认操作与安全继承：由 TD 自身的同一套语义规则解释，不能让 `Thing` 与 Snapshot 各自维护一套容易分叉的逻辑。

这也是三 Arena 设计不等于“先序列化再读回来”的原因。它保留的是 typed TD 的字段语义，而不是某次序列化恰好能够表达的子集。

未来 Planning 只通过借用式 `ValidatedThingView` 获取已经验证过的含义，而不是拿到裸 Arena 偏移或重新解析 Blob。原始和已解析的 Form URI 可以按正式设计一起存入 Byte Arena，但如何维持同源语义、准确预算以及零分配查询，还需要完整证明。

## 八、已经实现了什么，还有什么没有证明？

截至本文基线，三 Arena **不是已经落地的生产 `ValidatedThing`**。

已有的非生产证据包括：三保留 / 四临时 Arena 分配与封存原型；对多个 TD 字段、嵌套 Schema、Form、SecurityScheme 和顺序/键值关系的存储测试；默认操作与安全继承的局部共享语义原型。独立的 RFC3339 可暂停解码原型也已纳入相关证据。

但 WP-100 仍是 `planned / candidate / current`，尚未重新准入。完整 Basic 验证等价、共享 URI 解析、严格输入的全流程预算、完整有界遍历与回滚、外部 Planning View，以及完整目标平台证据，都不能因为这些局部原型成功就算作已经完成。

这里的技术代价同样明确：我们放弃了“验证以后直接保留整棵可修改的 Thing”，换来可控存储；同时把复杂度集中到了归一化、排序、资源核算、语义核与准入证明之中。

## 总结

第一，Arena 与 BTreeMap 并不互斥：前者是内存所有权和布局策略，后者主要是适合动态编辑的映射容器。ClinkZ-WoT 决定让它们分别服务于不同生命周期。

第二，三个 Arena 的意义不只是减少长期分配，而是让完成态的内存布局摆脱调用者的容量和构造历史；四个临时 Arena 和实际峰值核算负责补上构建过程的账。

第三，**归一化后的布局可以不同，但 TD 的解释不能改变。** 如果语义规则、预算与失败清理无法同时成立，再漂亮的连续内存也不能构成受限设备上的可靠基础设施。

## 延伸阅读

- 上一篇：暂无（本文为独立专题草稿）。
- 下一篇：待定，可进一步讨论共享语义核与借用式 Planning View。
- 项目资料：[WP-100 ValidatedThing 准入设计](https://github.com/yushun1990/clinkz-wot/blob/3c37240e4b94edc335a46d33a2a81003e4e0fc01/docs/work-packages/WP-100-consumer-validated-thing-admission.md)。
- 相关源码：[Thing 数据模型](https://github.com/yushun1990/clinkz-wot/blob/3c37240e4b94edc335a46d33a2a81003e4e0fc01/td/src/thing.rs)；
  [Arena 原型](https://github.com/yushun1990/clinkz-wot/blob/3c37240e4b94edc335a46d33a2a81003e4e0fc01/tools/architecture-fixtures/validated-thing-arena-layout/src/lib.rs)；
  [Typed snapshot 测试](https://github.com/yushun1990/clinkz-wot/blob/3c37240e4b94edc335a46d33a2a81003e4e0fc01/td/tests/support/normalized_snapshot_probe.rs)。

## 项目资料

- 主项目基线：`3c37240e4b94edc335a46d33a2a81003e4e0fc01`
- 正式架构与准入：`docs/work-packages/WP-100-consumer-validated-thing-admission.md`
- 准入状态：`docs/work-packages/index.toml`
- 当前编辑模型：`td/src/thing.rs`
- 非生产布局原型：`tools/architecture-fixtures/validated-thing-arena-layout/src/lib.rs`
- 字段存储与局部语义原型：`td/tests/support/normalized_snapshot_probe.rs`、`td/tests/support/semantic_kernel_probe.rs`

> **编辑状态**：DRAFTING。技术内容基于上述仓库基线，尚待作者审阅，尚未发布为知乎文章。