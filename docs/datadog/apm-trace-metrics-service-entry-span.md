# Datadog APM Trace Metrics 与 Service Entry Span 深入调研

> 调研日期：2026-09-29  
> 范围：Datadog APM Trace Metrics、Service Entry Span、Measured Span、Primary Operation，以及 OpenTelemetry SpanKind 到 Datadog APM Stats 的语义映射。

## 1. 摘要

Datadog APM 中最重要但容易被忽略的一层，是：

~~~text
Span
→ Datadog Span Classification
→ Trace Metrics
→ Service / Resource Statistics
→ Service Page / Catalog / Service Map
~~~

这里的关键不是“有没有 Trace”，而是：

> Datadog 把哪些 Span 认定为值得计算 APM Stats 的 Span？

Datadog 当前的核心规则可以概括为：

~~~text
Service Entry Span
+
Measured Span
→
Trace Metrics
~~~

对 OpenTelemetry 而言，Datadog 当前文档给出的映射是：

| OpenTelemetry | Datadog |
|---|---|
| Root Span | Service Entry Span |
| span.kind=server | Service Entry Span |
| span.kind=consumer | Service Entry Span |
| span.kind=client | Measured Span |
| span.kind=producer | Measured Span |
| span.kind=internal | 默认不生成 Trace Metrics |

~~~mermaid
flowchart LR
    S[SERVER] --> SE[Service Entry]
    C[CONSUMER] --> SE
    CL[CLIENT] --> M[Measured Span]
    P[PRODUCER] --> M
    I[INTERNAL] --> N[默认无 Trace Metrics]
    SE --> TM[Trace Metrics]
    M --> TM
~~~

这直接决定：

- Service Page 是否有 RED Metrics；
- Resource 是否有 Throughput / Error / Latency；
- Client / DB / Messaging 调用是否产生 APM Stats；
- OTel 数据在 Datadog 中是否表现为正常 APM Service。

---

## 2. Trace Metrics 是什么

Datadog 自动为 APM Service 和 Resource 生成 Trace Metrics。

核心 Namespace：

~~~text
trace.<SPAN_NAME>.<METRIC_SUFFIX>
~~~

其中 SPAN_NAME 是 Operation Name / Span Name。

常见指标：

~~~text
trace.<SPAN_NAME>.hits
trace.<SPAN_NAME>.errors
trace.<SPAN_NAME>.apdex
trace.<SPAN_NAME>.duration
~~~

当前更推荐用于延迟分析的是：

~~~text
trace.<SPAN_NAME>
~~~

它是 Datadog Distribution Metric。

---

## 3. Hits

~~~text
trace.<SPAN_NAME>.hits
~~~

类型：

~~~text
COUNT
~~~

表示具有该 Operation / Span Name 的统计 Span 数量。

常见维度包括：

- env；
- service；
- version；
- resource / resource_name；
- http.status_code；
- rpc.grpc.status_code；
- Host Tags；
- Additional Primary Tags。

对于 HTTP/Web Service，还存在：

~~~text
trace.<SPAN_NAME>.hits.by_http_status
~~~

用于按照 HTTP Status 分组。

---

## 4. Errors

~~~text
trace.<SPAN_NAME>.errors
~~~

类型：

~~~text
COUNT
~~~

表示对应 Operation 的 Error 数量。

还存在：

~~~text
trace.<SPAN_NAME>.errors.by_http_status
~~~

可以按 http.status_class 和 http.status_code 拆分。

典型 Error Rate 可以理解为：

~~~text
errors / hits
~~~

---

## 5. Latency Distribution

当前 Datadog 推荐的延迟指标是：

~~~text
trace.<SPAN_NAME>
~~~

类型：

~~~text
DISTRIBUTION
~~~

用于：

- avg latency；
- p50；
- p75；
- p90；
- p95；
- p99。

Datadog APM Service Page 和 Resource Page 会使用这类 Distribution Metric。

Trace Metrics 当前保留 15 个月，因此长期性能趋势主要依赖 Trace Metrics，而不是长期保存全部 Raw Trace。

---

## 6. duration 已是 Legacy Metric

旧指标：

~~~text
trace.<SPAN_NAME>.duration
~~~

仍然存在，但 Datadog 当前明确建议新的延迟分析优先使用 Distribution Metric。

duration 表达一个时间窗口内所有匹配 Span 的 Duration 总和。

平均延迟可以近似计算：

~~~text
sum(duration)
/
sum(hits)
~~~

但 duration 不支持 Percentile Aggregation。

因此新的 Dashboard / Monitor 应优先使用：

~~~text
trace.<SPAN_NAME>
~~~

而不是 legacy duration。

---

## 7. Apdex

HTTP/Web APM Service 还可以产生：

~~~text
trace.<SPAN_NAME>.apdex
~~~

类型为 GAUGE。

它是服务体验的历史指标。现代性能分析更常直接结合：

- Error Rate；
- Throughput；
- Latency Distribution；
- p95 / p99。

---

## 8. Trace Metrics 为什么通常不受 Ingestion Sampling 影响

Datadog Native 的核心架构可以理解为：

~~~mermaid
flowchart LR
    APP[Application<br/>100% traffic]
    AG[Datadog Agent]
    APP --> AG
    AG --> STATS[Trace Metrics]
    AG --> SAMPLE[Trace Sampling]
    SAMPLE --> RAW[Raw Trace Ingestion]
~~~

因此：

~~~text
Trace Metrics
~~~

通常基于：

~~~text
100% Application Traffic
~~~

而 Raw Trace 可以只 Ingest 一部分。

例如：

~~~text
真实请求 = 10000/s
Trace Ingestion = 5%
~~~

APM Hits 仍可以接近真实的 10000/s，同时 Raw Trace 只进入约 5%。

这也是 Datadog 把 Monitoring 与 Raw Trace Debug 解耦的基础。

---

## 9. “全量 Trace Metrics”存在例外

如果 Span 在到达 Stats 计算点之前已经被丢弃，Datadog无法恢复。

### 9.1 Application-side Sampling

~~~text
Application
→ Sampling
→ Agent
~~~

Agent 只看到采样后的 Span。

### 9.2 OpenTelemetry SDK Sampling

例如：

~~~text
ParentBased(
    TraceIdRatioBased(0.1)
)
~~~

OTel SDK 只发送约 10%，那么后续 Datadog Trace Metrics 也只能基于这部分数据。

### 9.3 其他上游采样

例如 AWS X-Ray 等系统如果在发送给 Datadog 之前已经采样，也存在同样问题。

核心原则：

> Stats Computation Point 必须位于不可逆 Sampling 之前，才可能得到全量 Stats。

---

## 10. Service Entry Span 是什么

Service Entry Span 是 Datadog 特有的 Span Classification。

它表示：

> 请求或工作进入某个 Service 的入口。

例如：

~~~mermaid
flowchart LR
    U[User]
    C[checkout SERVER]
    PC[payment CLIENT]
    PS[payment SERVER]
    DB[postgres CLIENT]
    U --> C
    C --> PC
    PC --> PS
    PS --> DB
~~~

其中：

~~~text
checkout SERVER
payment SERVER
~~~

都代表一次进入对应 Service 的请求。

因此 Datadog 可以分别对 checkout 和 payment 计算：

- Rate；
- Errors；
- Latency。

---

## 11. Service Entry Span 不等于 Trace Root

一条 Distributed Trace 只有一个 Root Span，但可以有多个 Service Entry Span。

~~~mermaid
flowchart LR
    A[frontend SERVER<br/>Root + Service Entry]
    B[checkout SERVER<br/>Service Entry]
    C[payment SERVER<br/>Service Entry]
    D[inventory SERVER<br/>Service Entry]
    A --> B --> C
    B --> D
~~~

因此：

~~~text
Root Span
通常也是一个 Service Entry

但每个下游 Service
还可以有自己的 Service Entry Span
~~~

这也是 Datadog 可以对每个微服务独立计算 RED Metrics 的基础。

---

## 12. 为什么 Consumer 也是 Service Entry

消息系统里没有传统 HTTP Server Request。

例如：

~~~mermaid
flowchart LR
    P[order-service<br/>PRODUCER]
    K[Kafka]
    C[fulfillment-service<br/>CONSUMER]
    P --> K --> C
~~~

对于 fulfillment-service，业务入口是 Kafka Consumer Span。

因此 Datadog 当前将：

~~~text
span.kind=consumer
~~~

映射成：

~~~text
Service Entry Span
~~~

这样 Queue Consumer Service 也可以拥有：

- Hits；
- Errors；
- Latency；
- Service Page。

---

## 13. Measured Span 是什么

并不是只有服务入口才值得计算 Stats。

例如：

~~~mermaid
flowchart LR
    S[checkout SERVER]
    DB[postgres CLIENT]
    HTTP[payment CLIENT]
    K[kafka PRODUCER]
    S --> DB
    S --> HTTP
    S --> K
~~~

这些 CLIENT / PRODUCER Span 不是 Service Entry，因为它们表示离开当前 Service 的调用。

但这些 Operation 本身仍需要：

- DB Query Rate；
- DB Error；
- DB Latency；
- HTTP Client Latency；
- Messaging Publish Latency。

因此 Datadog 将它们视为：

~~~text
Measured Span
~~~

并生成 Trace Metrics。

Datadog Native tracer 的内部 payload 中可以看到类似：

~~~text
_dd.measured = 1
~~~

的内部标记。这个字段属于 Datadog 内部实现细节，不应作为跨 Vendor 的公共 Semantic Convention 使用。

---

## 14. Service Entry 与 Measured Span 的关系

~~~mermaid
flowchart TD
    S[Span]
    S --> E{进入一个 Service?}
    E -->|是| SE[Service Entry Span]
    S --> O{重要出站 Operation?}
    O -->|是| M[Measured Span]
    SE --> TM[Trace Metrics]
    M --> TM
~~~

Service Entry 主要回答：

> 这个 Service 的入口请求表现怎么样？

Measured Span 主要回答：

> 这个重要 Operation / Dependency 的表现怎么样？

---

## 15. OpenTelemetry → Datadog 当前映射

Datadog 当前官方映射：

| OTel Convention | Datadog Convention |
|---|---|
| Root Span | Service Entry Span |
| span.kind=server | Service Entry Span |
| span.kind=consumer | Service Entry Span |
| span.kind=client | Measured Span |
| span.kind=producer | Measured Span |
| span.kind=internal | 默认不生成 Trace Metrics |

这套逻辑比单纯依赖 Parent/Child 关系更接近 OpenTelemetry SpanKind 的语义。

---

## 16. INTERNAL Span 为什么默认不生成 Trace Metrics

INTERNAL Span 通常表示：

- Service 内部函数；
- Framework Middleware；
- 业务内部步骤。

例如：

~~~mermaid
flowchart TD
    S[SERVER request]
    A[validate-cart INTERNAL]
    B[calculate-price INTERNAL]
    C[apply-promotion INTERNAL]
    S --> A --> B --> C
~~~

如果每个 INTERNAL Span 都自动产生 Trace Metrics：

~~~text
Metric Series 数量
+
Operation Cardinality
~~~

会迅速增长，同时 Service Page 也可能混入大量实现细节。

因此默认：

~~~text
INTERNAL
→ No Trace Metrics
~~~

形成清晰边界。

---

## 17. 如果需要 INTERNAL Span 的指标

Datadog 文档指出，可以在 OTel Collector 中用 transform processor 修改 SpanKind。

概念上：

~~~text
Internal
↓
OTel Transform
↓
Client
↓
Measured Span
↓
Trace Metrics
~~~

但这等于改变 Span 的语义分类。

因此不应该只是为了“想看一个 Metric”就随意修改 SpanKind。

如果只需要某个 Internal Operation 的业务指标，更合理的选择通常是：

- Custom Metric from Spans；
- 手工 Metric；
- 或重新检查 SpanKind 是否原本设置错误。

---

## 18. Primary Operation

Service Entry Span 解决：

> 哪些 Span 是 Service 的入口？

但一个 Service 仍可能有多个入口 Operation：

~~~text
web.request
grpc.server
celery.run
custom.request
~~~

Datadog 还需要选择：

> 哪个 Operation 代表 Service 的默认性能视角？

这就是：

~~~text
Primary Operation
~~~

---

## 19. Primary Operation 如何选择

Datadog Backend 会自动选择被认为最像 Service Entry Point 的 Operation Name。

如果一个 Service 有多个候选 Primary Operation：

~~~text
Operation A: 1000 req/s
Operation B: 50 req/s
Operation C: 2 req/s
~~~

默认会选择最高 Request Throughput 的 Operation。

例如：

~~~text
service = web-store

resources:
GET /user/home
GET /user/new
POST /checkout

operation:
web.request
~~~

这些 Resource 共享：

~~~text
Primary Operation = web.request
~~~

---

## 20. Operation Name 与 Resource Name 必须区分

推荐模型：

~~~text
Operation Name
=
稳定、低基数的操作类型

Resource Name
=
具体 Endpoint / Query / Operation Target
~~~

例如：

~~~text
operation:
web.request

resource:
GET /users/{id}
~~~

而不应该把：

~~~text
GET /users/123456
~~~

直接作为 Operation Name。

错误设计会造成：

- Operation Cardinality；
- Primary Operation 混乱；
- Service Page 数据碎片化。

Datadog 当前建议手工 instrumentation 时保持 Span Name 静态，把动态 Endpoint 放到 Resource。

---

## 21. Primary Operation 对 Service Page 的影响

Service-level Statistics 围绕 Primary Operation 展示。

~~~mermaid
flowchart TD
    S[Service]
    S --> OP1[web.request<br/>1000 rps]
    S --> OP2[admin.request<br/>10 rps]
    OP1 --> P[Primary Operation]
    P --> PAGE[Service Page]
    P --> CAT[Catalog]
    P --> MAP[Service Map]
~~~

Secondary Operation 仍可查看 Resource Stats，但不用于默认 Service-level Statistics。

所以当 Service Page 出现：

- Throughput 明显偏低；
- Error Rate 对不上；
- Latency 与真实入口不一致；

应该检查 Primary Operation 是否选错。

---

## 22. Primary Operation 可以手动覆盖

Datadog允许管理员在 APM Settings 中手工指定 Primary Operation。

适合：

- 一个 Service 同时跑 HTTP 和 Worker；
- 自动选择到高流量但非核心 Operation；
- Operation Name Migration；
- 自定义 instrumentation。

但长期更合理的是保证：

- SpanKind；
- span.name；
- resource；
- service；

本身具有稳定一致的语义。

---

## 23. Trace Metric Tags 不是全部 Span Tags

Span 上有一个 Tag，并不意味着 Trace Metric 一定能按它 Group By。

Trace Metrics 默认只有有限维度，例如：

- env；
- service；
- version；
- resource / resource_name；
- http.status_code；
- http.status_class；
- rpc.grpc.status_code；
- Host Tags；
- Additional Primary Tags。

普通自定义 Span Tag：

~~~text
customer.id
order.id
tenant.id
~~~

不会自动成为 Trace Metric Dimension。

这是为了控制 Metric Cardinality。

---

## 24. Additional Primary Tags

如果确实需要额外维度聚合 Trace Metrics，可以配置 Additional Primary Tags。

典型适合：

- availability_zone；
- datacenter；
- cluster；
- region。

这类维度应该：

- 低基数；
- 稳定；
- 对整个 APM Scope 有意义。

Datadog 当前文档给出的建议上限是：

~~~text
100 unique values per additional primary tag
~~~

因此 user.id、request.id、order.id 这类高基数字段不适合作为 Primary Tag。

---

## 25. Trace Metrics 的 Volume Guidelines

Datadog 当前对 APM Stats 有明确的数据量 Guidelines。

一个 40-minute Interval 内典型约束包括：

~~~text
5000 unique env + service combinations
100 unique operation names per env/service
1000 unique resources per env/service/operation
30 unique versions per env/service
100 unique values per additional primary tag
~~~

因为 Trace Metrics 基于大量未采样 Stats，Cardinality 爆炸会直接影响 APM 服务统计。

因此错误的：

- service name；
- operation name；
- resource name；
- primary tags；

设计不只是 UI 问题，还可能导致 Stats 延迟、丢失或服务无法正常出现在 Catalog Performance 中。

---

## 26. OTel Service Entry Mapping 的三条接入路径

Datadog 当前针对不同 OTel ingestion path 有不同要求。

### 26.1 OTLP HTTP + span_metrics

当前推荐配置要求：

~~~text
OpenTelemetry Collector Contrib >= 0.154.0
~~~

使用 Datadog 推荐的完整 span_metrics dimensions 配置。

这套配置可以让 Datadog识别：

- Service Entry Span；
- Operation Name；

不需要额外开启旧的 Service Entry Feature Flag。

### 26.2 Datadog Exporter + Datadog Connector

当前要求：

~~~text
OpenTelemetry Collector Contrib >= 0.100.0
~~~

并设置：

~~~yaml
traces:
  compute_top_level_by_span_kind: true
~~~

如果 Datadog Exporter 与 Datadog Connector 同时存在，两边都需要启用。

### 26.3 Datadog Agent OTLP Ingest

当前要求：

~~~text
Datadog Agent >= 7.53.0
~~~

并可通过：

~~~text
enable_otlp_compute_top_level_by_span_kind
~~~

启用新的基于 SpanKind 的 Service Entry 判断逻辑。

因此 OTel → Datadog 的 APM Stats 行为与 Agent / Collector Version 直接相关。

---

## 27. DDOT Connector 推荐配置

DDOT 当前示例中包含：

~~~yaml
connectors:
  datadog/connector:
    traces:
      compute_top_level_by_span_kind: true
      peer_tags_aggregation: true
      compute_stats_by_span_kind: true
~~~

这三个配置正好连接三类 Datadog 语义：

~~~mermaid
flowchart TD
    A[compute_top_level_by_span_kind]
    B[compute_stats_by_span_kind]
    C[peer_tags_aggregation]
    A --> SE[Service Entry Span]
    B --> TM[Trace Metrics]
    C --> PEER[Peer / Inferred Services]
    SE --> APM[Datadog APM Model]
    TM --> APM
    PEER --> APM
~~~

说明：

> Service Metrics 与 Dependency Metrics 共同依赖 Span Classification。

---

## 28. 新 Service Entry Logic 为什么可能影响 Monitor

Datadog 当前迁移文档明确警告：

> 新的 SpanKind Mapping 会改变产生 Trace Metrics 的 Span 集合。

例如以前某个 CLIENT Span 没被正确统计。

启用后：

~~~text
CLIENT
→ Measured Span
→ Trace Metrics
~~~

对应 Metric 可能增加。

反过来，一些以前被旧逻辑统计的 INTERNAL Span，在新逻辑下：

~~~text
INTERNAL
→ No Trace Metrics
~~~

指标可能减少。

因此升级可能影响：

- APM Monitor；
- Dashboard；
- SLO；
- Service Page。

---

## 29. Operation Name Mapping 也可能是 Metrics Schema Migration

Trace Metric 名称是：

~~~text
trace.<SPAN_NAME>.*
~~~

如果 OTel → Datadog Operation Name Mapping 改变：

~~~text
旧 Operation Name
→
新 Operation Name
~~~

对应 Trace Metric Namespace 也可能变化。

因此 Operation Name Migration 本质上可能同时是：

> Metrics Schema Migration。

Datadog 当前新的 Operation Name Mapping Logic 已明确提醒，这可能对引用旧 Operation Name 的 Monitor / Dashboard 构成 Breaking Change。

---

## 30. APM Metrics 与 Service Model 的完整关系

~~~mermaid
flowchart TD
    SPAN[Span]

    SPAN --> KIND[SpanKind]
    SPAN --> NAME[Operation Name]
    SPAN --> RES[Resource Name]
    SPAN --> SERVICE[Service Name]

    KIND --> CLASS[Service Entry / Measured]
    NAME --> CLASS

    CLASS --> STATS[Trace Metrics]

    STATS --> SERVICEPAGE[Service Page]
    STATS --> RESOURCEPAGE[Resource Page]
    STATS --> MONITOR[APM Monitor]

    SERVICE --> PRIMARY[Primary Operation Selection]
    NAME --> PRIMARY
    STATS --> PRIMARY

    PRIMARY --> SERVICEPAGE
~~~

所以 Service Page 不是简单读取 Raw Span。

它依赖：

~~~text
SpanKind
→ Span Classification
→ Trace Stats
→ Operation Selection
→ Service Model
~~~

---

## 31. 与 Inferred Services 的连接

之前研究的 Inferred Services 主要从 CLIENT / PRODUCER Span 提取 Peer Identity。

同时 CLIENT / PRODUCER 在这里又映射为 Measured Span。

~~~mermaid
flowchart TD
    C[CLIENT / PRODUCER Span]
    C --> M[Measured Span]
    M --> TM[Trace Metrics]
    C --> P[Peer Attributes]
    P --> IS[Inferred Service]
    TM --> DM[Dependency Performance]
    IS --> DM
~~~

这正是：

> Operation Metrics + Entity Identity

结合成 Dependency Observability 的地方。

---

## 32. 与 Sampling 的连接

Datadog Native：

~~~text
Span
→ Agent Stats
→ Trace Metrics

Span
→ Sampling
→ Raw Trace
~~~

因此 Sampling 与 Metrics 可以解耦。

OTel：

~~~text
SDK Sampling
→ Collector
~~~

如果 SDK 已经 Drop，Stats 无法恢复。

如果 Collector：

~~~text
span_metrics
→ tail_sampling
~~~

则仍可保持更完整 Stats。

所以 Trace Metrics 准确性的根本问题是：

> Stats Computation Point 是否位于不可逆 Sampling 之前。

---

## 33. 与 Retention 的连接

Trace Metrics 不依赖 Retention。

~~~mermaid
flowchart LR
    SPAN[Span]
    SPAN --> STATS[Trace Metrics]
    SPAN --> RET{Retention}
    RET -->|KEEP| RAW[Historical Raw Span]
    RET -->|DROP| D[No historical raw span]
~~~

这意味着可能出现：

> Service Page 显示昨天某个 Resource 的 p99 很高，但 Retention 中没有足够具体 Trace 可以继续 Debug。

因此 Metrics 与 Retention 必须一起设计。

---

## 34. Service Entry 示例

假设：

~~~text
service = checkout
span.name = web.request
resource = POST /checkout
span.kind = server
~~~

Datadog识别为 Service Entry Span，并生成：

~~~text
trace.web.request.hits
trace.web.request.errors
trace.web.request
~~~

这些 Metric 可以按：

~~~text
service:checkout
resource:"POST /checkout"
env:prod
~~~

查询。

web.request 也可能成为 checkout 的 Primary Operation。

---

## 35. Client Span 示例

~~~text
service = checkout
span.name = postgres.query
resource = SELECT orders
span.kind = client
db.system = postgresql
db.namespace = orders
~~~

Datadog映射：

~~~text
CLIENT
→ Measured Span
~~~

从而可以产生：

~~~text
trace.postgres.query.hits
trace.postgres.query.errors
trace.postgres.query
~~~

同时 Peer Identity 可能解析成数据库依赖实体。

于是 Datadog 同时拥有：

~~~text
Postgres Operation Performance
+
Postgres Dependency Identity
~~~

---

## 36. Messaging 示例

Producer：

~~~text
span.kind = producer
messaging.system = kafka
messaging.destination.name = order.created
~~~

映射成 Measured Span。

Consumer：

~~~text
span.kind = consumer
~~~

映射成 Service Entry Span。

~~~mermaid
flowchart LR
    O[order-service]
    P[Kafka PRODUCER<br/>Measured]
    K[order.created]
    C[fulfillment CONSUMER<br/>Service Entry]
    O --> P --> K --> C
~~~

这样可以同时形成：

- Producer latency/error stats；
- Consumer service RED metrics；
- Kafka dependency identity。

---

## 37. 常见问题排查矩阵

| 症状 | 优先检查 |
|---|---|
| Service 有 Trace 但没有 APM Metrics | SpanKind / Service Entry Mapping |
| Service Page Throughput 太低 | SDK Sampling / Stats 计算位置 |
| DB Client Span 没有 Metrics | CLIENT 是否映射成 Measured |
| Internal Span 没有 Trace Metrics | 默认行为，不一定是故障 |
| Service Page 主 Operation 错了 | Primary Operation |
| Resource 数量爆炸 | Resource Name Cardinality |
| Operation 太多 | Span Name 是否动态 |
| Monitor 升级后数值变化 | 新 SpanKind Mapping / Operation Mapping |
| OTel Stats 与 Native Datadog 不一致 | span_metrics dimensions / Collector Version |
| Trace 能看到但 Catalog 没性能数据 | Service Entry / Primary Operation / Volume Guidelines |

---

## 38. Observability-Lab 建议实验

建议创建：

~~~text
labs/datadog-trace-metrics/
~~~

统一 Workload：

~~~mermaid
flowchart LR
    L[Load Generator]
    A[checkout]
    B[payment]
    DB[(Postgres)]
    K[Kafka]
    L --> A
    A --> B
    A --> DB
    A --> K
~~~

### 实验 A：SpanKind Matrix

手工生成：

~~~text
SERVER
CONSUMER
CLIENT
PRODUCER
INTERNAL
~~~

验证：

- Service Entry；
- Measured；
- Trace Metric 是否存在。

### 实验 B：Primary Operation

同一个 Service 创建：

~~~text
web.request
admin.request
worker.request
~~~

控制不同 Throughput，验证 Datadog 自动选择哪个 Primary Operation。

### 实验 C：动态 Span Name

比较：

~~~text
正确：
span.name=web.request
resource=GET /user/{id}

错误：
span.name=GET /user/123456
~~~

观察 Operation Cardinality。

### 实验 D：OTel Mapping

比较：

1. Upstream Collector + span_metrics；
2. Datadog Connector；
3. Agent OTLP。

验证同一 OTel Trace 最终生成的：

- Operation Name；
- Service Entry；
- Metrics Namespace；
- Resource；
- Peer Service。

### 实验 E：Sampling Position

比较：

~~~text
span_metrics → tail_sampling
~~~

与：

~~~text
tail_sampling → span_metrics
~~~

验证 hits/errors/latency。

### 实验 F：Operation Mapping Migration

切换 Operation Name Logic 前后比较：

- Metrics Name；
- Service Page；
- Monitor；
- Dashboard。

---

## 39. 当前结论

Datadog APM Trace Metrics 不能简单理解成：

> 从 Trace 自动生成几个 Metric。

更准确的模型是：

~~~text
Span Semantics
    ↓
Service Entry / Measured Classification
    ↓
Operation / Resource Modeling
    ↓
Trace Stats Aggregation
    ↓
Primary Operation
    ↓
Service / Resource Performance Model
~~~

其中：

~~~text
SERVER / CONSUMER
→ Service Entry

CLIENT / PRODUCER
→ Measured

INTERNAL
→ 默认无 Trace Metrics
~~~

是 OpenTelemetry → Datadog 互操作的关键语义边界。

而：

~~~text
Service Entry Span
+
Primary Operation
+
Trace Metrics
~~~

共同决定：

> Datadog 眼里的一个 APM Service 到底是什么。

所以验证 Datadog 与 OpenTelemetry 兼容性时，不能只检查 Trace 是否成功写入；至少还要验证：

1. Service Entry 是否正确；
2. Measured Span 是否正确；
3. Operation Name 是否稳定；
4. Resource Name 是否低基数；
5. Trace Metrics 是否完整；
6. Primary Operation 是否正确；
7. Peer / Inferred Service 是否正确；
8. Sampling 是否发生在 Stats 计算之后。

---

## 相关调研

- [APM Trace Pipeline](./apm-trace-pipeline.md)
- [APM 采样策略](./apm-sampling.md)
- [APM Trace 数据保留策略](./apm-trace-retention.md)
- [推断服务（Inferred Services）](./inferred-services.md)

---

## 参考资料

### Datadog APM

- Trace Metrics  
  https://docs.datadoghq.com/tracing/metrics/metrics_namespace/
- APM Metrics  
  https://docs.datadoghq.com/tracing/metrics/
- Primary Operations in Services  
  https://docs.datadoghq.com/tracing/guide/configuring-primary-operation/
- DDSketch-based Metrics in APM  
  https://docs.datadoghq.com/tracing/guide/ddsketch_trace_metrics/
- Primary Tags  
  https://docs.datadoghq.com/tracing/guide/setting_primary_tags_to_scope/
- APM Troubleshooting / Data Volume Guidelines  
  https://docs.datadoghq.com/tracing/troubleshooting/

### Datadog + OpenTelemetry

- Mapping OpenTelemetry Semantic Conventions to Service-entry Spans  
  https://docs.datadoghq.com/opentelemetry/mapping/service_entry_spans/
- OpenTelemetry Trace Metrics  
  https://docs.datadoghq.com/opentelemetry/integrations/trace_metrics/
- Semantic Mapping  
  https://docs.datadoghq.com/opentelemetry/mapping/semantic_mapping/
- Operation Name Migration  
  https://docs.datadoghq.com/opentelemetry/migrate/migrate_operation_names/
- DDOT Collector Setup  
  https://docs.datadoghq.com/opentelemetry/setup/ddot_collector/
