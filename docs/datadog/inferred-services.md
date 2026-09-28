# Datadog 推断服务（Inferred Services）调研

> 调研日期：2026-09-28  
> 范围：Datadog APM **Inferred Services（推断服务）**，重点关注与 OpenTelemetry 的互操作关系。

## 1. 摘要

Datadog 的 Inferred Services 是**根据出站 Trace 数据推导出的远端依赖实体**。它不是一个已经独立完成埋点的服务，也不是通过机器学习模型“猜测”出来的服务。

可以用下面的模型理解：

~~~text
推断服务
=
出站 Span
+ 语义属性
+ Peer 身份解析
+ 实体分类
+ APM 统计聚合
~~~

一个已经接入 tracing 的服务可能调用 PostgreSQL、Redis、Kafka、云服务或第三方 HTTP API，而这些依赖本身并不一定产生 APM Span。只要调用方能够生成包含足够语义信息的 CLIENT 或 PRODUCER Span，Datadog 就可以从这些 Span 中推导出下游依赖，并在依赖拓扑、Trace 视图和相关 APM 数据中展示这些实体。

因此，Inferred Services 位于以下几个概念的交汇处：

- Span 语义；
- OpenTelemetry Semantic Conventions；
- Datadog peer service 聚合；
- 服务与依赖拓扑；
- APM Trace Metrics；
- Datadog 的实体模型。

---

## 2. 它解决什么问题

假设只有一个已经埋点的业务服务：

~~~mermaid
flowchart LR
    U[用户] --> C[checkout-service]
    C --> P[(PostgreSQL)]
    C --> R[(Redis)]
    C --> S[第三方 API]
    C --> K[Kafka]
~~~

可能只有 checkout-service 能够产生完整的 tracing 数据。

如果拓扑只能识别那些自己产生 SERVER Span 的服务，那么最终得到的服务关系会遗漏大量真实依赖。

调用方的出站 Span 本身已经携带了足够多的信息，例如：

~~~text
service.name = checkout-service
span.kind = CLIENT
db.system = postgresql
db.namespace = orders
server.address = postgres-prod.internal
duration = 42ms
~~~

即使 PostgreSQL 本身没有产生任何 APM Service Span，Datadog 仍然可以从这个 Span 中判断：

> checkout-service 正在调用一个与 PostgreSQL 相关的远端依赖。

这就是推断服务存在的基础。

---

## 3. 已埋点服务与推断服务的区别

两者不能混为一谈。

### 3.1 下游服务已经完整埋点

~~~mermaid
sequenceDiagram
    participant C as checkout-service
    participant P as payment-service

    C->>P: CLIENT Span
    activate P
    Note over P: SERVER Span + 内部 Span
    P-->>C: 响应
    deactivate P
~~~

此时 Datadog 能同时拿到调用方和被调用方产生的 telemetry。

### 3.2 下游服务没有埋点

~~~mermaid
sequenceDiagram
    participant C as checkout-service
    participant P as payment.internal

    C->>P: CLIENT Span
    Note over C,P: server.address=payment.internal
    P-->>C: 响应
    Note over C: 只有调用方观测到的数据
~~~

此时推断服务表示的是：

> **调用方能够观察到的远端依赖。**

它并不代表 Datadog 已经获得了这个远端系统内部的完整执行信息。

| 能力 | 已埋点服务 | 推断服务 |
|---|---:|---:|
| SERVER Span | 有 | 无 |
| 内部业务 Span | 可以有 | 无 |
| Runtime Telemetry | 可以有 | 无 |
| 从远端继续向下解析拓扑 | 可以 | 通常未知 |
| 调用方观察到的延迟 | 有 | 有 |
| 调用方观察到的错误 | 有 | 有 |
| 依赖节点 / 实体 | 有 | 有 |
| Request / Error / Latency 聚合 | 有 | 配置满足时有 |

因此，**推断服务不是远端服务自身的完整可观测性，而是 caller-side dependency observability。**

---

## 4. 核心处理链路

可以把 Datadog 的处理过程抽象成：

~~~mermaid
flowchart LR
    A[CLIENT / PRODUCER Span]
    B[语义属性]
    C[Peer 属性归一化]
    D[身份解析]
    E[实体分类]
    F[APM 统计]
    G[推断实体]

    A --> B --> C --> D --> E --> G
    A --> F --> G
~~~

这里最重要的一点是：

> Datadog 不只是根据 Span 在 Service Map 上画一个节点，而是在把原始 telemetry 转换成自己的依赖与实体模型。

---

## 5. 哪些 Span 最重要

推断远端依赖时，出站 Span 是主要证据来源。

OpenTelemetry 常见的 SpanKind 包括：

- CLIENT：同步远端调用，例如 HTTP、RPC、数据库访问；
- PRODUCER：向 Broker、Topic 或 Queue 发布消息；
- SERVER：处理入站请求；
- CONSUMER：消费消息；
- INTERNAL：进程内部操作。

对于推断依赖而言，CLIENT 和 PRODUCER 尤其重要，因为它们描述了离开当前服务边界的交互。

Datadog 提供了与 SpanKind 和 peer 聚合相关的配置：

~~~yaml
compute_stats_by_span_kind: true
peer_tags_aggregation: true
~~~

Datadog 文档说明，从 Datadog Agent 7.60.0 开始，这两个配置默认启用。

---

## 6. Peer 身份解析

推断服务最核心的问题是：

> **哪些 Span 属性能够标识远端依赖？**

Datadog 会将不同 tracing instrumentation 产生的属性归一化成 peer 相关身份字段，然后根据实体类型应用不同的优先级规则。

### 6.1 通用远端服务

对于普通远端服务，Datadog 文档给出的优先级为：

~~~text
peer.service
    ↓
peer.rpc.service
    ↓
peer.hostname
~~~

能够参与 peer hostname 推导的属性包括：

- net.peer.name
- hostname
- db.hostname
- network.destination.name
- grpc.host
- http.host
- http.server_name
- server.address

例如：

~~~text
service.name = checkout-service
span.kind = CLIENT
server.address = api.stripe.com
~~~

概念上可能得到：

~~~mermaid
flowchart LR
    C[checkout-service] --> S[api.stripe.com]
~~~

如果 Span 中存在更明确的逻辑 peer identity，它可以优先于 hostname 类信息。

---

## 7. 数据库依赖推断

数据库 Span 往往同时暴露多个候选身份：

~~~text
db.system = postgresql
db.namespace = orders
server.address = postgres-prod.internal
~~~

这些属性分别可以表达：

- 使用 PostgreSQL；
- 逻辑数据库或 namespace 是 orders；
- 数据库地址是 postgres-prod.internal。

Datadog 对数据库 peer identity 有专门的优先级规则，大体按照：

~~~text
数据库逻辑名称
    ↓
特定云服务 / 数据库实体名称
    ↓
hostname
    ↓
数据库系统类型
~~~

常见的 Datadog peer 属性包括：

- peer.db.name
- peer.hostname
- peer.db.system

归一化后的数据库名称可能来源于：

- db.name
- mongodb.db
- db.instance
- cassandra.keyspace
- db.namespace

因此，数据库推断并不是简单地“使用 db.system 作为名称”。

概念上可能形成：

~~~mermaid
flowchart LR
    C[checkout-service] --> D[(orders)]
    D -. 技术类型 .-> PG[PostgreSQL]
~~~

最终显示出来的实体名称取决于具体 instrumentation 产生的属性，以及 Datadog 的归一化和优先级规则。

---

## 8. Messaging / Queue 依赖推断

消息系统也存在类似的身份解析问题。

例如一个 OpenTelemetry Span：

~~~text
span.kind = PRODUCER
messaging.system = kafka
messaging.destination.name = order.created
server.address = kafka.internal
~~~

这里可能出现三个不同层次的身份：

- Kafka：技术类型；
- kafka.internal：Broker 地址；
- order.created：逻辑消息目的地。

在拥有足够属性时，Datadog 会优先考虑消息目的地，而不是只使用通用的 messaging system。

因为：

~~~text
checkout-service -> Kafka
~~~

只能说明使用了 Kafka。

而：

~~~text
checkout-service -> order.created
~~~

则表达了更有业务意义的依赖关系。

对于依赖分析，逻辑 destination 往往比底层 Broker 技术类型更有价值。

---

## 9. 为什么需要 Peer Aggregation

假设一个本地服务产生以下调用：

~~~text
checkout -> postgres
checkout -> postgres
checkout -> api.stripe.com
checkout -> api.stripe.com
checkout -> api.stripe.com
~~~

如果指标只按照本地 service 聚合：

~~~text
service=checkout
requests=5
~~~

那么依赖结构会完全丢失。

如果保留 peer 维度：

~~~text
service=checkout, peer=postgres
requests=2

service=checkout, peer=api.stripe.com
requests=3
~~~

Datadog 就可以计算依赖级别的：

- Request Rate；
- Error Rate；
- Latency。

概念上：

~~~mermaid
flowchart LR
    C[checkout-service]
    C -->|2 次请求| P[postgres]
    C -->|3 次请求| S[api.stripe.com]
~~~

这就是 peer_tags_aggregation 的核心意义：

> 如果要产生 dependency-level metrics，就必须让 peer identity 在指标聚合过程中被保留下来。

---

## 10. 与 Datadog APM Metrics 的关系

Datadog Trace Metrics 主要包括：

- 请求数量；
- 错误数量；
- 延迟 / Duration。

在 Datadog 原生 tracing 模型中，这些指标通常基于完整应用流量计算，而不依赖后续 Trace Ingestion Sampling 最终保留了多少条 Trace。

但 OpenTelemetry 如果过早采样，会改变这个前提。

### 10.1 有问题的链路

~~~mermaid
flowchart LR
    App --> SDK[OTel SDK<br/>10% Head Sampling]
    SDK --> Collector
    Collector --> DD[Datadog]
~~~

如果 SDK 已经丢弃 90% 的 Span，那么这些 telemetry 根本不会进入 Collector 和 Datadog。

Datadog 无法根据剩余 10% 的 Span 准确恢复完整流量。

### 10.2 更合理的 Collector 处理方式

~~~mermaid
flowchart LR
    App --> SDK[OTel SDK]
    SDK --> Collector
    Collector --> SM[span_metrics]
    SM --> Metrics[APM Metrics]
    Collector --> TS[tail_sampling]
    TS --> Traces[Trace Ingestion]
~~~

Datadog 明确建议：

> span_metrics connector 应该在 sampling processor 之前接收到 Trace。

这样 APM Trace Metrics 才能基于未采样的流量进行计算。

这个顺序对推断服务同样重要，因为用于构造 dependency metrics 的 peer dimensions 必须在 Span 被丢弃之前参与统计。

---

## 11. OpenTelemetry Collector 集成

对于新的 OpenTelemetry Collector 配置，Datadog 当前推荐的主要路径包括：

- 通过 OTLP HTTP 向 Datadog 导出；
- 使用 span_metrics connector 生成 APM Trace Metrics。

Datadog 推荐的 span_metrics 配置包含用于推导以下信息的 dimensions：

- Host Tags；
- Peer Services；
- Operation Names；
- Resource Names。

需要特别注意：

> 较老的 peer_tags 配置与当前推荐的 span_metrics dimensions 并不完全一致。

Datadog 也明确指出，两种配置最终产生的 inferred entities 可能不同。

因此可以提出一个很值得实验验证的假设：

~~~mermaid
flowchart TD
    A[相同的应用 Telemetry]
    A --> B[Datadog Agent]
    A --> C[Datadog Connector]
    A --> D[OTel span_metrics]

    B --> E[实体集合 A]
    C --> F[实体集合 B]
    D --> G[实体集合 C]
~~~

业务架构完全一样，但因为不同 pipeline 保留和归一化的 dimensions 不同，最终得到的实体拓扑可能不同。

---

## 12. 与 OpenTelemetry Semantic Conventions 的关系

OpenTelemetry 的数据模型倾向于明确区分：

1. 本地服务是谁；
2. 本地服务正在调用谁。

### 12.1 本地服务身份

~~~text
service.name = checkout-service
~~~

描述产生 telemetry 的本地服务。

### 12.2 远端服务器地址

~~~text
server.address = payment.internal
~~~

在很多客户端语义约定中，用于描述远端服务器的逻辑地址。

OpenTelemetry 中的 server.address 是稳定属性，通常推荐记录逻辑 hostname/address，而不是通过反向 DNS 把地址转成另一个名称。

### 12.3 Peer Service 属性

OpenTelemetry 还定义了：

~~~text
service.peer.name
service.peer.namespace
~~~

用于描述连接另一端的逻辑服务。

截至本次调研日期，这些 peer service 属性在 OpenTelemetry Semantic Conventions 中仍标记为 **Development**。

因此不能直接假设：

~~~text
OpenTelemetry service.peer.name
==
Datadog peer.service
~~~

它们属于两个不同的数据和语义模型。

真正应该实验验证的问题是：

> **某一种具体的 Datadog ingestion path，会如何把一组 OpenTelemetry 属性映射成 Datadog 的 peer identity 和 inferred entity？**

---

## 13. Service Naming：为什么推断服务让模型更清晰

历史上一个常见问题是：

> 使用 Span 的 service name 来表示远端依赖。

例如，一个进程逻辑上只有：

~~~text
checkout-service
~~~

但是不同 integration 产生的 CLIENT Span 可能出现：

~~~text
checkout-service
postgres
redis
http-client
~~~

这样相当于让一个字段同时表达两个概念：

1. 谁产生了 telemetry；
2. 它正在调用谁。

更清晰的模型应该是：

~~~mermaid
flowchart LR
    S[CLIENT Span]
    S --> L[本地身份<br/>service=checkout-service]
    S --> R[远端身份<br/>peer=postgres]
~~~

Inferred Services 允许 Datadog 在不修改本地 service identity 的前提下表达下游 dependency。

因此从数据建模角度看，更合理的是：

~~~text
service
=
谁产生了 telemetry

peer
=
它正在调用谁
~~~

---

## 14. 不只是 Service Map：Datadog 的实体模型

Datadog 提供了一个 DDSQL Dataset：

~~~text
dd.inferred_services
~~~

这个数据集包含从以下来源推导出的 backend service entity：

- APM；
- Universal Service Monitoring（USM）；
- Software Catalog。

这些实体通常还没有被完整解析成一个显式定义的服务实体。

因此可以把整体架构理解成：

~~~mermaid
flowchart TD
    T[Telemetry]
    T --> PI[Peer Identity]
    PI --> E[Entity Model]

    E --> SM[Service / Dependency Map]
    E --> TV[Trace View]
    E --> M[Dependency Metrics]
    E --> C[Catalog]
    E --> SQL[DDSQL]
~~~

所以 Inferred Services 更适合作为 Datadog **Entity Resolution（实体解析）层**的一部分来研究，而不只是一个 APM UI 功能。

---

## 15. Cardinality 风险

依赖推断和 Cardinality 管理是强相关的。

容易产生问题的 identity 包括：

- 临时 IP 地址；
- Kubernetes Pod 地址；
- 每个请求都不同的 hostname；
- 动态 Queue / Topic 名称；
- 带 Tenant ID 的数据库名称；
- 用户可控的 peer label。

如果不稳定的值进入聚合维度，那么推断实体数量和 Metric Series 数量都可能快速增长。

Datadog 对看起来像 IP 地址的 peer value 有特殊处理，这类 dependency 可能显示成：

~~~text
blocked-ip-address
~~~

而不是让大量 IP 直接形成独立的 peer entity。

OpenTelemetry Semantic Conventions 同样倾向于优先使用稳定、低基数的逻辑服务身份。

### 设计原则

如果可以获得逻辑服务地址，应优先考虑：

~~~text
payment.default.svc.cluster.local
~~~

而不是：

~~~text
10.42.7.183
~~~

因为后者往往是短生命周期、基础设施级的地址，不适合作为长期稳定的服务身份。

---

## 16. 容易出现错误或歧义的场景

| 场景 | 风险 |
|---|---|
| 使用 Kubernetes Pod IP 作为 peer identity | 实体频繁变化、高基数 |
| Service Mesh Proxy | 观察到的 peer 可能是 Proxy，而不是真实逻辑服务 |
| Gateway / Load Balancer | server.address 可能表示中间层 |
| 动态 Kafka Topic | 产生大量 inferred destination entity |
| SDK 层已经做 Trace Sampling | Dependency Metrics 只能反映采样后的流量 |
| 手工设置错误的 peer.service | 高优先级错误值可能覆盖更合理的 hostname |
| Semantic Convention 版本变化 | 同一 workload 可能产生不同属性 |
| 混用 Datadog 与 OTel Instrumentation | 可能出现重复或不一致的 dependency identity |
| Integration Service Override | 一个依赖可能同时表现成 Service 和 Inferred Entity |

---

## 17. Observability-Lab 需要验证的问题

以下问题应该通过实验回答，而不是只根据文档推测。

### 17.1 Entity Identity

1. 不同 ingestion path 下，哪些 OTel 属性最终会成为 Datadog peer identity？
2. server.address 与显式 peer-service 属性同时存在时如何选择？
3. logical peer name 和 hostname 同时存在时优先级是什么？
4. 数据库 namespace 与数据库 hostname 如何排序？
5. Kafka topic/destination 与 Broker identity 如何表现？

### 17.2 Pipeline 差异

建议比较：

1. Datadog SDK + Agent；
2. OTel SDK + Datadog Agent OTLP Ingest；
3. OTel SDK + Upstream Collector + 推荐的 span_metrics；
4. OTel SDK + DDOT Collector；
5. 在适用场景下测试 OTel SDK + Datadog Connector。

### 17.3 Sampling

需要验证：

1. SDK Head Sampling 与 Collector Tail Sampling；
2. span_metrics 位于 sampling 之前和之后的区别；
3. 两种方案下 dependency metric 的准确性。

### 17.4 Cardinality

需要验证：

1. logical hostname 与 IP；
2. 固定 Topic 与动态 Topic；
3. Tenant 独立数据库；
4. Kubernetes Service DNS 与 Pod IP。

---

## 18. 建议实验

创建：

~~~text
labs/datadog-inferred-services/
~~~

使用一个 checkout-service，同时访问：

~~~mermaid
flowchart LR
    C[checkout-service]
    C --> PG[(PostgreSQL)]
    C --> R[(Redis)]
    C --> K[Kafka<br/>order.created]
    C --> EXT[api.stripe.com]
    C --> P[payment-service]
~~~

对于每一个 outbound Span，记录：

- 原始 SpanKind；
- 原始 Span Attributes；
- Resource Attributes；
- Datadog peer identity；
- 推断出的实体名称；
- 推断出的实体类型；
- Dependency Metrics；
- Service / Dependency Map 中的结果。

### 18.1 属性变更矩阵

执行以下受控实验：

| 变体 | 属性变化 | 主要验证目标 |
|---|---|---|
| A | 仅提供 server.address | 验证 hostname-based inference |
| B | logical peer + server.address | 验证优先级 |
| C | db.namespace + DB Host | 验证数据库 identity 优先级 |
| D | Kafka Destination + Broker | 验证 Messaging identity 优先级 |
| E | 使用 IP 作为 Peer Address | 验证 Cardinality / IP 处理 |
| F | SDK 开启 Sampling | 验证 Metrics 准确性影响 |
| G | span_metrics 后执行 Collector Sampling | 验证未采样 Metrics 的保留 |

### 18.2 实验产物

实验最终应该保留：

- 原始 OTLP Span Fixtures；
- Collector 配置；
- Datadog 截图或导出的 Query 结果；
- Entity Mapping 表；
- Metrics 对比表；
- 完整 Version Matrix。

---

## 19. 当前结论

在这个仓库里，可以使用下面的定义：

> **Datadog Inferred Services 是根据出站 telemetry 推导出的远端依赖实体。Datadog 从 Span 属性中解析 peer identity，并将这个身份用于依赖拓扑、实体模型和 APM 统计。**

从架构角度看，更值得关注的是下面这条链路：

~~~text
Tracing 不只是存储 Span

Span 语义
    -> 身份解析
    -> 拓扑
    -> Metrics
    -> Entity Model
~~~

这也是 OpenTelemetry 与 Datadog 做互操作时必须深入验证的一层。

两个系统都支持 OTLP，并不代表它们会：

- 以相同方式解释 Span；
- 得到相同的 dependency identity；
- 构建相同的 Service Map；
- 计算相同的 dependency metrics；
- 生成相同的实体模型。

因此，**协议兼容只是互操作性的第一层，语义与实体解析才是下一层需要验证的问题。**

---

## 参考资料

### Datadog

- Inferred Services  
  https://docs.datadoghq.com/tracing/services/inferred_services/
- OpenTelemetry Trace Metrics  
  https://docs.datadoghq.com/opentelemetry/integrations/trace_metrics/
- Ingestion Sampling with OpenTelemetry  
  https://docs.datadoghq.com/opentelemetry/ingestion_sampling/
- Trace Metrics  
  https://docs.datadoghq.com/tracing/metrics/metrics_namespace/
- OpenTelemetry Getting Started  
  https://docs.datadoghq.com/opentelemetry/getting_started/
- DDSQL: Inferred Services  
  https://docs.datadoghq.com/ddsql_reference/data_directory/dd/dd.inferred_services.dataset/

### OpenTelemetry

- Semantic Conventions  
  https://opentelemetry.io/docs/specs/otel/semantic-conventions/
- General Network Attributes / server.address  
  https://opentelemetry.io/docs/specs/semconv/general/attributes/
- Service Peer Attributes  
  https://opentelemetry.io/docs/specs/semconv/registry/attributes/service/
- Database Client Spans  
  https://opentelemetry.io/docs/specs/semconv/db/database-spans/
- Kafka Semantic Conventions  
  https://opentelemetry.io/docs/specs/semconv/messaging/kafka/
