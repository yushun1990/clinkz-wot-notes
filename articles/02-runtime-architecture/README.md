# ClinkZ-WoT 运行时架构

本系列解释 ClinkZ-WoT 如何把 Thing Description 转换为可执行、协议中立、
生命周期明确的运行时行为。

## 前置阅读

建议先阅读：

- WOT-002｜W3C WoT 不是一种新协议
- WOT-003｜Thing Description 是语义契约
- WOT-004｜一个 Thing 有多个 Form 时如何选择

## 主要主线

```text
Thing Description
  -> planning
  -> logical plan
  -> binding artifact
  -> immutable compiled plan set
  -> Servient orchestration
  -> Protocol Binding I/O
```

## 关键问题

- Servient 与 Binding 谁拥有执行权；
- 为什么运行时不在热路径重新解释 TD；
- 为什么 producer exposure 需要事务性激活；
- 为什么计划、route、call 和 subscription 都需要显式生命周期；
- 静态链接如何与平台级动态扩展共存。

文章必须重新读取主项目最新正式规范，不能把本 README 当成架构权威。
