---
id: "WOT-001"
title: "重新理解 WoT | W3C WoT 到底解决什么问题？"
subtitle: "协议碎片化通常源于真实系统的长期演进，WoT 在其上建立统一的 Thing 接口"
series: "重新理解 WoT"
series_order: 1
status: "DRAFTING"
author: "yushun1990"
created: "2026-07-28"
updated: "2026-07-29"
summary: "真实的物联网系统往往同时包含存量设备、现场协议、消息系统、厂商平台和 Web API。W3C Web of Things 不要求它们改用同一种通信协议，而是用 Thing Description 描述共同的能力和交互方式，再由 Form 与 Protocol Binding 将 Property、Action 和 Event 映射到具体接口。"

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
  previous: "00-column-guide.md"
  next: "WOT-002"
  articles:
    - "WOT-002"
    - "WOT-003"
  docs:
    - "docs/architecture/10-primary-data-flows.md"
  source: []
  tests: []
---

# 重新理解 WoT | W3C WoT 到底解决什么问题？

> 本文基于 ClinkZ-WoT commit `f453f165c2ea775e5f0d10c36f1e419fcc1d79f3`。
>
> ClinkZ-WoT 仍处于架构闭合和分阶段实现过程中。本文首先解释 W3C Web of Things 的问题空间，再说明 ClinkZ-WoT 为什么选择从 Thing Description 开始构建运行时；文中的完整执行链并不代表当前已经全部实现。

真实的物联网系统通常不是在同一天、由同一支团队、围绕同一种协议设计出来的。

一套已经运行多年的工厂、园区或供水系统，可能经历过多轮建设：

- 早期控制器和仪表仍在稳定运行，通过现场协议或工业网关接入；
- 后来增加的无线传感器，由厂商网关汇聚并向平台发布消息；
- 某些成套设备只能通过厂商提供的管理平台或 HTTPS API 操作；
- 新建设备可能使用另一套消息总线，边缘服务和云端服务又有自己的接口；
- 业务系统最终需要把这些来源组合成监控、告警、联动和运维流程。

没有人会为了“架构统一”轻易替换仍可使用的控制器，也没有一种通信方式适合低功耗传感器、实时控制、局域网设备和跨公网服务等所有环境。

因此，多协议共存并不一定是设计失败。它往往是设备寿命、网络条件、成本、供应商边界和系统长期演进共同产生的结果。

W3C WoT Architecture 1.1 对这个前提说得很直接：IoT 使用多种设备访问协议，因为没有一种协议适合所有上下文。W3C 在 2026 年发布的 WoT Use Cases and Requirements 中，也专门收录了工业跨协议交互、存量设备、跨厂商集成和协议抽象等场景。

所以，介绍 WoT 不需要先虚构“一台空调用 HTTP、一台传感器用 MQTT”。现实问题本来就存在。真正需要追问的是：

> 当系统中的设备和服务已经能够通信，为什么上层应用仍然越来越难维护？

## 多协议共存为什么会成为长期状态

协议碎片化通常来自四类现实约束。

### 设备的生命周期远长于软件系统

工业控制器、楼宇设备、仪表和网关可能连续运行十几年。新的平台上线时，现场设备不可能全部替换。

这类设备可能通过 Modbus、BACnet、OPC UA 或厂商私有协议工作。为了让现代应用访问它们，系统通常在网关、边缘服务或代理层完成协议转换。

### 不同场景需要不同通信特性

设备侧可能重视低功耗、弱网络和小报文；控制网络重视确定性和本地自治；云端系统重视可扩展接入；第三方服务则更适合通过 Web API 集成。

选择 MQTT、HTTP、CoAP、OPC UA、BACnet、Modbus 或其他协议，往往是在解决不同约束，而不是团队对“最佳协议”没有共识。

### 采购和厂商生态会形成边界

成套设备经常带有自己的控制器、网关、数据格式和管理平台。系统集成方能够获得的接口，可能是一个寄存器表、一组 OPC UA Node、若干 MQTT Topic，也可能只是一份 REST API 文档。

业务系统必须面对厂商实际提供的接口，而不是理想中的统一协议。

### 系统会不断增加新的应用

最初系统可能只需要采集数据，后来又增加告警、远程控制、数字孪生、移动端、AI 分析和跨系统联动。

同一批设备会被越来越多的应用使用。如果每个应用都重新理解底层接口，接入复杂度就会随着应用数量再次增长。

## 真正让应用痛苦的，不只是协议名称

假设业务系统需要操作一个泵站。它关心的是：

```text
读取出口压力
读取运行状态
启动水泵
订阅过载告警
```

但这些能力可能散落在不同接口中：

```text
出口压力
  -> 由边缘网关读取现场寄存器后提供

运行状态
  -> 出现在某个 MQTT Topic 的厂商 Payload 中

启动水泵
  -> 只能调用设备管理服务的 HTTPS API

过载告警
  -> 由另一个告警服务推送
```

这里并不是说某种设备“天生应该”使用某种协议。重点在于，业务应用看到的不是四项稳定能力，而是四套不同的访问约定：

- 地址或资源如何定位；
- 请求如何构造；
- 数据如何编码；
- 身份认证如何完成；
- 错误如何表达；
- 订阅如何建立和取消；
- 厂商字段究竟代表什么。

如果直接集成，协议和厂商细节会逐渐进入业务代码：

```text
监控页面知道 Topic 和 JSON 字段
规则引擎知道寄存器地址
运维服务知道厂商 URL 和鉴权方式
告警流程知道某种回调 Payload
```

系统当然仍然能够工作，但“设备能力”没有形成独立、可复用的接口模型。更换设备、增加供应商或接入新应用时，多个模块都可能需要修改。

这才是 WoT 试图处理的问题：**通信已经打通，但应用仍缺少共同、机器可读的 Thing 接口。**

## 直觉方案：把所有接口都转成 MQTT 或 HTTP

一个自然的解决办法，是在网关或平台入口统一协议。

例如：

```text
Modbus / BACnet / OPC UA / vendor protocol
                    |
                    v
                  Gateway
                    |
                    v
              MQTT or HTTP API
```

这一步很有价值。它可以减少连接方式，统一认证入口，也便于集中运维。

但统一传输不等于统一接口。

即使所有数据都进入 MQTT，不同设备仍可能拥有完全不同的：

- Topic 结构；
- Payload Schema；
- 设备身份表达；
- 命令与响应关联方式；
- 错误编码；
- 在线状态和重试语义。

即使全部改成 HTTP，不同厂商仍会提供不同的 URL、method、字段、状态码和异步任务模型。

因此，统一协议最多解决了“消息怎样到达”的一部分问题。它并没有自动回答：

- 这个对象是什么；
- 它有哪些状态；
- 哪些状态可读、可写或可观察；
- 可以触发哪些操作；
- 会产生哪些事件；
- 输入和输出满足什么约束；
- 同一能力可以通过哪些接口访问。

对于范围固定、设备单一、只服务一个应用的系统，平台内部自定义一套统一 API 往往已经足够。WoT 并不是每个项目都必须增加的一层。

当系统需要跨设备、跨协议、跨厂商并长期演进时，才更需要把“Thing 的能力”从具体传输接口中分离出来。

## WoT 不是另一种通信协议

W3C Web of Things（WoT）不要求现场设备改用一种名为“WoT”的新协议。

它位于现有协议之上，试图建立一套共同的描述和交互模型。

| 层次 | 主要问题 | 典型内容 |
|---|---|---|
| 领域与能力 | 这个对象是什么，能做什么 | 压力、开关、启动、告警 |
| 交互模型 | 应用可以怎样操作它 | Property、Action、Event |
| 接口描述 | 数据、安全和访问入口是什么 | Data Schema、Security、Form |
| 协议通信 | 请求和消息怎样传输 | HTTP、MQTT、CoAP、Modbus、OPC UA、Zenoh |
| 运行时执行 | 如何选择、校验、调用和清理 | planning、Binding、lifecycle |

下层协议负责把数据或请求送到另一端。

WoT 负责描述：

- 这是一个什么 Thing；
- 它暴露哪些状态；
- 可以触发哪些操作；
- 会产生哪些通知；
- 输入和输出数据满足什么约束；
- 需要什么安全配置；
- 可以通过哪些网络接口完成交互。

两者的关系更接近：

```text
Thing 的能力与交互语义
          |
          v
Thing Description
          |
          v
Form + Protocol Binding
          |
          v
HTTP / MQTT / CoAP / Modbus / OPC UA / Zenoh / ...
```

因此，WoT 与 MQTT 不是竞争者，WoT 与 HTTP 也不是竞争者。

WoT 希望避免的是：上层应用把某个 Topic、寄存器、URL 或厂商 Payload 当成 Thing 本身。

## Thing Description：把能力和访问方式写进同一份契约

Thing Description（TD）是 WoT 的中心构件。它使用机器可读的格式描述 Thing 的元数据和面向网络的接口。

一个 TD 通常包含：

```text
Thing metadata
  |
  +-- Properties
  +-- Actions
  +-- Events
  +-- Data Schemas
  +-- Security Definitions
  +-- Forms
  +-- Links and semantic annotations
```

仍以泵站为例，应用真正需要的接口可以被表达为：

```text
Property: outletPressure
Property: running
Action: start
Event: overloadAlarm
```

TD 进一步说明这些交互的数据类型、安全要求和访问入口。

一段只用于说明层次的概念示意如下，它不是完整的生产 TD：

```json
{
  "title": "Pumping Station 17",
  "properties": {
    "outletPressure": {
      "type": "number",
      "unit": "kPa",
      "readOnly": true,
      "forms": [
        {
          "href": "https://edge.example.com/stations/17/pressure",
          "op": ["readproperty"]
        }
      ]
    }
  },
  "actions": {
    "start": {
      "input": {
        "type": "object"
      },
      "forms": [
        {
          "href": "https://vendor.example.com/stations/17/start",
          "op": ["invokeaction"]
        }
      ]
    }
  }
}
```

这里最重要的不是 JSON 语法，而是依赖方向：

```text
应用要读取 outletPressure
          |
          v
找到 outletPressure Property
          |
          v
选择支持 readproperty 的 Form
          |
          v
由对应 Protocol Binding 执行
```

应用表达的是“读取出口压力”，而不是“调用这个 URL”或“读取某个寄存器”。

具体地址和协议仍然存在，只是被放到了明确的接口描述和 Binding 边界中。

## 存量设备不必原生支持 TD

看到 TD 的 JSON 表达后，很容易产生一个误解：是不是所有现场设备都要升级固件，自己提供 TD？

不是。

W3C WoT Architecture 明确考虑了 brownfield device、受限设备、休眠设备和专用协议设备。TD 可以由单独的服务提供，网关或代理也可以把传统设备暴露为一个 Web Thing。

因此，现实中的接入链路可能是：

```text
传统设备
  -> 现场协议
  -> 网关或代理
  -> TD + 网络接口
  -> WoT Consumer
```

WoT 不是假装底层差异不存在，而是在适合的位置建立一个可供应用消费的统一接口。

## Property、Action 和 Event 描述什么

W3C WoT 使用三类 Interaction Affordance 描述 Consumer 可以怎样与 Thing 交互。

### Property：Thing 的状态

Property 表达可以读取、写入或观察的状态，例如：

- 当前压力；
- 电机转速；
- 门锁状态；
- 目标温度；
- 设备运行模式。

Property 不等于某一条遥测消息。它强调的是状态及其读、写、观察能力。

它可以由 HTTP GET、CoAP 请求、Modbus 读取、MQTT 消息或 Zenoh Query 等不同机制实现。

### Action：触发一个操作

Action 表达调用者可以触发的功能或过程，例如：

- 启动水泵；
- 重启控制器；
- 执行校准；
- 开始固件升级；
- 生成诊断报告。

Action 可以定义输入与输出。它表达“调用某项能力”，而不是绑定到固定的 HTTP method、Topic 或寄存器写入。

### Event：Thing 主动产生的通知

Event 表达 Thing 产生的异步通知源，例如：

- 过热告警；
- 电机堵转；
- 门被打开；
- 作业完成；
- 连接状态异常。

Event 的传输可以使用 MQTT Subscription、Server-Sent Events、WebSocket、CoAP Observe、Zenoh Subscriber 或其他机制。

WoT 关心的是通知在 Thing 接口中代表什么，Protocol Binding 再处理它如何传输。

## Protocol Binding：把交互意图变成协议行为

Protocol Binding 负责把 Interaction Affordance 映射到特定协议或 IoT 生态的消息。

假设应用执行：

```text
read Property "outletPressure"
```

不同 Binding 可能产生完全不同的行为：

```text
HTTP Binding
  -> GET https://edge.example.com/stations/17/pressure

Modbus Binding
  -> 读取指定 unit 和 register，再按数据模型解码

MQTT Binding
  -> 根据约定发布请求、订阅响应并进行 correlation

Zenoh Binding
  -> 对某个 key expression 发起 query
```

WoT 不要求这些协议表现得一模一样，也不会抹平 QoS、请求响应、发布订阅、轮询和流式通信的差异。

它提供的是共同的上层交互模型，并要求具体 Binding 明确承担协议适配责任。

这意味着：

- MQTT 仍然使用 Topic、QoS 和会话机制；
- HTTP 仍然使用 URL、method、header 和 status code；
- Modbus 仍然有 unit、function 和 register；
- Zenoh 仍然有 key expression、query、put 和 subscriber；
- 应用不必让这些协议概念散落到每一份业务代码中。

## WoT 面对的是接口异构，不只是协议异构

把问题概括为“协议太多”仍然不够准确。

即使两个设备都使用 MQTT，它们也可能拥有完全不同的 Topic、Payload、命令响应和错误表达。

即使两个服务都使用 HTTP，它们也可能对同一种“启动”能力采用不同的输入、鉴权、同步性和失败语义。

反过来，同一个 Thing 的同一项能力也可能同时提供多个 Form，例如局域网内通过一种接口访问，远程运维时通过另一种接口访问。

因此，WoT 需要同时描述：

- 共同能力；
- 数据约束；
- 安全要求；
- 具体访问入口；
- 协议和生态专属元数据。

WoT 并不能让真实差异自动消失。它做的是把差异集中到可以被发现、检查、测试和替换的边界中，而不是让业务系统各自维护一套隐含知识。

## Thing 不只代表一块硬件

WoT 中的 Thing 不是“联网硬件”的同义词。

Thing 可以是：

- 真实传感器；
- 工业控制器；
- 网关代理的传统设备；
- 数字孪生；
- 聚合多个设备能力的虚拟对象；
- 通过网络接口提供交互的软件服务。

前文中的“泵站”就可以是一个聚合 Thing。它的压力来自仪表，运行状态来自控制器，启动操作由控制服务完成，告警则来自另一个监测模块。

上层应用不一定需要知道这些能力由几个物理设备和服务共同提供。它消费的是一个具有稳定接口的 Thing。

这也解释了为什么 WoT 关注的不只是“设备接入”，而是物理或虚拟对象的网络接口。

## WoT 不是什么

理解 WoT 的边界同样重要。

### WoT 不是统一传输协议

WoT 不要求设备把现有协议替换成“WoT 协议”。它保留并补充已有标准和解决方案。

### WoT 不是完整物联网平台

TD、Interaction Model 和 Protocol Binding 不会自动提供：

- 设备资产管理；
- 时序数据存储；
- 规则引擎；
- 告警中心；
- 多租户和计费；
- OTA；
- 可视化组态。

这些能力可以围绕 WoT 模型构建，但不都属于 WoT Runtime。

### WoT 不是自动兼容设备的魔法层

如果设备接口文档不可靠、数据含义不清楚，或者多个接口的行为根本不等价，增加一份 TD 并不能修复事实本身。

TD 必须准确描述真实能力，Binding 也必须正确实现协议语义。错误模型只会把错误变成机器可读格式。

### WoT 不保证应用完全忽略协议差异

实时性、离线行为、QoS、轮询频率、流控和安全机制在某些应用中仍然重要。

协议中立不等于协议差异被删除，而是差异通过明确的能力、策略和 Binding 边界暴露，不再无控制地渗入所有业务模块。

## 只有 TD 还不够，为什么需要 WoT Runtime

TD 是描述，不会自行完成一次属性读取。

一个可执行的 WoT Runtime 至少需要处理：

- 解析并校验 TD；
- 识别 Property、Action、Event 和操作类型；
- 解析安全要求和 Data Schema；
- 判断哪些 Form 可以使用；
- 选择对应 Protocol Binding；
- 构造并执行协议请求；
- 校验和转换返回结果；
- 管理调用、订阅、取消、资源和清理生命周期。

因此，WoT 的工程难点不只是“生成一份 JSON”。

真正的问题是：如何把机器可读的描述稳定地变成可执行交互，并在失败、取消、更新和资源受限时保持清楚的责任边界。

这也是 ClinkZ-WoT 关注的核心。

## ClinkZ-WoT 为什么从 TD 开始

ClinkZ-WoT 的目标，是实现一个协议中立的 Rust WoT Runtime。

主项目当前的 active design revision 是 **v5.0 bounded-core authority**。其规范中的主数据流可以概括为：

```text
TD document or ProducedThing draft
        |
        v
parse + preserve extensions + validate
        |
        v
capture immutable planning context
        |
        v
shared planner
        |
        +----> logical plans
        |
        +----> binding compiler extensions
                       |
                       v
            admitted immutable plan set
                       |
                       v
              runtime interaction
```

这条链路试图保持两个边界。

第一，运行时共同层处理协议中立事实：

- 操作哪个 Thing；
- 目标是哪个 Property、Action 或 Event；
- 执行 read、write、invoke、observe 还是 subscribe；
- 使用什么 Schema、安全分支和候选 Form；
- 谁拥有计划、生命周期和清理责任。

第二，具体 Binding 拥有协议专属知识：

- HTTP method、URL、header 与状态映射；
- MQTT Topic、QoS、correlation 和会话行为；
- Zenoh key expression、query、subscriber 和编码设置；
- 其他协议自己的通信与流控机制。

主项目还规定：一旦某个决定已经进入不可变计划，执行阶段不应回到完整 TD 中重新发现同一个决定。

这样做不仅考虑热路径开销，更重要的是让选择结果、所有权、资源预算和错误边界可以被验证。

必须说明当前状态：截至本文基线，v5 authority 已经激活，但 M1 与 M2 仍在进行中，M3 Planning and Compilation Pipeline 尚未进入产品源码实现。WP-100 的部分 Foundation 与 Core 工作已经完成；计划中的 `clinkz-wot-planning` crate、Logical Plan、Binding Artifact、Binding Compiler 和完整 Property Read 计划链仍未落入产品 Rust 代码，现有 Protocol Binding 路径仍保留旧的直接执行边界。

因此，这里描述的是 v5 已接受的目标架构，不是完整 WoT Runtime 已经发布的声明。

## 这如何落到 Rust 边界

下面不是 ClinkZ-WoT 当前公开 API，而是概念上的模块责任：

```text
Application
  -> ReadProperty(ThingId, PropertyName)

Core / Planning
  -> validate interaction
  -> select admitted candidate
  -> produce protocol-neutral execution facts

Protocol Binding
  -> own protocol artifact
  -> execute protocol I/O
  -> return transport result

Servient
  -> own orchestration, lifecycle and cleanup
```

Rust 类型边界需要防止职责重新混在一起：

- 协议中立请求不应携带具体 MQTT、HTTP 或 Zenoh 客户端对象；
- Binding Artifact 应由对应 Binding 拥有，Core 不应猜测其内部结构；
- Servient 应拥有运行时编排权，而不是让 Binding 在隐藏任务中直接调用应用 Handler；
- 调用和订阅需要明确的 owned value、generation、取消和终结清理路径；
- Host 与 `no_std + alloc` 可以拥有不同推进机制，但应保持相同的交互语义。

语义层没有消除复杂度。它把原本散落在接入代码和业务代码中的复杂度，集中到了可以被检查、测试和治理的边界中。

## 采用 WoT 需要付出什么

WoT 并不是免费抽象。

首先，团队必须认真建模。Property、Action 和 Event 的划分如果不准确，运行时再完善也无法弥补语义错误。

其次，每种协议或生态仍然需要可靠的 Protocol Binding。Binding 不只是更换地址格式，它还要处理协议消息、序列化、安全、关联、订阅、轮询、流控和错误映射。

再次，运行时必须处理选择与生命周期。当一个 Interaction Affordance 存在多个 Form 时，哪个可用、哪个安全、是否允许回退、失败后谁清理，都需要明确规则。

最后，并非所有项目都需要完整 WoT Runtime。一个范围固定、接口稳定、只服务单一应用的系统，直接使用协议 API 或内部适配层可能更简单。

WoT 的价值主要出现在这些条件逐渐出现时：

- 设备来自多个时期和厂商；
- 系统需要同时使用多种协议或平台；
- 同一批能力会被多个应用长期复用；
- 底层设备和接口会持续替换；
- 团队希望把业务语义与接入细节分离。

## 总结

1. 多协议共存不是为了说明 WoT 而虚构的场景，而是设备寿命、场景约束、厂商生态和系统演进的自然结果。
2. 统一 MQTT 或 HTTP 可以统一部分传输方式，但不会自动统一设备能力、数据含义和交互语义。
3. W3C WoT 使用 TD 描述 Thing 的 Property、Action、Event、Schema、安全和 Form，再由 Protocol Binding 把交互映射到真实协议；WoT Runtime 则负责把描述变成可执行且可治理的调用。

理解这一点之后，才能继续讨论下一层问题：Thing Description 究竟应该描述什么，又为什么不能把它写成一份运行时配置文件。

## 延伸阅读

- 上一篇：[专栏导读：从零开发一个 Rust WoT 引擎](../00-column-guide.md)
- 下一篇：重新理解 WoT | Thing Description 是语义契约，而不是设备配置文件（计划中）
- 相关项目规范：ClinkZ-WoT `docs/architecture/10-primary-data-flows.md`
- 相关源码与测试：本文不对尚未完成的完整执行链作实现声明

## 项目资料

- ClinkZ-WoT commit: `f453f165c2ea775e5f0d10c36f1e419fcc1d79f3`
- `README.md`
- `PROJECT_STATE.md`
- `PLAN.md`
- `docs/architecture/10-primary-data-flows.md`

## 外部资料

- [W3C Web of Things (WoT) Architecture 1.1](https://www.w3.org/TR/wot-architecture11/)
- [W3C Web of Things (WoT) Thing Description 1.1](https://www.w3.org/TR/wot-thing-description11/)
- [W3C Web of Things (WoT) Binding Templates](https://www.w3.org/TR/wot-binding-templates/)
- [W3C Web of Things (WoT) Discovery](https://www.w3.org/TR/wot-discovery/)
- [W3C Web of Things (WoT): Use Cases and Requirements, 2026-02-05](https://www.w3.org/TR/2026/NOTE-wot-usecases-20260205/)

<!--
内部事实分类：

- 多协议共存、brownfield、跨协议交互和协议抽象：外部标准与 W3C use case 支持的现实问题。
- 开篇泵站场景：用于组合说明真实系统约束的 CONCEPTUAL COMPOSITE，不声称对应某个具体项目。
- W3C WoT 的 Thing、TD、Interaction Affordance 和 Protocol Binding 定义：外部标准事实。
- ClinkZ-WoT 的协议中立、Servient 编排、不可变 Planning Context、Logical Plan、Binding Artifact 和 Compiled Plan Set：ACCEPTED_DESIGN。
- v5.0 bounded-core authority、M1/M2/M3 状态和当前阻塞：CURRENT PROJECT STATE。
- WP-100 部分 Foundation/Core 工作：IMPLEMENTED；Planning、Binding Compiler/Artifact 与完整 Property Read 计划链：PLANNED / NOT YET IMPLEMENTED。
- 文中 TD、接口和模块流程：CONCEPTUAL ILLUSTRATION。

发布前检查：

- [x] 已读取主项目 AGENTS.md、PROJECT_STATE.md、PLAN.md
- [x] 已检查主题对应的正式规范
- [x] 已记录完整 commit
- [x] 已区分 implemented / accepted / planned
- [x] 未把 MQTT 或其他协议描述为 WoT 的竞争者
- [x] 未虚构公开 Rust API、性能或规模结论
- [x] 已用 W3C use case 验证现实中的多协议与跨协议问题
- [ ] 完成作者理解校验
- [ ] 完成第二轮技术事实校验
- [ ] 回填知乎发布信息
-->
