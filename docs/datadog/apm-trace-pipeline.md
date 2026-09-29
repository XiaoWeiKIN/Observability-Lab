# Datadog APM Trace Pipeline 深入调研

> 调研日期：2026-09-29  
> 范围：Datadog APM Trace Pipeline 的完整数据生命周期，包括 Instrumentation、Sampling、Ingestion、Trace Metrics、Backend Enrichment、Processing Pipelines、Custom Metrics from Spans、Live Search、Retention、Trace Queries、Usage/Cost，以及 OpenTelemetry → Datadog 路线。

## 1. 摘要

Datadog Trace Pipeline 不应该被理解成一条简单的“Span → Datadog → 存储”流水线。

更准确的模型是一个 DAG（有分叉的数据处理图）：

~~~mermaid
flowchart TD
    APP[Application / SDK]

    APP --> SAMPLE[Sampling Decision]
    SAMPLE --> AGENT[Datadog Agent / OTel Collector]

    AGENT --> TM[APM Trace Metrics]
    AGENT --> INGEST[Datadog Ingestion]

    INGEST --> LIVE[Live Search]
    INGEST --> ENRICH[Datadog Enrichment]

    ENRICH --> PROC[Processing Pipelines]

    PROC --> CM[Custom Metrics from Spans]
    PROC --> RET[Retention]

    RET --> INTEL[Intelligent Retention]
    RET --> DEF[Default Retention Filters]
    RET --> CUS[Custom Retention Filters]

    INTEL --> INDEX[Indexed Dataset]
    DEF --> INDEX
    CUS --> INDEX

    INDEX --> SPAN[Span Explorer]
    INDEX --> TQ[Trace Queries]

    INGEST --> U1[Ingested Usage]
    INDEX --> U2[Indexed Usage]
~~~

一条 Trace 在 Datadog 中会派生出多个不同的数据产品：

1. APM Trace Metrics；
2. Live Search 数据；
3. Indexed Historical Spans；
4. Complete Trace Dataset；
5. Custom Metrics from Spans；
6. Service / Dependency / Entity 信息；
7. Usage / Cost Accounting。

这些数据集的来源、采样方式、生命周期和费用维度并不相同。

---

## 2. Trace Pipeline 的核心数据边界

如果只保留一个整体认知模型，可以记住下面四个边界：

~~~mermaid
flowchart LR
    A[Application Traffic]
    B[Ingested Dataset]
    C[Processed Dataset]
    D[Indexed Dataset]
    E[Aggregated Metrics]

    A --> B --> C --> D
    A --> E
~~~

### Boundary 1：Application → Ingested

由 Sampling 控制。

回答：

> 哪些 Trace / Span 值得进入 Datadog？

### Boundary 2：Ingested → Processed

由 Datadog Enrichment 和 Processing Pipelines 改变、规范 Span 的语义。

### Boundary 3：Processed → Indexed

由 Retention 控制。

回答：

> 哪些已经进入 Datadog 的 Span / Trace 值得长期可搜索？

### Boundary 4：Raw Trace → Metrics

将高容量 Raw Trace 转换为 Hits、Errors、Duration 和 Custom Metrics，用于长期监控、Dashboard、SLO 和 Alert。

---

## 3. 第一阶段：Instrumentation

Trace Pipeline 的输入来自应用和运行时产生的 Span。

例如：

~~~mermaid
flowchart TD
    C[checkout-service<br/>SERVER Span]

    C --> P[payment-service<br/>CLIENT Span]
    C --> DB[PostgreSQL<br/>CLIENT Span]
    C --> K[Kafka<br/>PRODUCER Span]
~~~

这个阶段的 Telemetry 主要属于 Application-side Data Model，需要关注：

- service.name；
- SpanKind；
- operation/span name；
- resource name；
- HTTP/RPC/DB/Messaging semantic attributes；
- Trace Context；
- Peer Attributes。

Datadog Native SDK 的 Span 会进入 Datadog Agent。

OpenTelemetry 常见路径是：

~~~text
OTel SDK
→ OTel Collector / DDOT / Datadog Agent OTLP
→ Datadog
~~~

---

## 4. 第二阶段：Sampling 与 Ingestion

Sampling 是 Trace Pipeline 的第一个真正数据削减点。

~~~mermaid
flowchart LR
    ALL[Application Traces]
    DEC{Sampling Decision}

    ALL --> DEC
    DEC -->|KEEP| ING[Ingested]
    DEC -->|DROP| DROP[Dropped]
~~~

Datadog Native 主要使用：

- Automatic Head Sampling；
- Resource Sampling Rules；
- Adaptive Sampling；
- Error Sampling；
- Rare Sampling；
- Single Span Sampling；
- Manual Keep / Drop。

OpenTelemetry 场景还可能使用：

- SDK Head Sampling；
- Collector Tail Sampling；
- Agent Probabilistic Sampling。

详细机制见：

- [APM 采样策略](./apm-sampling.md)

---

## 5. ingestion_reason：为什么这个 Span 进入 Datadog

Datadog 为 Ingested Span 标记 Ingestion Reason。

常见原因包括：

~~~text
auto
rule
error
rare
manual
single_span
rum
synthetics
otel
~~~

它回答：

> 这条 Span 为什么没有被 Sampling 丢弃？

这个字段对 Sampling Debug、Ingestion Usage Attribution 和 Sampling Rule 验证很有价值。

---

## 6. Sampling Decision Maker 与 sampling_service

分布式 Trace 中，真正决定整条 Trace 是否被采样的 Service 很重要。

例如：

~~~mermaid
flowchart LR
    F[frontend]
    C[checkout]
    P[payment]
    DB[postgres]

    F --> C --> P --> DB
~~~

如果 frontend 是 Root Service，并在 Trace Root 处做 Sampling Decision，那么 Datadog Usage 数据中可以把相关 Ingested Bytes 归因到：

~~~text
sampling_service:frontend
~~~

即使大部分 Span 实际来自 checkout、payment 或 postgres。

这说明成本治理不能只看谁产生最多 Span，还要看谁决定整条 Trace 是否进入 Datadog。

---

## 7. Ingestion Control 本质上是一个反馈控制系统

Datadog 的 Ingestion Controls 可以理解成：

~~~mermaid
flowchart LR
    U[Usage Metrics]
    C[Sampling Control]
    R[Sampling Rules]
    SDK[SDK / Agent]
    T[Trace Traffic]

    U --> C --> R --> SDK --> T --> U
~~~

Adaptive Sampling 把这个反馈环自动化：

~~~text
Monthly Ingestion Target
        ↓
Datadog Controller
        ↓
Dynamic Sampling Rates
        ↓
SDK
~~~

因此 Adaptive Sampling 的本质是：

> 根据 Ingestion Budget 动态调整 Head Sampling Rate。

它不是 Tail Sampling。

---

## 8. APM Trace Metrics 是一条独立分支

Datadog 自动生成的核心 APM Trace Metrics 包括：

~~~text
trace.<span_name>.hits
trace.<span_name>.errors
trace.<span_name>.duration
~~~

这些和 Backend 的 Generate Metrics from Spans 不是同一个系统。

Datadog Native 路线可以概念化为：

~~~mermaid
flowchart LR
    APP[Application Traffic]
    AG[Datadog Agent]

    APP --> AG

    AG --> M[APM Trace Metrics]
    AG --> S[Sampling]
    S --> I[Trace Ingestion]
~~~

因此 Trace 明细可以低比例 Ingest，而 Request/Error/Latency Metrics 仍可尽量基于完整应用流量。

这是 Datadog Native APM 的核心设计之一。

---

## 9. Service Entry Span

Datadog 的 APM Trace Metrics 依赖一个重要概念：

~~~text
Service Entry Span
~~~

可以理解为：

> 一次请求进入某个 Service 的 Span。

例如：

~~~mermaid
flowchart LR
    A[checkout SERVER<br/>Service Entry]
    B[payment CLIENT]
    C[payment SERVER<br/>Service Entry]
    D[postgres CLIENT]

    A --> B --> C --> D
~~~

checkout SERVER 和 payment SERVER 更接近 Service Entry Span。

它们用于生成服务级 RED Metrics：

- Rate；
- Errors；
- Duration。

这也是 Service Page、Resource Page 和 APM Monitor 的重要数据来源。

---

## 10. Service Entry Span 与 OpenTelemetry 的语义差异

Service Entry Span 是 Datadog 的概念，不是 OpenTelemetry 的标准字段。

OpenTelemetry 更标准的是：

~~~text
SpanKind.SERVER
SpanKind.CLIENT
SpanKind.PRODUCER
SpanKind.CONSUMER
~~~

因此 OTel → Datadog 需要完成语义映射：

~~~mermaid
flowchart LR
    OTEL[OTel Span]

    OTEL --> K[SpanKind]
    OTEL --> S[service.name]
    OTEL --> H[HTTP / RPC / Messaging Attributes]

    K --> MAP[Datadog Mapping]
    S --> MAP
    H --> MAP

    MAP --> SE[Service Entry Span]
    MAP --> OP[Operation Name]
    MAP --> RN[Resource Name]
~~~

这意味着：

> OTLP 能够成功发送，并不代表 Datadog 一定能够正确识别 Service Entry Span。

OpenTelemetry 互操作测试必须包含：

- Service Entry Recognition；
- Operation Name；
- Resource Name；
- Service Name；
- Peer Service。

---

## 11. 为什么 OTel Pipeline 强调 span_metrics 在 Sampling 前

OTel + Datadog 推荐架构：

~~~mermaid
flowchart LR
    SDK[OTel SDK]
    COL[Collector]

    SDK --> COL

    COL --> SM[span_metrics]
    SM --> RED[APM Trace Metrics]

    COL --> TS[tail_sampling]
    TS --> DD[Datadog]
~~~

核心规则：

~~~text
span_metrics
必须在 Sampling 之前看到 Trace
~~~

否则：

~~~text
tail_sampling
→ span_metrics
~~~

产生的 hits、errors 和 duration 只反映 Sampled Traffic。

同样会影响：

- Peer Service Metrics；
- Inferred Services；
- Operation/Resource Recognition。

---

## 12. Trace Metrics 与 Custom Metrics from Spans 是两个系统

### 12.1 Automatic Trace Metrics

主要包括：

~~~text
hits
errors
duration
~~~

目标：

- RED Metrics；
- Service Monitoring；
- SLO；
- Alert。

通常尽量基于完整应用流量。

### 12.2 Custom Metrics from Spans

用户可以从 Ingested Span 生成：

~~~text
payment.amount
checkout.item_count
db.rows_returned
~~~

这类 Metric 的输入是 Ingested Spans，不是全部 Application Traffic。

假设：

~~~text
真实请求 = 10000
Ingestion = 10%
~~~

则：

~~~text
Automatic Trace Metrics
理想情况下仍反映约 10000

Custom Metric from Spans
输入可能只有约 1000 个 Ingested Span
~~~

这两个指标系统不能混为一谈。

---

## 13. Datadog Backend Enrichment

Span 到达 Datadog Backend 后，会经过 Datadog Enrichment。

Datadog 可以结合：

- Host；
- Container；
- Kubernetes；
- Cloud；
- CI / Git；
- Source Code Metadata；

添加上下文。

~~~mermaid
flowchart LR
    RAW[Raw Span]
    META[Infrastructure / Cloud / Git Metadata]

    RAW --> E[Datadog Enrichment]
    META --> E

    E --> ENRICHED[Enriched Span]
~~~

Processing Pipeline 处理的是 Enrichment 之后的 Span，而不是单纯原始 OTLP Payload。

---

## 14. Processing Pipelines

Processing Pipelines 可以理解为 Datadog Backend 的 Trace ETL Layer。

当前主要能力包括：

- Attribute Rename；
- Attribute Merge；
- Remove；
- Grok Parsing；
- Normalization。

例如不同团队产生：

~~~text
customer_id
customerId
user.customer.id
~~~

可以统一为：

~~~text
customer.id
~~~

~~~mermaid
flowchart LR
    A[customer_id]
    B[customerId]
    C[user.customer.id]

    A --> R[Remapper]
    B --> R
    C --> R

    R --> N[customer.id]
~~~

因此 Processing Pipelines 可以充当：

> Organization-wide Observability Schema Normalization Layer。

---

## 15. Processing Pipeline 的执行位置

Backend 数据链路可以概念化为：

~~~text
Span Ingest
    ↓
Datadog Enrichment
    ↓
Processing Pipelines
    ↓
Retention / Metrics from Spans
    ↓
Index / Derived Data
~~~

从用户可见的配置语义看，Processing Pipeline 产生的新属性可以被后续 Retention 和 Metrics from Spans 使用。

例如：

~~~text
transactionType
~~~

经过 Processing Pipeline 转成：

~~~text
transaction.type
~~~

后，Retention Filter、Custom Metric 和 Trace Explorer 都可以围绕统一属性工作。

---

## 16. Processing Pipelines 的边界

Processing Pipeline 不是无限制的 Trace Rewrite Engine。

需要注意：

- 在 Datadog Backend 执行；
- 只影响新进入的数据；
- 不 retroactively 修改历史 Span；
- Pipeline 自上而下执行；
- Processor 在 Pipeline 内按顺序执行；
- 当前主要操作 Span Attributes；
- 功能可用性与 Datadog Site / Feature Availability 有关。

更准确的定位是：

> Backend Attribute Transformation / Normalization Layer。

---

## 17. Custom Metrics from Spans 与 Retention 解耦

Custom Metric 的生成不要求 Span 最终被 Index。

~~~mermaid
flowchart LR
    I[Ingested Span]

    I --> M[Custom Metric]
    I --> R{Retention}

    R -->|KEEP| X[Indexed Raw Span]
    R -->|DROP| D[Raw Span 不长期可查]
~~~

因此可以实现：

~~~text
短期保留 Raw Span
+
长期保留 Derived Metric
~~~

这是一种典型的数据生命周期设计。

---

## 18. Live Search

Datadog Trace Explorer 的 Live Search 查询的是：

> 已经 Ingest 的 Span。

当前典型窗口：

~~~text
最近 15 分钟
~~~

必须注意：

~~~text
Live Search = 100% of Ingested Data
~~~

而不是：

~~~text
100% Application Traffic
~~~

因为 Sampling 可能已经在前面丢掉部分 Trace。

---

## 19. 三个不同的“100%”

Trace Pipeline 中经常混淆三个集合：

### 100% Application Traffic

真实请求全集。

### 100% Ingested Traffic

经过 Sampling 后进入 Datadog 的全部 Trace/Span。

### Indexed Dataset

经过 Retention 后长期可查询的数据。

通常：

~~~text
Application Traffic
    >=
Ingested Dataset
    >=
Indexed Dataset
~~~

这是理解 Datadog Trace 数据规模的基础关系。

---

## 20. Retention

Retention 决定：

> 哪些 Ingested Span / Trace 被长期 Index。

详细研究见：

- [APM Trace 数据保留策略](./apm-trace-retention.md)

主要机制：

~~~text
Intelligent Retention
+
Default Retention Filters
+
Custom Retention Filters
~~~

Intelligent Retention 又包括：

~~~text
Diversity Sampling
+
1% Flat Sampling
~~~

---

## 21. Span Explorer 与 Trace Queries

这两个查询能力的数据要求不同。

### Span Explorer

主要基于：

~~~text
Indexed Span
~~~

只要某个 Span 被 Index，就可以历史搜索。

### Trace Queries

需要：

~~~text
Complete Trace Dataset
~~~

因为 Query 可能包含结构关系：

~~~text
checkout => payment
web -> database
service A && downstream service B
~~~

所以 Trace Queries 依赖：

- Intelligent Retention 捕获的完整 Trace；
- Trace-level Retention Filter。

---

## 22. Index Span 与 Index Trace 是两个概念

~~~mermaid
flowchart LR
    SR[Span-level Retention]
    TR[Trace-level Retention]

    SR --> S[Indexed Span]
    S --> SE[Span Explorer]

    TR --> T[All Spans of Trace]
    T --> Q[Trace Queries]
~~~

如果只 Index payment span，你可以历史查询 payment。

但不一定能做完整的 checkout → payment → postgres 结构化 Trace Query。

---

## 23. Retention 与 Trace Visualization 不是同一个概念

某个 Span 被 Index 后，打开它时 Datadog 可能显示完整 Trace Context：

~~~text
checkout
  -> payment
       -> postgres
       -> kafka
~~~

但：

~~~text
能看到 Trace Context
!=
所有 Span 都被长期 Index
~~~

只有真正 Indexed 的 Span 才可以在历史 Span Query 中独立搜索。

---

## 24. Usage Accounting

Trace Pipeline 同时维护自己的使用量和成本数据。

两个最重要的指标：

~~~text
datadog.estimated_usage.apm.ingested_bytes
datadog.estimated_usage.apm.indexed_spans
~~~

### Ingested Bytes

主要受：

- Trace Sampling；
- Trace 数；
- Span 数；
- Span Payload Size；

影响。

### Indexed Spans

主要受：

- Retention Filters；
- Span Rate；
- Trace Rate；
- 完整 Trace Span Count；

影响。

因此：

~~~text
Ingestion Cost
!=
Indexing Cost
~~~

---

## 25. Trace Pipeline 有两个主要成本旋钮

### 降低 Ingested Bytes

主要调：

- Automatic Sampling；
- Adaptive Sampling；
- Resource Sampling Rules；
- OTel Head/Tail Sampling。

### 降低 Indexed Spans

主要调：

- Custom Retention Filter；
- Span Rate；
- Trace Rate；
- Filter Scope。

所以：

> 降低 Retention Rate 不能解决上游 Ingestion Volume。

同样：

> Sampling 太激进，也会导致后续 Retention 没有足够有价值的数据可以选择。

---

## 26. Trace Pipeline 实际产出的多个“数据产品”

可以把一条 Trace Stream 视为同时构建多个数据产品：

~~~mermaid
flowchart TD
    T[Trace Stream]

    T --> P1[APM Trace Metrics]
    T --> P2[Live Trace Search]
    T --> P3[Indexed Historical Spans]
    T --> P4[Complete Trace Dataset]
    T --> P5[Custom Metrics from Spans]
    T --> P6[Entity / Dependency Data]

    P1 --> U1[Monitoring / SLO]
    P2 --> U2[实时 Debug]
    P3 --> U3[历史 Span Analysis]
    P4 --> U4[Trace Queries]
    P5 --> U5[长期业务 / 技术 Metrics]
    P6 --> U6[Service Map / Catalog]
~~~

这些 Dataset 不应该混成一个“Trace 数据库”来理解。

---

## 27. Datadog Native Trace Pipeline

~~~mermaid
flowchart TD
    APP[Datadog SDK]

    APP --> AG[Datadog Agent]

    AG --> TM[Trace Metrics]
    AG --> SAMPLE[Sampling]

    SAMPLE --> ING[Datadog Ingestion]

    ING --> LIVE[Live Search]
    ING --> ENRICH[Enrichment]

    ENRICH --> PROC[Processing Pipelines]

    PROC --> CM[Custom Metrics from Spans]
    PROC --> RET[Retention]

    RET --> INDEX[Indexed Dataset]

    INDEX --> SPAN[Span Explorer]
    INDEX --> TQ[Trace Queries]
~~~

最大的架构特点是：

> Metrics Accuracy 与 Raw Trace Volume 解耦。

---

## 28. OpenTelemetry → Datadog Trace Pipeline

~~~mermaid
flowchart TD
    APP[OTel SDK]

    APP --> COL[OTel Collector]

    COL --> SM[span_metrics]
    SM --> RED[Datadog APM Metrics]

    COL --> TS[Optional tail_sampling]

    TS --> EXP[OTLP / Datadog Export]

    EXP --> DD[Datadog Ingestion]

    DD --> LIVE[Live Search]
    DD --> PROC[Processing]
    PROC --> RET[Retention]
    RET --> IDX[Indexed Dataset]
~~~

这个路线必须重点验证：

- span_metrics 是否位于 Sampling 前；
- Service Entry Span 是否正确识别；
- Operation Name 是否正确；
- Resource Name 是否正确；
- Peer Service 是否正确；
- Inferred Service 是否正确；
- Sampling Priority / Trace Completeness。

---

## 29. OTLP Compatibility 不等于 Semantic Compatibility

对于 OpenTelemetry，真正的问题不是：

> OTLP Export 是否成功？

而是：

~~~mermaid
flowchart TD
    OTLP[OTLP Span]

    OTLP --> SE[Service Entry Recognition]
    OTLP --> OP[Operation Name]
    OTLP --> RN[Resource Name]
    OTLP --> PEER[Peer Service]
    OTLP --> METRIC[RED Metrics]
    OTLP --> RET[Retention]
~~~

即：

> 同一个 OTLP Span 进入 Datadog 后，会被解释成什么 Datadog 语义？

这是 Observability-Lab 后续最值得实验验证的方向之一。

---

## 30. Trace Pipeline 与 Inferred Services

Inferred Services 是 Trace Pipeline 的 Entity Resolution 分支之一。

详细研究见：

- [推断服务（Inferred Services）](./inferred-services.md)

~~~mermaid
flowchart TD
    S[Outbound Span]

    S --> P[Peer Attributes]
    P --> E[Peer Identity Resolution]
    E --> I[Inferred Service]

    S --> M[Dependency Metrics]

    I --> MAP[Service / Dependency Map]
    M --> MAP
~~~

因此 Sampling、Semantic Mapping、span_metrics 都可能影响：

- Peer Entity Coverage；
- Dependency Metrics；
- Service Map。

---

## 31. Trace Pipeline 与 Retention

Retention 不是 Sampling 的附属功能，而是独立的数据生命周期层。

~~~text
Sampling
回答：
哪些 Trace 进入 Datadog？

Retention
回答：
进入后哪些 Trace / Span 长期可查询？
~~~

两者之间还有：

- Enrichment；
- Processing；
- Custom Metrics。

完整逻辑更接近：

~~~text
Sampling
→ Ingestion
→ Enrichment / Processing
→ Metrics / Retention
~~~

---

## 32. Trace Pipeline 与 Metrics

Trace Pipeline 中至少有三种不同 Metrics：

### 32.1 APM Trace Metrics

~~~text
hits
errors
duration
~~~

用于服务 RED Monitoring。

### 32.2 Dependency / Peer Metrics

用于：

- Inferred Services；
- Dependency Map；
- Peer Service Analysis。

### 32.3 Custom Metrics from Spans

用户定义，从 Ingested Span 中生成。

这三类数据：

- 输入数据集不同；
- 计算位置不同；
- Sampling 敏感性不同；
- 生命周期不同。

---

## 33. 一个完整 Payment Trace 示例

假设 POST /checkout 产生：

~~~text
checkout
 -> payment
 -> postgres
 -> kafka
~~~

### Stage 1：Instrumentation

产生 12 个 Span。

### Stage 2：Sampling

Root Sampling Decision：

~~~text
KEEP
~~~

### Stage 3：APM Trace Metrics

生成：

~~~text
checkout hits
checkout errors
checkout duration

payment hits
payment errors
payment duration
~~~

### Stage 4：Ingestion

12 Span 进入 Datadog。

### Stage 5：Enrichment

添加：

~~~text
kube_cluster
pod
host
version
git.commit.sha
~~~

### Stage 6：Processing Pipeline

例如：

~~~text
merchantId
~~~

统一为：

~~~text
merchant.id
~~~

### Stage 7：Custom Metric

从 Ingested Span 生成：

~~~text
payment.amount
~~~

### Stage 8：Retention

例如：

~~~text
@transaction_amount:>10000
Span Rate = 100%
Trace Rate = 100%
~~~

完整 Trace 被 Index。

### Stage 9：历史查询

可以：

~~~text
Span Explorer:
@merchant.id:123

Trace Query:
checkout => payment -> postgres
~~~

---

## 34. 生产环境真正需要设计的是 Data Policy

不应该只配置：

~~~text
Sampling Rate = 10%
~~~

而应该分别回答：

| 问题 | Trace Pipeline 层 |
|---|---|
| 哪些请求需要产生 Span？ | Instrumentation |
| 哪些 Trace 应进入 Datadog？ | Sampling / Ingestion |
| 哪些 Metrics 必须准确？ | Trace Metrics / span_metrics |
| 哪些 Attribute 需要统一？ | Processing Pipelines |
| 哪些信息需要长期趋势？ | Metrics from Spans |
| 哪些 Span 要长期可搜索？ | Span Retention |
| 哪些 Trace 要完整保存？ | Trace-level Retention |
| 谁产生最多 Ingestion Cost？ | Usage Metrics |
| 谁做 Sampling Decision？ | sampling_service |
| 为什么 Span 被 Ingest？ | ingestion_reason |
| 为什么 Span 被保留？ | retained_by |

Trace Pipeline 本质上可以看作：

> Telemetry Data Policy Engine。

---

## 35. 建议的 Observability-Lab 实验

建议后续创建：

~~~text
labs/datadog-trace-pipeline/
~~~

统一 Workload：

~~~mermaid
flowchart LR
    L[Load Generator]
    C[checkout-service]
    P[payment-service]
    DB[(PostgreSQL)]
    K[Kafka]

    L --> C
    C --> P
    P --> DB
    C --> K
~~~

实验维度：

### A. Datadog Native

记录：

- Spans Generated；
- Trace Metrics；
- Ingested Bytes；
- Indexed Spans。

### B. OTel SDK Head Sampling

比较 100% 与 10%，观察：

- APM Metrics；
- Trace Coverage。

### C. span_metrics before/after Sampling

验证 Metrics Accuracy。

### D. Processing Pipeline

统一：

~~~text
merchantId
customer_id
customer.id
~~~

到一个 canonical attribute。

### E. Metrics from Spans

验证 Ingested 但未 Indexed 的 Span 是否仍能生成 Metric。

### F. Retention

比较：

- Intelligent；
- Span-level；
- Trace-level。

### G. Usage

记录：

~~~text
apm.ingested_bytes
apm.indexed_spans
~~~

分析两个成本控制面。

---

## 36. 当前结论

Datadog Trace Pipeline 更准确的理解是：

> **一套将高容量 Trace Stream 转换成多个不同生命周期、不同查询能力、不同费用模型的数据产品的处理系统。**

它完成：

~~~text
Instrumentation
    ↓
Sampling
    ↓
Ingestion
    ↓
Trace Metrics
    ↓
Enrichment
    ↓
Processing
    ↓
Custom Metrics
    ↓
Retention / Index
    ↓
Search / Query
    ↓
Usage / Cost Accounting
~~~

对于 OpenTelemetry 场景，还要增加：

~~~text
OTel Semantic Conventions
    ↓
Datadog Semantic Mapping
~~~

所以真正的互操作问题不是：

> Datadog 是否支持 OTLP？

而是：

> **Datadog 能否把 OTel Span 正确映射成它自己的 Service、Resource、Operation、Peer、Metrics、Retention 和 Entity 模型？**

这也是 Observability-Lab 后续 Datadog / OpenTelemetry 实验最值得持续验证的主线。

---

## 相关调研

- [APM 采样策略](./apm-sampling.md)
- [APM Trace 数据保留策略](./apm-trace-retention.md)
- [推断服务（Inferred Services）](./inferred-services.md)

---

## 参考资料

### Datadog Trace Pipeline

- Trace Pipeline  
  https://docs.datadoghq.com/tracing/trace_pipeline/
- Ingestion Mechanisms  
  https://docs.datadoghq.com/tracing/trace_pipeline/ingestion_mechanisms/
- Ingestion Controls  
  https://docs.datadoghq.com/tracing/trace_pipeline/ingestion_controls/
- Processing Pipelines  
  https://docs.datadoghq.com/tracing/trace_pipeline/processing_pipelines/
- Generate Metrics from Spans  
  https://docs.datadoghq.com/tracing/trace_pipeline/generate_metrics/
- Trace Retention  
  https://docs.datadoghq.com/tracing/trace_pipeline/trace_retention/
- Trace Pipeline Metrics  
  https://docs.datadoghq.com/tracing/trace_pipeline/metrics/

### Datadog APM

- APM Trace Metrics  
  https://docs.datadoghq.com/tracing/metrics/metrics_namespace/
- Trace Explorer  
  https://docs.datadoghq.com/tracing/trace_explorer/
- Trace Queries  
  https://docs.datadoghq.com/tracing/trace_explorer/trace_queries/
- Trace Queries Dataset  
  https://docs.datadoghq.com/tracing/guide/trace_queries_dataset/

### Datadog + OpenTelemetry

- OpenTelemetry Trace Metrics  
  https://docs.datadoghq.com/opentelemetry/integrations/trace_metrics/
- Service Entry Span Mapping  
  https://docs.datadoghq.com/opentelemetry/mapping/service_entry_spans/
- OpenTelemetry Ingestion Sampling  
  https://docs.datadoghq.com/opentelemetry/ingestion_sampling/
