---
id: "WOT-002"
title: "重新理解 WoT | TD、网关与 Directory 在真实系统中如何协作"
subtitle: "从业务能力建模，到网关暴露 Thing，再到应用发现和消费 TD"
series: "重新理解 WoT"
series_order: 2
status: "DRAFTING"
author: "yushun1990"
created: "2026-07-29"
updated: "2026-07-29"
summary: "在真实项目中，Thing Description 通常不是由业务人员逐字段手写，也不要求现场设备原生支持 WoT。业务与工艺人员定义 Thing 应提供的能力，集成人员将现场协议和点位映射到这些能力，网关或平台生成并托管具体 TD，Directory 再帮助应用发现合适的 Thing。本文用一条完整的数据流说明设备、网关、TD Server、Thing Description Directory、Consumer 与 WoT Runtime 分别承担什么责任。"

clinkz_wot:
  repository: "https://github.com/yushun1990/clinkz-wot"
  branch: "master"
  commit: "f453f165c2ea775e5f0d10c36f1e419fcc1d79f3"
  inspected_at: "2026-07-29"

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
    - "WOT-006"
  docs:
    - "docs/architecture/10-primary-data-flows.md"
    - "docs/work-packages/WP-500-discovery.md"
  source:
    - "td/src/lib.rs"
    - "td/src/thing.rs"
    - "td/src/thing_model.rs"
  tests: []
---

# 重新理解 WoT | TD、网关与 Directory 在真实系统中如何协作

> 本文基于 ClinkZ-WoT commit `f453f165c2ea775e5f0d10c36f1e419fcc1d79f3`。
>
> ClinkZ-WoT 当前已经实现 TD/TM 数据结构、序列化、反序列化、校验和扩展成员保留。本文最后讨论的 Discovery 与 Directory 客户端边界属于 active v5 架构；对应 WP-500 仍处于计划阶段，不能理解为完整 Directory 客户端已经实现。

上一篇文章解释了 W3C WoT 为什么存在：真实系统即使已经打通通信，上层应用仍然缺少共同、机器可读的 Thing 接口。

但是仅仅知道“TD 描述 Property、Action 和 Event”仍然不够。

在真实项目中，人们很快会遇到更实际的问题：

- 现场设备根本不支持 WoT，TD 从哪里来？
- 一个 PLC、几块仪表和一个变频器，应该分别建立 Thing，还是组合成一个泵站 Thing？
- TD 应该放在设备里、网关里，还是平台里？
- Directory 是不是所有设备数据都必须经过的中心节点？
- 应用找到 TD 以后，究竟怎样调用真实设备？
- 编写 TD 的人需要懂 HTTP、MQTT、Modbus 和 JSON-LD 吗？

这些问题比罗列 TD 字段更接近 WoT 的实际应用。

本文用一个供水泵站的完整接入过程，解释 Thing Model、Thing Description、网关、TD Server、Thing Description Directory、Consumer 和 WoT Runtime 如何协作。

## 先从一个真实的角色分工开始

假设一个泵站包含：

- PLC；
- 压力仪表；
- 流量计；
- 变频器；
- 电机保护装置；
- 一台工业网关。

现场设备可能使用 Modbus RTU、Modbus TCP、OPC UA 或厂商私有协议。业务系统并不想直接操作寄存器地址，而是希望看到：

```text
Property: outletPressure
Property: targetPressure
Property: runningState
Action: start
Action: stop
Event: overload
```

这里至少涉及三类人员。

### 工艺或业务人员知道“它应该做什么”

工艺人员知道：

- 出口压力的业务含义；
- 正常范围和单位；
- 哪些状态允许修改；
- 启停操作需要哪些前置条件；
- 什么情况应产生过载告警。

他们不一定知道压力值来自 PLC 的哪个寄存器，也不需要知道最终通过 HTTP、MQTT 还是 Zenoh 暴露。

### 设备与集成人员知道“现场接口在哪里”

集成人员知道：

- `40021` 寄存器代表出口压力；
- 某个线圈控制启动；
- 变频器运行状态需要怎样解码；
- 电机保护装置怎样上报告警；
- 哪些操作必须在本地网关中执行联锁检查。

他们负责把现场协议和点位映射为稳定的 Thing 能力。

### 平台或运行时知道“怎样把能力暴露给应用”

平台与 WoT Runtime 决定：

- 为哪些 Thing 生成具体 TD；
- TD 中使用哪些 Form；
- 通过哪个网关或服务接收调用；
- 如何发布、注册和更新 TD；
- Consumer 获取 TD 后怎样执行交互。

因此，TD 通常不是某一个角色从头手写到底的文件。

更现实的过程是：

```text
工艺知识
  -> Thing 能力模型
  -> 现场协议与点位映射
  -> 网关/平台生成具体 TD
  -> 发布或注册 TD
  -> 应用发现并消费 TD
```

## Thing Model：先描述“一类设备应该具有什么能力”

如果一个项目有几百个同型号泵站，没有必要让每个站点重新定义一遍业务能力。

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

Thing Model 更接近一类 Thing 的能力模板。

它可以规定：

- Property、Action 和 Event 的名称与含义；
- 数据类型、单位、范围和枚举；
- 哪些能力是必需的，哪些是可选的；
- 可复用的领域语义和元数据。

此时设备可能尚未安装，IP 地址、网关地址、设备编号和具体协议入口都还不存在。

W3C WoT Architecture 将 Thing Model 定义为一类具有相同能力的 Thing 的描述。与具体 TD 相比，Thing Model 不包含足够的信息去识别并操作某一个实例。

这与实际项目中的角色分工很吻合：业务和工艺人员更适合参与定义 Thing Model，而不是负责填写每个现场实例的网络地址。

## Thing Description：把能力模板实例化为可交互的 Thing

当 `Pump Station 17` 完成部署后，系统才能生成具体 TD。

这份 TD 会补充实例信息，例如：

```text
id:
  urn:example:pump-station:17

location:
  青岛某供水区域

forms:
  通过 edge-gateway-03 提供的网络入口访问

security:
  需要 operator 权限
```

TD 描述的不只是“这类泵站通常具有什么能力”，而是：

> 这个具体 Thing 现在提供哪些交互，以及 Consumer 应怎样访问它。

例如，泵站 TD 的一部分可能是：

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
  },
  "actions": {
    "start": {
      "forms": [
        {
          "href": "https://edge-03.example.com/things/pump-17/actions/start",
          "op": "invokeaction"
        }
      ]
    }
  }
}
```

这并不意味着压力仪表和 PLC 原生支持 HTTP。

TD 中的 Form 指向网关提供的网络接口，而网关内部再把交互转换为现场协议。

```text
Consumer
  -> read Property "outletPressure"
  -> HTTPS Form on gateway
  -> gateway mapping
  -> Modbus register 40021
  -> pressure value
```

Thing Description 因此成为业务能力与真实通信入口之间的连接点。

## 现场设备不需要原生理解 WoT

这是 WoT 实际落地时最重要的一点。

大量存量设备：

- 无法修改固件；
- 没有能力托管 JSON 文档；
- 只暴露寄存器、串口帧或厂商接口；
- 可能长期离线或只在特定时段唤醒。

W3C WoT 并不要求它们重新实现一套 WoT 协议栈。

Thing Description 可以由设备自己提供，也可以由外部服务托管。对于资源受限设备和后接入 WoT 的存量设备，由网关或平台提供 TD 是标准明确支持的模式。

因此，一个设备成为 Web Thing，不等于它必须亲自完成所有工作。

更准确地说：

```text
物理设备
  + 外部描述
  + 可以执行该描述中交互的网络入口
  = 可以被 WoT Consumer 使用的 Thing
```

## 网关不只是“协议转换器”

在传统物联网架构中，网关常被理解为：

```text
Modbus -> MQTT
```

但在 WoT 系统中，网关可以承担更完整的 Intermediary 角色。

它可能同时负责：

- 连接现场设备；
- 将寄存器、Topic 或私有接口映射为 Property、Action 和 Event；
- 聚合多个物理设备，形成一个虚拟 Thing；
- 提供对外的 HTTP、MQTT、CoAP 或 Zenoh 接口；
- 执行本地校验、联锁和访问控制；
- 生成或补全具体 TD；
- 托管 TD；
- 将 TD 注册到 Directory；
- 在本地设备不可直接访问时代理实际交互。

W3C WoT Architecture 中的 Edge Device 通常同时具有本地计算和跨协议桥接能力；Intermediary 则可以代理、增强或虚拟化其他 Thing。

对于前面的泵站，网关可以把多个现场设备组合为一个更符合业务边界的虚拟 Thing：

```text
压力仪表 ----+
流量计 ------+
PLC ---------+--> Gateway --> PumpStation Thing
变频器 ------+
保护装置 ----+
```

业务应用面对的是“泵站”，而不是被迫分别操作五种现场设备。

当然，也可以为压力仪表、变频器和泵站分别建立 TD。究竟怎样划分 Thing，不由协议决定，而由业务边界、权限边界、生命周期和复用方式决定。

## TD 放在哪里：设备、网关和平台都可能

W3C WoT 并不规定 TD 必须与物理设备存放在一起。

常见方式有三种。

### 方式一：Thing 自己托管 TD

具备完整网络能力的设备或服务可以提供自己的 TD。

```text
Consumer
  -> 获取设备自己的 TD
  -> 根据 TD 中的 Form 直接交互
```

这种模式适合：

- 原生支持 WoT 的设备；
- 网络长期在线的服务；
- 点对点或较小规模系统。

### 方式二：网关或代理服务托管 TD

存量设备和资源受限设备通常更适合这种模式。

```text
Legacy Device
      |
      v
Gateway / Intermediary
  - 托管 TD
  - 暴露网络接口
  - 转换现场协议
```

TD 描述的 Thing 可以是单个设备，也可以是网关聚合出的虚拟对象。

### 方式三：平台或 Directory 集中管理 TD

当系统中存在大量动态设备、多个网关和多个应用时，应用不可能预先知道每个 TD 的 URL。

这时可以把 TD 注册到 Thing Description Directory。

```text
Gateway A --register--> Directory
Gateway B --register--> Directory
Cloud Thing --register-> Directory

Application --search--> Directory
Application <--TD------- Directory
```

这三种方式并不互斥。

一个网关可以自己托管 TD，同时把 TD 注册到 Directory；Directory 保存的是可检索副本或登记信息，具体交互仍然按照 TD 中的 Form 执行。

## TD Server 与 Directory 不是同一个东西

为了理解 TD 的分发，需要区分两个概念。

### Thing Description Server：提供一份 TD

Thing Description Server 本质上是一个可以通过 URL 获取 TD 的 Web 资源。

它可能位于：

- Thing 自身；
- 网关；
- 边缘服务；
- 云端设备管理平台；
- 静态 Web 服务。

TD Server 的核心问题是：

> 已经知道地址后，怎样取得这份 TD？

### Thing Description Directory：管理一组 TD

Thing Description Directory 管理描述其他 Thing 的 TD 集合，并提供登记、读取、更新、删除、列举以及可选搜索能力。

Directory 解决的问题是：

> 当系统中有很多 Thing，而且调用方事先不知道具体地址时，怎样找到符合条件的 TD？

因此：

```text
TD Server
  -> 根据已知 URL 获取一份 TD

Directory
  -> 在一组 TD 中登记、管理和查找
```

Directory 自己也是一个网络服务，甚至可以拥有描述自身接口的 TD。

## Directory 不是设备数据总线

这是实际应用中很容易产生的误解。

Directory 保存和检索的是 Thing Description，也就是接口元数据；它不等于：

- MQTT Broker；
- 时序数据库；
- 遥测数据中心；
- 设备消息路由器；
- 每次 Property Read 的中转站；
- 每次 Event 的转发服务。

典型交互流程是：

```text
1. Application 查询 Directory
2. Directory 返回目标 Thing 的 TD
3. Application / WoT Runtime 解析 TD
4. Runtime 根据 Form 与目标 Thing 交互
```

第 4 步可能直接访问设备，也可能访问网关或云端 Intermediary，取决于 TD 中的 Form。

```text
                 +----------------+
                 |   Directory    |
                 |  stores TDs    |
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

Directory 不必位于实际调用的数据路径上。

如果 TD 的 Form 指向网关，那么网关位于调用路径；如果 Form 指向设备本身，则 Consumer 可以直接访问设备。

## 应用怎样从“发现”走到“调用”

把整个过程连起来，可以得到一条更实际的数据流。

### 第一步：定义业务能力

工艺和业务人员定义泵站需要向外提供：

```text
outletPressure
runningState
start
stop
overload
```

这些能力可以先进入 Thing Model、设备模板或平台中的领域模型。

### 第二步：配置现场映射

集成人员告诉网关：

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

这一步属于设备接入和 Binding 实现，不是让业务人员理解寄存器协议。

### 第三步：网关形成可对外暴露的 Thing

网关将现场映射与业务模型结合，形成一个 ProducedThing 或等价的内部对象，并生成具体 TD：

- 分配 Thing ID；
- 填入实例元数据；
- 生成对外 Form；
- 声明安全要求；
- 暴露网络路由。

### 第四步：发布 TD

规模较小时，可以把 TD URL 静态配置给应用，或者由网关直接提供 TD。

规模较大时，网关把 TD 注册到 Directory。

### 第五步：应用发现目标 Thing

应用可以按照系统支持的查询方式查找：

- 指定 Thing ID；
- 指定设备类型；
- 指定位置；
- 指定某项 Property、Action 或 Event；
- 指定领域语义标签。

Directory 返回一个或多个 TD。具体能否按照语义或复杂条件检索，取决于 Directory 的实现和它支持的查询能力。

### 第六步：WoT Runtime 消费 TD

Consumer 所在的 Servient 获取 TD 后：

```text
TD document
  -> parse and validate
  -> create ConsumedThing
  -> application requests read/invoke/subscribe
  -> select Form
  -> Protocol Binding executes communication
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

具体协议行为由 TD、Form、Protocol Binding 和网关映射共同完成。

## 一个完整的部署图

前面的角色可以组合为：

```text
                       business / process experts
                                  |
                                  v
                         Thing Model / template
                                  |
                                  v
field devices --> gateway mapping + WoT runtime
 Modbus/OPC UA       |             |
 vendor protocol     |             +--> exposes network Forms
                     |             +--> hosts concrete TD
                     |             +--> registers TD
                     |                       |
                     v                       v
               physical operation       TD Directory
                                              |
                                        search / retrieve
                                              |
                                              v
                                       Consumer Servient
                                              |
                                       ConsumedThing API
                                              |
                                      read / write / invoke
                                              |
                                              v
                                    gateway Form or direct Form
```

这张图中没有要求所有设备支持 WoT，也没有要求所有流量经过 Directory。

WoT 统一的是 Thing 的描述和交互入口，而不是强行统一现场设备的实现方式。

## 什么时候根本不需要 Directory

Directory 是可选组件，不是构造 Servient 的前置条件。

以下场景可能不需要 Directory：

- 应用只操作一个已知设备；
- TD 随应用一起部署；
- Thing 的 TD URL 固定；
- 局域网通过简单发现机制即可找到 TD；
- 系统规模小，静态配置比运行时检索更可靠。

例如：

```text
Application
  -> load ./pump-17.td.json
  -> consume TD
  -> interact
```

当这些情况出现时，Directory 才开始体现价值：

- Thing 数量多且持续变化；
- 多个网关不断加入和离开；
- 多个应用需要共享设备元数据；
- 需要按照能力、位置、类型或语义查找 Thing；
- 需要集中控制谁能查看哪些 TD；
- TD 需要更新、版本和生命周期管理。

因此，Directory 属于系统级元数据基础设施，而不是 WoT Runtime 的必备内核。

## TD 更新时，谁负责什么

真实系统中的 TD 不是永远不变的。

以下变化可能产生新的 TD：

- Thing 新增或删除能力；
- 网关地址改变；
- 对外协议或 Form 改变；
- 安全要求改变；
- Thing 从一个网关迁移到另一个网关；
- 原来的本地 Thing 被云端 shadow 或代理替代。

通常由拥有暴露接口的一方生成或更新 TD：

```text
Thing / Gateway / Platform
  -> update TD
  -> update TD Server resource
  -> re-register or update Directory entry
```

Directory 负责保存和分发新的描述，但不会自动保证某个已经运行的 Consumer 如何迁移。

Consumer 或 WoT Runtime 仍需要决定：

- 何时重新获取 TD；
- 是否接受新的接口版本；
- 旧的调用和订阅如何结束；
- 新 TD 是否作为新的 Thing generation 使用。

这些生命周期问题会在后续运行时架构文章中继续讨论。

## ClinkZ-WoT 把 Directory 放在哪一层

ClinkZ-WoT 是 WoT Runtime，不是完整物联网平台。

active v5 架构明确区分：

```text
Directory service
  -> 保存、索引、查询和授权管理 TD
  -> 属于外部平台或独立服务

ClinkZ-WoT Discovery client
  -> 查询、发布、观察和解析远程 Directory
  -> 返回带来源与版本信息的 TD document

ClinkZ-WoT consume path
  -> 对获得的 TD 使用与手工输入 TD 相同的校验和规划流程
```

主项目 `docs/architecture/10-primary-data-flows.md` 规定：Discovery 产生带来源信息的 TD 文档，随后进入与应用直接提供 TD 相同的 consume admission 路径。

同时，`DIR-SCOPE-001` 明确要求：

- 构造 Servient 不会自动创建一个进程内 Directory；
- Runtime 不拥有 Directory 的存储、服务端查询、复制和多租户策略；
- Directory 只作为远程客户端能力进入引擎边界。

WP-500 进一步把目标范围限定为 client-only：负责 Directory 请求、分页、watch、publication、取消、资源边界和结果元数据，但不实现 Directory 服务与存储后端。

必须说明当前状态：这一边界属于 active v5 已接受架构；WP-500 仍为计划工作，不能据此声称完整的新 Directory 客户端已经进入产品实现。

这种划分意味着，未来 ClinkZ 平台可以选择：

- 部署符合 W3C API 的独立 Thing Description Directory；
- 对接已有 Directory；
- 在简单系统中完全不部署 Directory；
- 让应用直接传入 TD；
- 让其他 Discovery 机制返回 TD。

ClinkZ-WoT 只要求最终获得一份可以进入 consume 流程的 TD，而不强迫所有系统采用同一种元数据服务拓扑。

## 总结

1. TD 的实际来源可以是 Thing 自身、网关、边缘服务或平台。存量设备不需要原生支持 WoT，网关可以将现场协议映射为 Property、Action 和 Event，并为物理设备或聚合对象生成 TD。
2. TD Server 负责提供一份已知 TD，Thing Description Directory 负责管理和查找一组 TD。Directory 保存的是接口元数据，通常不位于每一次设备调用和数据传输的路径上。
3. 实际落地过程是：业务人员定义能力，集成人员建立现场映射，网关或平台生成具体 TD，Directory 可选地负责发现，Consumer 获取 TD 后再通过 Form 和 Protocol Binding 与真实 Thing 交互。

理解了 TD 在系统中的流转方式，下一篇才能继续深入一个更具体的问题：同一个 Thing 的同一项能力同时拥有 HTTP、MQTT、CoAP 或 Zenoh Form 时，Runtime 与 Protocol Binding 应该怎样执行。

## 延伸阅读

- 上一篇：[重新理解 WoT | W3C WoT 到底解决什么问题？](./001-what-does-wot-solve.md)
- 下一篇：重新理解 WoT | 同一个 Thing 如何通过不同协议交互（计划中）
- 后续主题：重新理解 WoT | WoT Directory 是什么，为什么 Runtime 不应实现 Directory 服务
- 相关项目规范：ClinkZ-WoT `docs/architecture/10-primary-data-flows.md`
- 相关工作包：ClinkZ-WoT `docs/work-packages/WP-500-discovery.md`

## 项目资料

- ClinkZ-WoT commit: `f453f165c2ea775e5f0d10c36f1e419fcc1d79f3`
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
- 泵站、角色分工、点位映射和部署图：CONCEPTUAL COMPOSITE，用于说明常见工程协作，不声称对应某个具体项目。
- ClinkZ-WoT td crate 的 TD/TM 数据结构、序列化、校验与扩展保留：IMPLEMENTED。
- Discovery 结果进入统一 consume admission、Servient 不隐式创建 Directory、Directory client-only 边界：ACCEPTED_DESIGN。
- WP-500 Directory and Discovery Client Runtime：PLANNED，尚未完成迁移。

发布前检查：

- [x] 已读取主项目 PROJECT_STATE.md、Primary Data Flows 与 WP-500
- [x] 已检查 W3C Architecture 1.1、TD 1.1 与 Discovery
- [x] 已记录完整 commit
- [x] 已区分 TD Server、Directory、Gateway 与 Runtime
- [x] 未把 Directory 写成设备数据总线
- [x] 未要求业务人员理解现场协议或手写完整 TD
- [x] 已区分 ClinkZ-WoT accepted design 与当前实现
- [ ] 完成作者理解校验
- [ ] 完成第二轮技术事实校验
- [ ] 压缩知乎版本篇幅和重复解释
- [ ] 回填知乎发布信息
-->
