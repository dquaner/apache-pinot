---
description: 如何在 Pinot 的分布式系统架构中计算查询。
---

# Architecture

本页介绍 Apache Pinot 设计背后的指导原则。在这里，你将学习分布式系统架构，该架构允许 Pinot 根据集群中的节点数量线性扩展查询性能。还将介绍用于在 offline (batch) 或 real-time (stream) 模式下接收 (ingest) 和查询 (query) 数据的两种不同类型的表。

> 建议首先阅读[基本概念](Concepts.md)以更好地理解本页中使用的术语。

## 设计指导原则

Pinot 是由 LinkedIn 和 Uber 的工程师设计的，可以根据集群中的节点数量来扩展查询性能。随着节点的增加，查询性能将根据预期的每秒查询量配额不断提高。为了在不降低性能的情况下实现对无限数量的节点和数据存储的水平可伸缩性，建立了以下设计原则。

* **高可用性：**Pinot 为面向客户的应用程序提供低延迟分析查询。根据设计，Pinot 不会出现单点故障。当节点关闭时，系统继续提供查询服务。
* **可横向扩展：**能够在工作负载发生变化时通过添加新节点进行扩展。
* **延迟和存储：**Pinot 即使在高吞吐量下也能提供低延迟。为此开发了分段 (segment) 分配策略、路由 (routing) 策略、星树 (star-tree) 索引等特性。
* **不可变数据：**Pinot 假设所有存储数据都是不可变的。为了符合 GDPR（通用数据保护条例），我们提供了一个附加解决方案，在保证性能的同时清除数据。
* **动态配置更改：**必须在不影响查询可用性或性能的情况下执行添加新表、扩展集群、接收 (ingesting) 数据、修改索引配置和重新平衡 (re-balancing) 等操作。

## 核心组件

正如[基本概念](Concepts.md)中所描述的，Pinot 具有多个分布式系统组件：[Controller](https://docs.pinot.apache.org/basics/components/controller)，[Broker](https://docs.pinot.apache.org/basics/components/broker)，[Server](https://docs.pinot.apache.org/basics/components/server)，和 [Minion](https://docs.pinot.apache.org/basics/components/minion)。 [Apache Helix](http://helix.apache.org/) 作为一个代理嵌入到 Pinot 的不同组件中，帮助 Pinot 进行集群管理；[Apache Zookeeper](https://zookeeper.apache.org/) 负责协调和维护整体集群状态和健康状况。

![Pinot Core Components](images/Architecture\_core-components.svg)

### Apache Helix and Zookeeper

所有的 Pinot 服务器 ([servers](https://docs.pinot.apache.org/basics/components/server)) 和代理 ([brokers](https://docs.pinot.apache.org/basics/components/broker)) 都由 Helix 管理。Helix 是一个通用的集群管理框架，用于管理分布式系统中的分区 (partitions) 和副本 (replicas) 。可以将 Helix 看作是一个事件驱动的发现服务，它可以推送和拉取通知，可以将集群的状态驱动到理想的配置。一个维护着有状态操作 (stateful operations) 合约的有限状态机 (finite-state machine) ，将集群的健康状况驱动到最佳配置。随着 Helix 根据数据在集群中的存储位置更新节点之间的路由配置，查询负载也得到了优化。

Helix 根据节点的职责将其划分为三个不同的逻辑组件：

1. **参与者 (Participant) :** 集群中实际承载分布式存储资源的节点。
2. **观察者 (Spectator) :** 观察每个参与者的当前状态，并相应地路由请求。例如，路由器 (routers) 需要知道分区所在的实例及其状态，以便将请求路由到适当的端点。随着存储原语 (primitives) 的添加和更改，不断更改路由以优化集群性能。
3. **控制器 (Controller) :** 观察和管理参与者的状态。控制器负责协调集群中的所有状态转换，并确保在保持集群稳定性的同时满足状态约束 (constraints) 。

Helix 使用 Zookeeper 维护集群状态。Pinot 集群中的每个组件都以一个 Zookeeper 地址作为启动参数。分布在 Pinot 集群中的各种组件将监听 Zookeeper 通知，并通过其嵌入的 Helix-defined 的代理发布更新。

| 组件         | Helix 映射                                                                                                      |
| ---------- | ------------------------------------------------------------------------------------------------------------- |
| Segment    | 建模为 **Helix Partition** 。每个分段 (segment) 可以有多个副本，称为 **Replicas** 。                                             |
| Table      | 建模为 **Helix Resource** 。多个分段 (segments) 被分组到一个表 (table) 中，同一个表中的所有分段 (segments) 都有相同的模式 (schema) 。            |
| Controller | 嵌入驱动集群整体状态的 Helix 代理。                                                                                         |
| Server     | 建模为 **Helix Participant** 并存储分段 (segments) 。                                                                  |
| Broker     | 建模为 **Helix Spectator** ，观察集群中分段 (segments) 和服务器 (servers) 的状态变化。为了支持多租户，Broker 还被建模为 **Helix Participant** 。 |
| Minion     | 建模为 **Helix Participant** 。                                                                                   |

Helix 代理使用 Zookeeper 存储和更新配置，以及协调分布式集群。Zookeeper 存储的集群信息如下：

| 资源              | 存储的属性                                                                                                                                                                                         |
| --------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Controller      | <ul><li>被分配为当前领导者的控制器</li></ul>                                                                                                                                                               |
| Servers/Brokers | <ul><li>servers/brokers 列表以及它们的配置</li></ul><ul><li>Health status</li></ul>                                                                                                                    |
| Tables          | <ul><li>List of tables</li></ul><ul><li>Table configurations</li></ul><ul><li>Table schema information</li></ul><ul><li>List of segments within a table</li></ul>                             |
| Segment         | <ul><li>Exact server location(s) of a segment (routing table)</li></ul><ul><li>State of each segment (online/offline/error/consuming)</li></ul><ul><li>Meta data about each segment</li></ul> |

了解集群中 Helix 代理的 Zookeeper 中的 `ZNode` 布局结构有助于集群状态和运行状况的相关操作和故障排除。

![Pinot's Zookeeper Browser UI](images/Architecture\_zookeeper-ui.png)

### Controller

Pinot 控制器 (controller) 充当集群整体状态和运行状况的驱动。由于它作为 Helix 参与者和观察者的角色来驱动其他组件的状态，它通常是 Zookeeper 之后启动的第一个组件。启动控制器需要准备两个参数：Zookeeper 地址和集群名称。如果集群还不存在，控制器将通过 Helix 自动创建一个集群。

#### 容错

为了实现容错，可以启动多个控制器（通常是三个），其中一个控制器将充当领导者 (leader) 。如果领导者崩溃或死亡，则自动选出另一个领导者。领导者选举由 Apache Helix 负责。在集群上执行 DDL 等效操作（如添加表或分段）至少需要一个控制器。

控制器不会干扰查询执行。即使所有控制器节点离线，查询执行也不受影响。如果所有控制节点都离线，集群的状态将保持在上一个领导者下线时的状态。当新的领导者上线时，集群会恢复重新平衡活动，并可以接受新的表或分段。

#### 控制器 REST 接口

控制器提供 REST 接口，以便在所有存储资源 (servers、brokers、tables 和 segments )上执行 CRUD 操作。

> 有关网站管理工具的更多信息，请参阅 [Pinot 数据资源管理器](https://docs.pinot.apache.org/basics/components/exploring-pinot)。

### Broker

代理 (Broker) 的职责是将给定的查询路由到适当的服务器实例。代理将收集所有服务器的响应并将其合并为最终结果，并发送回请求客户端。代理提供接受 SQL 查询的 HTTP 端点，并以 JSON 格式返回响应。

代理启动需要准备三个关键参数：

* Cluster name
* Zookeeper address
* Broker instance name

启动时，代理 (broker) 注册为 Helix Participant ，并等待其他 Helix 代理 (agents) 的通知。这些通知将要求处理表的创建，新分段的加载，服务器的启动/关闭，以及任何配置的更改。

#### 服务发现/路由表

不管哪种通知，代理 (broker) 的主要职责是维护查询路由表。查询路由表是分段和分段所在服务器之间的映射。通常，一个分段可能存储在多个服务器上。代理根据为一个表配置的路由策略计算多个路由表。默认策略是在所有可用服务器上平衡查询负载。

> 还有一些高级路由策略可用，比如 ReplicaAware 路由、基于分区的路由和最小服务器选择路由。这些策略适用于特殊或通用的情况，用于提供非常高吞吐量的查询。

```json5
// 这是一个在 Helix 中为 EXTERNAL VIEW 配置 ZNode 的例子
{
  "id" : "baseballStats_OFFLINE",
  "simpleFields" : {
    ...
  },
  "mapFields" : {
    "baseballStats_OFFLINE_0" : {
      "Server_10.1.10.82_7000" : "ONLINE"
    }
  },
  ...
}
```

#### 查询处理

对于每个查询，一个集群 (cluster) 的代理 (broker) 执行以下操作:

* 获取根据表配置中定义的路由策略为查询计算的路由。
* 计算每个服务器上要查询的分段列表。要了解更多信息，请查看[路由](https://github.com/pinot-contrib/pinot-docs/blob/latest/basics/operators/tuning/routing.md)。
* Scatter-Gather: 将请求发送到每个服务器并收集响应。
* Merge: 合并从每个服务器返回的查询结果。
* 将查询结果发送给客户端。

```json5
// Query: select count(*) from baseballStats limit 10

// RESPONSE
// ========
{
    "resultTable": {
        "dataSchema": {
            "columnDataTypes": ["LONG"],
            "columnNames": ["count(*)"]
        },
        "rows": [
            [97889]
        ]
    },
    "exceptions": [],
    "numServersQueried": 1,
    "numServersResponded": 1,
    "numSegmentsQueried": 1,
    "numSegmentsProcessed": 1,
    "numSegmentsMatched": 1,
    "numConsumingSegmentsQueried": 0,
    "numDocsScanned": 97889,
    "numEntriesScannedInFilter": 0,
    "numEntriesScannedPostFilter": 0,
    "numGroupsLimitReached": false,
    "totalDocs": 97889,
    "timeUsedMs": 5,
    "segmentStatistics": [],
    "traceInfo": {},
    "minConsumingFreshnessTimeMs": 0
}
```

#### 容错

代理 (Broker) 实例可以水平扩展，没有上限。但在大多数情况下，只需要三个代理。如果返回给客户端的大多数查询结果小于 1MB ，则可以在同一个实例容器中运行代理和服务器。对于不需要在生产环境中保证严格的 SLA 查询性能的用例，这降低了集群部署的总体占用空间。

### Server

服务器管理分段，并在查询处理期间完成大部分繁重的工作。尽管体系结构中显示有两种服务器，实时 (real-time) 服务器和离线 (offline) 服务器，但服务器并不真正知道它将成为实时服务器还是离线服务器。服务器的职责取决于表分配策略。

> 理论上，一台服务器可以同时管理实时段和离线段。但在实践中，我们为实时服务器和离线服务器使用不同类型的机器 SKUs 。将实时服务器和离线服务器分开的好处是允许各自独立地扩展。

#### Offline servers

离线服务器通常管理不可变 (immutable) 的分段。在这种情况下，分段是在集群外部创建的，并通过 shell-based [curl](https://curl.haxx.se/) 请求上传。根据主从复制因素 (replication factor) 和分段分配策略，控制器选择一台或多台服务器来托管分段。服务器通过 Helix 得到关于新分段的通知，服务器在准备好为查询服务之前，从深层存储中获取分段并加载它们。这时，集群的代理 (broker) 会检测到有新的分段可用，并开始在查询响应中包含它们。

#### Real-time servers

实时服务器与离线服务器不同，实时服务器节点从流数据源，如 Kafka，中摄取 (ingest) 数据，并在内存中生成带索引的分段，然后定期将分段刷新到磁盘。在内存中，分段 (segments) 也称为消耗分段 (consuming segments) ；根据完成阈值（基于行数、时间或分段大小）定期刷新这些消耗分段，此时，它们被称为已完成分段 (completed segments) ，已完成分段类似于离线服务器上的分段。查询会去遍历 in-flight (consuming) segments 和 completed segments 。

### Minion

Minion 是一个可选组件，在刚开始使用 Pinot 的阶段不需要它。Minion 用于从 Pinot 集群中清除数据（出于英国的 GDPR 合规等原因）。

## 数据接收 (ingestion) 概述

在 Pinot 中，逻辑表被建模为两种类型的物理表之一：离线或实时。之所以有两种类型的表，是因为每种表都遵循一个不同的状态模型。

实时表和离线表为索引提供了不同的配置选项；实时表还为流数据源（如 Kafka）提供了连接器属性。表类型还允许用户为实时和离线服务器节点使用不同的容器。例如，离线服务器倾向于使用具有更大存储容量的虚拟机，而实时服务器可能需要更高的系统内存和更多的 CPU 内核。

两种表的扩展方式也不同：

* 实时表具有较小的保留期 (retention period) ，并根据摄取速率扩展查询性能。
* 离线表具有更大的保留期，并基于存储数据的规模大小扩展性能。

在为工作负载配置不同类型的表时，有几件事需要注意：从同一个源接收数据时，可以有两个表，它们接收相同的数据，但对实时查询和离线查询进行了不同的配置；即使这两个表有相同的数据，根据你的需求，查询的性能也会有所不同；在这样的场景中，实时表和离线表必须共享相同的模式 (schema) 。

> 可以根据使用需求对实时表和离线表进行不同的配置。例如，你可以选择为离线表启用星树索引，而具有相同模式的实时表可能不需要它。

### Batch data flow

![Batch data flow](images/Architecture\_batch-data-flow.jpeg)

在批处理模式 (batch mode) 下，数据通过接收作业 ([ingestion job](https://docs.pinot.apache.org/basics/data-import/batch-ingestion)) 被导入到 Pinot 中。接收作业将原始数据源（如 CSV 文件）转换为分段。一旦为导入的数据生成了分段，接收作业将它们存储到集群的分段存储区（也称为深度存储区），并通知控制器。收到通知后，控制器上的 Helix 代理会更新 Zookeeper 中的理想状态配置。然后，Helix 将通知离线服务器有新的可用分段。为了响应来自控制器的通知，离线服务器直接从集群的分段存储中下载新创建的分段。集群的代理 (broker) 监听 Helix 中的状态变化，检测新分段并将其添加到要查询的分段列表（segment-to-server 路由表）。

### Real-time data flow

在创建表时，控制器在 Zookeeper 中为正在消费的分段创建一个新条目。Helix 注意到新的分段并通知实时服务器，实时服务器开始使用来自流源的数据。代理 (broker) 监听变化，检测新分段并将其添加到要查询的分段列表（segment-to-server 路由表）。

![Real-time data flow](images/Architecture\_real-time-data-flow.svg)

当分段完成(即满)时，实时服务器通知控制器，控制器检查所有副本并选择一个胜出者提交分段。胜出者提交该分段并将其上传到集群的分段存储中，将该分段的状态从“正在使用”更新为“在线”。然后控制器准备一个处于“消费”状态的新段。

## 查询概述

查询由代理接收，它根据分段到服务器的路由表检查请求，将请求分散到实时服务器和离线服务器之间。

![Real-time data flow](images/Architecture\_query-overview.svg)

然后，这两个表通过过滤和聚合查询到的数据来处理请求，然后将这些数据返回给代理。最后，代理将查询响应的所有部分聚集在一起，并将结果返回给客户端。
