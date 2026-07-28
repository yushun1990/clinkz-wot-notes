# Editorial Guide

本文件定义 ClinkZ-WoT 技术文章的读者、语言、结构和表达方式。

## 1. 目标读者

主要读者：

- 有后端、嵌入式或物联网经验的开发者；
- 熟悉 MQTT/HTTP，但不熟悉 W3C WoT 的工程师；
- 对 Rust async、`no_std`、运行时和协议适配感兴趣的开发者；
- 正在尝试使用 AI 长期推进复杂项目的人。

文章不假设读者已经理解：

- Thing Description；
- Servient；
- Protocol Binding；
- Logical Plan；
- generation-bearing lifecycle；
- ClinkZ-WoT 的内部 work package 编号。

第一次出现的项目术语必须解释。

## 2. 写作定位

文章首先是一篇能独立成立的技术文章，其次才是项目宣传。

推荐顺序：

1. 提出一个普遍存在的工程问题；
2. 展示直觉方案为什么会失败；
3. 比较主要替代方案；
4. 介绍 ClinkZ-WoT 的选择；
5. 说明该选择如何落到 Rust 和运行时边界；
6. 坦率说明代价、限制和未完成部分。

不要从“ClinkZ-WoT 是一个……”开始连续介绍内部模块，除非文章本身就是项目
总览。

## 3. 标题规范

推荐格式：

```text
ClinkZ-WoT 设计笔记 01｜为什么物联网平台不应该从 MQTT Topic 开始设计
```

标题应突出问题或矛盾，不只罗列名词。

好的标题：

- 为什么 Protocol Binding 不能直接调用 Handler
- 删除一个 Thing，为什么不等于资源已经释放
- build.rs 能不能实现 Rust 插件系统
- Subscription 为什么不应该 Clone

较弱的标题：

- ClinkZ-WoT Protocol Binding SPI 介绍
- 关于 Plan 的一些思考
- Rust 异步架构最佳实践

## 4. 推荐正文结构

### 开场：现实问题

用一个具体场景开始，例如：

- 一个 TD 同时包含 HTTP、HTTPS 和 Zenoh form；
- Binding 收到请求后是否可以直接调用 Handler；
- 旧 generation 的异步响应晚于新 generation 返回；
- 一个 clone 出来的 Subscription 到底是广播还是竞争消费。

### 直觉方案

公平地写出直觉方案的好处。不要为了突出当前架构而把其他方案写得愚蠢。

### 失败场景

至少给出一个可执行的失败时序、资源泄漏路径或权责冲突。

### 方案比较

比较 2–4 个真正不同的选择，指出：

- 谁拥有状态；
- 谁拥有生命周期；
- 错误在哪里暴露；
- 是否有隐藏任务；
- 是否可取消和清理；
- 在 host 与 constrained profile 中是否成立。

### ClinkZ-WoT 的选择

说明当前选择属于：

- 已实现；
- 已接受设计；
- 计划；
- 探索。

同时写明放弃了什么能力。

### Rust 映射

落到至少一种具体形式：

- trait 边界；
- owned value 或 lease；
- state machine；
- module ownership；
- immutable plan；
- generation；
- bounded queue；
- compile fixture；
- host/manual progress model。

### 当前限制

主动说明：

- 尚未实现的部分；
- 仍开放的 gate；
- 可能变化的 API；
- 文章基于哪个 commit。

### 总结

给出 3 个以内可复述的结论。

## 5. 语言风格

使用：

- 直接、准确的中文；
- 短段落；
- 清楚的因果关系；
- 必要的英文术语；
- 具体名词代替泛化代词；
- 状态机、时序图和对照表。

避免：

- 过度口语化；
- 大量感叹号；
- 过长的项目背景；
- 每段都使用列表；
- 为了显得深刻而制造晦涩句子；
- “优雅、完美、先进、业界领先”等无法验证的评价；
- 把复杂度简单归因于 Rust。

## 6. 术语规范

统一使用：

| 中文/英文 | 说明 |
|---|---|
| ClinkZ-WoT | 项目名 |
| Web of Things / WoT | 首次出现给出全称 |
| Thing Description / TD | 不翻译成“物描述” |
| Servient | 保留英文，首次解释为 WoT 运行时主体 |
| Protocol Binding / Binding | 协议绑定；后文可简称 Binding |
| Logical Plan | 逻辑计划 |
| Binding Artifact | Binding 拥有的协议产物 |
| Compiled Plan Set | 编译计划集合 |
| ProducedThing | 应用向外提供的 Thing |
| ConsumedThing | 应用消费的远程 Thing |
| Handler | 处理器；涉及 API 时保留英文 |
| generation | 代际/世代；正文优先保留英文并解释 |
| `no_std + alloc` | 保持代码格式 |

不要在同一篇文章中交替使用 Servient、Servant、服务体等不同名称。

## 7. 图表规范

图必须回答一个问题，而不是只起装饰作用。

优先使用：

- 数据流图；
- 时序图；
- 生命周期状态机；
- 模块权责图；
- 方案对照表；
- 失败路径图。

每张图应有：

- 图题；
- 对应架构基线；
- 正文解释；
- 可读的替代文本。

避免在技术图中加入尚未实现的组件，除非明确标记为 planned。

## 8. 代码规范

- 代码片段应足以解释边界，不追求可独立编译；
- 项目真实 API 尚未稳定时，可以使用标明为“概念示意”的伪代码；
- 真实源码片段必须对应文章记录的 commit；
- 不要用虚构的 Rust trait 冒充项目当前 API；
- 单个片段尽量控制在 30 行以内。

## 9. 文章长度

建议：

- 基础概念：2500–4000 字；
- 架构深挖：4000–7000 字；
- Rust 机制：3500–6000 字；
- AI 工程复盘：3000–5000 字。

为了讲清楚可以超出范围，不要为了长度拆成缺乏独立价值的多篇短文。

## 10. 系列连续性

每篇文章末尾包含：

```markdown
## 延伸阅读

- 上一篇：
- 下一篇：
- 项目资料：
- 相关源码：
```

前置文章只链接真正需要的知识，不要形成过长的阅读依赖链。
