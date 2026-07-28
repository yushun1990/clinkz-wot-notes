# WoT 基础与语义模型

本系列面向熟悉 MQTT、HTTP 或一般物联网平台，但尚未系统理解 W3C Web of
Things 的读者。

## 目标

解释：

- WoT 为什么不是一种新传输协议；
- TD 如何提供协议之上的语义契约；
- Property、Action、Event 与 Form 的关系；
- 为什么运行时不应该从 MQTT Topic 或 HTTP URL 开始建模。

## 推荐顺序

1. WOT-001｜为什么物联网平台不应该从 MQTT Topic 开始设计
2. WOT-002｜W3C WoT 不是一种新协议
3. WOT-003｜Thing Description 是语义契约
4. WOT-004｜一个 Thing 有多个 Form 时如何选择
5. WOT-005｜ConsumedThing 与 ProducedThing
6. WOT-006｜WoT Directory 的边界

## 写作边界

本系列可以使用 ClinkZ-WoT 作为真实案例，但不应要求读者先理解项目内部
work package、gate 或治理机制。
