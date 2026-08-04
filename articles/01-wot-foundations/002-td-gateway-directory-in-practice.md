---
id: "WOT-002"
title: "重新理解 WoT | 现场设备不支持 WoT，Thing Description 从哪里来？"
subtitle: "从业务能力建模、现场点位映射，到网关暴露 Thing 与 Directory 发现"
series: "重新理解 WoT"
series_order: 2
status: "DRAFTING"
author: "yushun1990"
created: "2026-07-29"
updated: "2026-08-04"
summary: "Thing Description 并不要求由现场设备生成，也不应由某一个人逐字段手写。业务人员定义 Thing 应提供的能力，集成人员完成现场协议和点位映射，网关或平台生成具体 TD 并暴露可执行接口，Directory 则可选地帮助应用发现这些 Thing。"

clinkz_wot:
  repository: "https://github.com/yushun1990/clinkz-wot"
  branch: "master"
  commit: "30485b1a51470f328e79453ba0e82e3358c14f79"
  inspected_at: "2026-08-04"

publication:
  platform: "zhihu"
  published_at: null
  canonical_url: null

related:
  previous: "WOT-001"
  next: "WOT-003"
  articles:
    - "WOT-001"
    - "WOT-003"
  docs:
    - "docs/architecture/10-primary-data-flows.md"
    - "docs/work-packages/WP-500-discovery.md"
  source:
    - "td/src/lib.rs"
    - "td/src/thing.rs"
    - "td/src/thing_model.rs"
  tests: []
---

# 重新理解 WoT | 现场设备不支持 WoT，Thing Description 从哪里来？

> 本文基于 ClinkZ-WoT commit `30485b1a51470f328e79453ba0e82e3358c14f79`。
>
> ClinkZ-WoT 已经具有 Thing Description 与 Thing Model 的数据结构、序列化、反序列化、校验和扩展成员保留能力。本文最后涉及的 Discovery 与 Directory 客户端边界属于 active v5 架构，WP-500 仍处于计划阶段，不能理解为完整 Directory 客户端已经实现。

上一篇文章解释了 W3C Web of Things 为什么存在。

真实物联网系统即使已经通过 MQTT、HTTP、Modbus 或其他协议打通通信，上层应用仍然缺少一套共同、机器可读的 Thing 接口。

Thing Description（TD）可以描述一个 Thing 的 Property、Action、Event、数据结构、安全要求和访问入口。

但理解到这里，新的问题马上出现了：

> 现场的 PLC、仪表和控制器根本不认识 WoT，它们怎么可能提供 TD？

更具体地说：

- 一块只能读取 Modbus 寄存器的压力仪表，TD 从哪里来？
- 一个泵站应该建立一个 Thing，还是把 PLC、变频器和仪表分别建成 Thing？
- TD 应该保存在设备、网关还是云平台？
- 是不是需要业务人员手写大量 JSON-LD？
- Directory 是不是所有设备数据都必须经过的中心节点？
- 应用找到 TD 后，又是谁真正操作现场设备？

这些问题比逐个解释 TD 字段更接近真实工程。

本文继续使用一个供水泵站，完整走一遍从现场设备到 WoT Consumer 的接入过程。

## 一个泵站里，并不存在现成的 Thing

假设一个泵站包含：

- 一台 PLC；
- 一块出口压力仪表；
- 一台流量计；
- 一台变频器；
- 一套电机保护装置；
- 一台工业网关。

现场可能同时存在 Modbus RTU、Modbus TCP、OPC UA 和厂商私有协议。

对于设备集成人员来说，泵站可能表现为：

```text
Modbus register 40021
PLC status word bit 3
PLC coil 00017
motor protector alarm code 7
```

但业务系统并不想直接操作这些地址。

业务真正关心的是：

```text
Property: outletPressure
Property: targetPressure
Property: runningState

Action: start
Action: stop

Event: overload
```

从寄存器、状态位和厂商报文，到 Property、Action 和 Event，中间还缺少一次建模和映射。

这个过程不会因为引入 WoT 而自动完成。

## TD 通常不是由一个人手写出来的

一份可以真实运行的 TD，同时包含业务知识、设备知识和网络接口信息。

很少有人能够独立掌握这三部分。

更合理的方式，是把责任分给不同角色。

### 工艺人员定义“这个对象应该提供什么”

工艺或业务人员知道：

- 出口压力代表什么；
- 使用什么单位；
- 正常范围是多少；
- 哪些状态只能读取；
- 启停操作需要满足什么条件；
- 什么情况应该产生过载告警。

他们适合定义稳定的业务能力：

```text
outletPressure
targetPressure
runningState
start
stop
overload
```

但他们通常不知道：

- 压力来自哪个寄存器；
- 状态位怎样解码；
- 最终使用 HTTP、MQTT 还是 Zenoh；
- Consumer 应该访问哪个网络地址。

业务人员不应该为了建立 Thing 模型，被迫学习所有现场协议。

### 集成人员定义“这些能力在现场对应什么”

设备或系统集成人员知道：

```text
outletPressure
  <- Modbus register 40021
  <- unsigned 16-bit integer
  <- scale 0.01
  <- unit kPa

runningState
  <- PLC status word bit 3

start
  -> write PLC coil 00017

overload
  <- motor protector alarm code 7
```

他们负责确认：

- 点位和寄存器地址；
- 编码与字节序；
- 比例系数；
- 读写方式；
- 控制时序；
- 告警来源；
- 现场联锁。

这一层负责把真实设备行为映射成稳定的 Thing 能力。

### 网关或平台定义“应用怎样访问这些能力”

平台和 WoT Runtime 还需要决定：

- 为哪些对象生成 TD；
- 给 Thing 分配什么 ID；
- 通过哪些 Form 暴露交互；
- 使用什么安全机制；
- TD 保存在哪里；
- 是否把 TD 注册到 Directory；
- Consumer 获取 TD 后怎样执行交互。

因此，一份具体 TD 更可能是多类信息组合后的产物：

```text
工艺知识
    |
    v
Thing 能力模型
    |
    v
现场协议和点位映射
    |
    v
网关或平台生成具体 TD
    |
    v
托管或注册 TD
    |
    v
Consumer 获取并消费 TD
```

TD 是这个过程的结果，而不是整个过程的起点。

## 先定义一类泵站，再创建具体实例

如果项目中有几百个同类型泵站，没有必要为每个泵站重新讨论一次能力名称、数据类型和单位。

可以先建立一个 Thing Model：

```text
PumpStation Model

properties:
  outletPressure
  targetPressure
  runningState

actions:
  start
  stop

events:
  overload
```

Thing Model 描述的是一类 Thing 应该具有什么能力。

它可以定义：

- Property、Action 和 Event；
- 数据类型与单位；
- 数值范围；
- 枚举值；
- 必选和可选能力；
- 可复用的领域语义。

此时泵站可能尚未安装。

设备编号、IP 地址、网关位置和具体协议入口都还没有确定。

等到 `Pump Station 17` 真正完成部署后，平台再根据 Thing Model、实例信息和现场映射生成具体 TD：

```text
id:
  urn:example:pump-station:17

title:
  Pump Station 17

location:
  Qingdao Water Supply Area A

forms:
  通过 edge-gateway-03 暴露

security:
  需要 operator 权限
```

Thing Model 回答的是：

> 这类对象通常具有什么能力？

Thing Description 回答的是：

> 这个具体对象是谁，现在可以通过什么接口操作？

## 现场设备不需要原生支持 WoT

假设 `outletPressure` 最终通过网关的 HTTPS 接口暴露。

概念性的 TD 片段可能是：

```json
{
  "id": "urn:example:pump-station:17",
  "title": "Pump Station 17",
  "properties": {
    "outletPressure": {
      "type": "number",
      "unit": "kPa",
      "readOnly": true,
      "forms": [
        {
          "href": "https://edge-03.example.com/things/pump-17/properties/outletPressure",
          "op": "readproperty"
        }
      ]
    }
  }
}
```

这并不意味着压力仪表原生支持 HTTPS，也不意味着仪表内部保存着这份 JSON。

真实的数据流可能是：

```text
Consumer
    |
    | read Property "outletPressure"
    v
Gateway HTTPS Form
    |
    | 根据映射执行现场访问
    v
Modbus register 40021
    |
    | 解码、缩放、转换
    v
Pressure value
```

TD 中的 Form 指向网关提供的网络入口。

网关再根据内部映射读取真实寄存器，并把结果转换成 TD 声明的数据结构。

因此，一个传统设备成为可被 WoT Consumer 使用的 Thing，并不要求设备自己实现 WoT：

```text
传统设备
  + 外部提供的 Thing Description
  + 能够执行 TD 中交互的网关或代理
  = 可供 Consumer 使用的 Thing
```

这对存量设备尤其重要。

许多工业设备：

- 无法修改固件；
- 只有串口或现场总线；
- 没有能力托管 JSON 文档；
- 可能通过专用网关访问；
- 可能只在特定时间上线。

要求这些设备全部原生实现 WoT，既不现实，也没有必要。

## 网关不只是把 Modbus 转成 MQTT

传统网关经常被理解成一个协议转换器：

```text
Modbus -> MQTT
```

这种转换解决了消息怎样进入平台的问题，但不一定形成稳定的业务接口。

例如，网关可能只是把寄存器值包装成：

```json
{
  "40021": 352,
  "40022": 1,
  "40023": 0
}
```

上层应用仍然需要知道：

- `40021` 代表出口压力；
- 数值需要乘以 `0.01`；
- `40022` 的不同位代表什么；
- 哪个值可以写；
- 写入前是否需要联锁检查。

WoT 网关可以承担更完整的 Intermediary 角色：

- 连接现场设备；
- 将寄存器、Topic 和厂商接口映射成 Property、Action 与 Event；
- 组合多个物理设备；
- 形成更符合业务边界的虚拟 Thing；
- 生成或补全具体 TD；
- 托管 TD；
- 暴露 HTTP、MQTT、CoAP 或 Zenoh Form；
- 执行本地校验、鉴权和联锁；
- 将 TD 注册到 Directory；
- 代理 Consumer 与真实设备之间的交互。

对于泵站，可以把多个现场组件组合成一个 Thing：

```text
压力仪表 ----+
流量计 ------+
PLC ---------+--> Gateway --> PumpStation Thing
变频器 ------+
保护装置 ----+
```

业务应用面对的是“泵站”，而不是五个没有业务关系的通信端点。

## 一个泵站应该是一个 Thing，还是多个 Thing？

这个问题没有唯一答案。

可以把整个泵站建模为一个 Thing：

```text
PumpStation Thing
  - outletPressure
  - flow
  - runningState
  - start
  - stop
  - overload
```

也可以分别建立：

```text
PressureSensor Thing
FlowMeter Thing
PLC Thing
FrequencyConverter Thing
PumpStation Thing
```

甚至可以同时存在。

底层使用几种协议，并不能直接决定 Thing 的边界。

更应该考虑以下因素。

### 业务边界

应用真正操作的是一块仪表，还是一个完整泵站？

### 生命周期

压力仪表是否会独立更换、独立维护和独立登记？

### 权限边界

读取压力和启动水泵是否属于不同权限？

### 复用方式

其他应用是否需要单独消费流量计或变频器？

### 故障边界

一个组件离线时，整个 Thing 是否仍能提供部分能力？

因此，Thing 首先是一个接口和生命周期边界，而不是物理外壳的机械映射。

## TD 应该保存在哪里？

TD 不一定保存在物理设备中。

常见方式大致有三种。

### Thing 自己提供 TD

具有完整网络能力的设备或软件服务，可以直接托管自己的 TD：

```text
Consumer
  -> 获取 Thing 自己的 TD
  -> 根据 Form 直接交互
```

这种方式适合原生支持 WoT、长期在线并且可以直接访问的设备或服务。

### 网关或代理托管 TD

存量设备和资源受限设备更适合由网关托管：

```text
Legacy Device
      |
      v
Gateway / Intermediary
  - 托管 TD
  - 暴露 Form
  - 转换现场协议
```

TD 描述的可以是一台设备，也可以是网关聚合出的虚拟 Thing。

### 平台或 Directory 管理 TD

当系统中有大量 Thing、多个网关和多个应用时，应用很难预先知道每一份 TD 的地址。

这时可以把 TD 注册到 Thing Description Directory：

```text
Gateway A ----register----+
Gateway B ----register----+--> Directory
Cloud Thing --register----+

Application ---search----> Directory
Application <----TD------- Directory
```

网关可以继续托管原始 TD，同时将其登记到 Directory。

这三种方式并不冲突。

## TD Server 和 Directory 不是一回事

理解 TD 的分发过程时，需要区分两个概念。

### TD Server：通过已知地址取得一份 TD

TD Server 可以是：

- Thing 自身；
- 网关；
- 边缘服务；
- 云平台；
- 静态 Web 服务。

它解决的问题是：

> 我已经知道地址，怎样取得这份 TD？

### Thing Description Directory：管理和查找一组 TD

Directory 管理描述其他 Thing 的 TD 集合。

它可以提供：

- 注册；
- 读取；
- 更新；
- 删除；
- 列举；
- 搜索。

它解决的问题是：

> 系统中存在大量 Thing，我怎样找到符合条件的 TD？

两者的区别可以概括为：

```text
TD Server
  -> 从已知 URL 获取一份 TD

Directory
  -> 在一组 TD 中登记、管理和查找
```

## Directory 不是设备数据总线

看到“所有 TD 都注册到 Directory”，很容易把 Directory 想成 WoT 系统的中心节点。

但 Directory 管理的是接口元数据，不是设备运行数据。

它不等于：

- MQTT Broker；
- 时序数据库；
- 遥测数据中心；
- 规则引擎；
- 设备消息路由器；
- 每次属性读取的中转站；
- 每个 Event 的转发服务。

典型流程是：

```text
1. Application 查询 Directory
2. Directory 返回目标 Thing 的 TD
3. Consumer Servient 解析并消费 TD
4. Runtime 根据 TD 中的 Form 与 Thing 交互
```

完整关系更接近：

```text
                 +----------------+
                 |   Directory    |
                 |   stores TDs   |
                 +-------+--------+
                         ^
                    search/register
                         |
Application ------------+
    |
    | use Form from TD
    v
Gateway / Thing -----------------> Physical Device
```

Directory 通常不位于每一次设备调用的数据路径上。

真正的交互访问网关还是设备本身，由 TD 中的 Form 决定。

## 从设备接入到应用调用的完整过程

现在把整个过程连接起来。

### 第一步：定义业务能力

业务或工艺人员定义：

```text
outletPressure
runningState
start
stop
overload
```

这些能力可以先进入 Thing Model、设备模板或平台领域模型。

### 第二步：建立现场映射

集成人员建立能力与真实点位的关系：

```text
outletPressure
  <- Modbus register 40021

runningState
  <- PLC status word bit 3

start
  -> write PLC coil 00017

overload
  <- motor protector alarm code 7
```

### 第三步：形成可以暴露的 Thing

网关或平台将业务模型与现场映射组合起来：

- 分配 Thing ID；
- 加入实例元数据；
- 声明数据结构；
- 配置安全要求；
- 生成对外 Form；
- 建立实际网络路由。

### 第四步：发布 TD

规模较小时，可以：

- 让网关直接提供 TD；
- 把 TD URL 静态配置给应用；
- 将 TD 文件随应用部署。

规模较大时，可以把 TD 注册到 Directory。

### 第五步：应用发现目标 Thing

应用可以按照系统支持的条件查找：

- Thing ID；
- 类型；
- 位置；
- 所具有的 Property、Action 或 Event；
- 领域语义标签。

Directory 返回一个或多个 TD。

具体支持哪些搜索方式，由 Directory 的实现决定。

### 第六步：WoT Runtime 消费 TD

Consumer 所在的 Servient 获取 TD 后执行：

```text
TD document
    |
    v
parse and validate
    |
    v
create ConsumedThing
    |
    v
application requests read/invoke/subscribe
    |
    v
select Form
    |
    v
Protocol Binding executes communication
```

应用表达的是：

```text
read outletPressure
```

而不是：

```text
GET 某个 URL
读取 40021 寄存器
订阅某个 Topic
```

具体访问行为由 TD、Form、Protocol Binding 和网关映射共同完成。

## 什么时候不需要 Directory？

Directory 是可选的系统组件，不是使用 WoT 的前置条件。

以下场景通常不需要 Directory：

- 应用只操作少量已知设备；
- TD 随应用一起部署；
- TD URL 长期固定；
- 系统使用静态配置；
- 局域网已有简单发现机制；
- 运行时检索带来的复杂度高于收益。

最简单的系统完全可以这样运行：

```text
Application
  -> load ./pump-17.td.json
  -> consume TD
  -> interact
```

当下面的问题逐渐出现时，Directory 才更有价值：

- Thing 数量较多并持续变化；
- 网关会动态加入或离开；
- 多个应用需要共享设备元数据；
- 应用需要按照能力、位置或类型搜索 Thing；
- TD 需要统一更新和版本管理；
- 不同用户只能查看特定 TD。

Directory 更接近系统级元数据基础设施，而不是 WoT Runtime 必须内置的功能。

## TD 变化以后会发生什么？

TD 并不是永远不变的静态文件。

以下变化都可能产生新的 TD：

- 新增或删除能力；
- 网关地址改变；
- Form 改变；
- 对外协议改变；
- 安全要求改变；
- Thing 迁移到另一台网关；
- 本地设备改由云端代理。

通常由拥有暴露接口的一方更新 TD：

```text
Thing / Gateway / Platform
    |
    v
generate updated TD
    |
    v
update TD Server resource
    |
    v
update Directory entry
```

Directory 可以保存和分发新的描述，但它不能自动替已经运行的 Consumer 决定：

- 什么时候重新获取 TD；
- 是否接受新的接口；
- 旧调用怎样结束；
- 已建立的订阅怎样取消；
- 新 TD 是否应当进入新的 generation；
- 旧资源何时可以安全清理。

从这一刻开始，问题已经不再只是 TD 的存放和发现，而是运行时生命周期管理。

这也是后续 ClinkZ-WoT 架构文章需要重点讨论的内容。

## ClinkZ-WoT 为什么不实现 Directory 服务端？

ClinkZ-WoT 的定位是协议中立的 WoT Runtime，而不是完整物联网平台。

在 active v5 架构中，Directory 的职责被明确拆开：

```text
Directory Service
  -> 保存和索引 TD
  -> 执行服务端查询
  -> 处理授权、租户和存储策略
  -> 属于外部平台或独立服务

ClinkZ-WoT Discovery Client
  -> 查询远程 Directory
  -> 发布或更新 TD
  -> 观察 Directory 变化
  -> 解析返回的 TD Document

ClinkZ-WoT Consume Path
  -> 校验获得的 TD
  -> 进入统一 planning 流程
  -> 建立 ConsumedThing
```

这意味着构造一个 Servient 时，不会自动创建进程内 Directory。

ClinkZ-WoT 也不负责：

- Directory 数据库存储；
- 服务端查询执行；
- 多租户策略；
- TD 索引；
- Directory 复制；
- 服务端 watch fan-out；
- Directory 服务可用性。

运行时只负责客户端边界。

无论 TD 来自：

- 应用直接传入；
- 本地文件；
- 网关；
- TD Server；
- Directory；
- 其他发现机制；

最终都应该进入相同的 TD 校验和消费流程。

这种边界允许不同系统自行选择拓扑：

```text
小型系统
  -> 直接加载 TD，不部署 Directory

边缘系统
  -> 从网关获取 TD

平台系统
  -> 部署独立 Directory

已有生态
  -> 对接第三方 Directory
```

ClinkZ-WoT 不强迫所有系统采用同一种元数据基础设施。

必须强调当前状态：这一客户端边界属于已接受的 active v5 设计，WP-500 仍处于 Planned 状态，不能据此声称新的完整 Directory 客户端已经进入产品实现。

## 总结

第一，现场设备不需要原生支持 WoT。业务人员定义 Thing 能力，集成人员建立现场点位映射，网关或平台可以为存量设备和聚合对象生成并托管 TD。

第二，TD Server 提供一份已知 TD，Directory 管理和查找一组 TD。Directory 保存的是接口元数据，通常不位于实际设备交互的数据路径中。

第三，从设备接入到应用调用的完整过程是：定义业务能力、建立现场映射、生成具体 TD、可选地注册到 Directory，再由 Consumer Servient 消费 TD，并通过 Form 与 Protocol Binding 执行真实通信。

理解 TD 从哪里来、保存在哪里以及如何被找到之后，下一个问题才真正浮现出来：

> 同一个 Thing 的同一项能力同时提供 HTTP、MQTT、CoAP 或 Zenoh Form 时，Runtime 到底应该选择哪一个，又由谁执行？

## 延伸阅读

- 上一篇：[重新理解 WoT | W3C WoT 到底解决什么问题？](./001-what-does-wot-solve.md)
- 下一篇：重新理解 WoT | 同一个 Thing 如何通过不同协议交互
- 相关规范：ClinkZ-WoT `docs/architecture/10-primary-data-flows.md`
- 相关工作包：ClinkZ-WoT `docs/work-packages/WP-500-discovery.md`

## 项目资料

- ClinkZ-WoT commit：`30485b1a51470f328e79453ba0e82e3358c14f79`
- `td/src/lib.rs`
- `td/src/thing.rs`
- `td/src/thing_model.rs`
- `docs/architecture/10-primary-data-flows.md`
- `docs/work-packages/WP-500-discovery.md`

## 外部资料

- [W3C Web of Things (WoT) Architecture 1.1](https://www.w3.org/TR/wot-architecture11/)
- [W3C Web of Things (WoT) Thing Description 1.1](https://www.w3.org/TR/wot-thing-description11/)
- [W3C Web of Things (WoT) Discovery](https://www.w3.org/TR/wot-discovery/)

<!--
内部事实分类：

- TD、Thing Model、TD Server、Thing Description Directory、Consumer、Intermediary 和 Discovery 流程：外部标准事实。
- 泵站、人员分工、点位映射和部署图：CONCEPTUAL COMPOSITE，用于说明常见工程协作，不声称对应某个具体项目。
- ClinkZ-WoT TD/TM 数据结构、序列化、校验与扩展成员保留：IMPLEMENTED。
- Discovery 结果进入统一 consume path、Servient 不隐式创建 Directory、Directory client-only：ACCEPTED_DESIGN。
- WP-500 Directory and Discovery Client Runtime：PLANNED。

发布前检查：

- [x] 已与第一篇划分内容边界
- [x] 已区分 Thing Model 与具体 TD
- [x] 已区分 TD Server、Directory、Gateway 与 Runtime
- [x] 未把 Directory 描述成设备数据总线
- [x] 未要求现场设备原生支持 WoT
- [x] 已区分 ClinkZ-WoT accepted design 与当前实现
- [ ] 完成作者理解校验
- [ ] 完成第二轮技术事实校验
- [ ] 制作泵站接入流程图
- [ ] 回填知乎发布信息
-->
