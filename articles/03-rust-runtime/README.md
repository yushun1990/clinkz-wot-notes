# Rust 运行时机制

本系列从 ClinkZ-WoT 的真实约束出发，讨论可迁移到其他系统的 Rust 运行时
问题。

## 主题

- generation 与陈旧异步结果；
- cancellation 与 cleanup；
- 外部回调和锁；
- bounded resources；
- linear Subscription；
- logical time 与 deadline；
- host async 和 `no_std + alloc` manual progress；
- fallible cleanup。

## 写作要求

每篇文章至少提供：

- 一个失败时序；
- 一个所有权或生命周期边界；
- 一个 Rust 类型、trait、state machine 或 compile fixture 映射；
- 当前项目实现状态。

不要把“Rust 所有权很安全”作为结论。必须解释具体由哪个类型或协议保证安全。
