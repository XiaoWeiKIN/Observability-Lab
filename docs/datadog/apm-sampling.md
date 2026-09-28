# Datadog APM 采样策略调研

> 调研日期：2026-09-28  
> 范围：Datadog APM Trace Sampling，包括 Datadog SDK/Agent、Ingestion Controls、Retention、OpenTelemetry SDK/Collector/Agent OTLP Ingest 等路径。

## 1. 摘要

Datadog APM 的“采样”不是单一算法，而是一个分层的数据控制体系。

最重要的区分是：

~~~text
Ingestion Sampling
=
决定哪些 Trace / Span 被发送并进入 Datadog

Retention Sampling
=
决定已经进入 Datadog 的数据中，哪些被索引并长期保留
~~~

两者解决的问题不同，发生的位置也不同。

可以用下面的模型理解整个 Trace Pipeline：

~~~mermaid
flowchart LR
    A[应用 / SDK]
    B[Head-based Sampling]
    C[Datadog Agent / OTel Collector]
    D[补充或高级 Sampling]
    E[Datadog Ingestion]
    F[Live Search<br/>最近 15 分钟]
    G[Retention Filters]
    H[Indexed Data<br/>15 天]

    A --> B --> C --> D --> E
    E --> F
    E --> G --> H

    C --> M[APM Trace Metrics]
~~~

在 Datadog 原生 tracing 中，主要模型是：

~~~text
Head-based Sampling
+
Error / Rare / Single Span 等补偿机制
+
Retention Filters
~~~

在 OpenTelemetry 场景中，还可以引入：

~~~text
OTel SDK Head Sampling
OTel Collector Tail Sampling
Datadog Agent Probabilistic Sampling
~~~

因此研究 Datadog APM Sampling 时，至少要同时考虑：

- Trace 完整性；
- APM Metrics 是否仍然基于全量流量；
- Ingestion Volume；
- Indexed Volume；
- Error / Rare Trace 可见性；
- 分布式 Trace 的上下文传播；
- OpenTelemetry Pipeline 中 Sampling 的位置。

---

## 2. Ingestion 与 Retention 是两个独立阶段

Datadog Trace Pipeline 明确把 Ingestion 和 Retention 分开。

### 2.1 Ingestion

Ingestion Controls 控制：

> 哪些 Trace 被应用和 Agent/Collector 发送到 Datadog。

主要影响：

- Ingested Spans；
- Ingested Bytes；
- Ingested Traces；
- Trace Explorer 最近 15 分钟能看到的数据；
- APM ingestion 使用量。

### 2.2 Retention

Trace 已经进入 Datadog 后，Retention Filters 再决定：

> 哪些 Span 被索引并保留 15 天。

概念上：

~~~mermaid
flowchart LR
    I[Ingested Trace]
    I --> L[Live Search<br/>15 分钟]
    I --> R[Retention Filters]
    R --> X[Indexed Span / Trace<br/>15 天]
~~~

因此：

~~~text
已 Ingest
≠
一定长期保留
~~~

同样：

~~~text
Retention Filter
不会降低上游 Ingestion Volume
~~~

如果目标是降低 Trace 摄取量或成本，需要调整 Ingestion Sampling，而不是只修改 Retention Filter。

---

## 3. Datadog 默认模型：Head-based Sampling

Datadog 默认的 Trace Sampling 机制是 **Head-based Sampling**。

决策发生在 Trace 根部开始时：

~~~mermaid
sequenceDiagram
    participant A as Service A
    participant B as Service B
    participant C as Service C

    Note over A: Trace 开始<br/>决定 KEEP / DROP
    A->>B: Trace Context + Sampling Decision
    B->>C: Trace Context + Sampling Decision
~~~

它的核心特点是：

- 决策发生得早；
- 还不知道 Trace 最终是否报错；
- 还不知道最终 latency；
- 可以在分布式上下文中传播 sampling decision；
- 正常情况下整条 Trace 一致地 KEEP 或 DROP；
- CPU、网络和 ingestion 成本较低。

这也是 Datadog 原生 tracing 的主要基础。

---

## 4. Agent Automatic Sampling

如果没有在 SDK 中配置 sampling rules，Datadog Agent 会持续计算采样率，再把这些采样率提供给 Datadog SDK。

默认目标：

~~~text
10 traces / second / Agent
~~~

配置：

~~~yaml
apm_config:
  target_traces_per_second: 10
~~~

环境变量：

~~~bash
DD_APM_TARGET_TPS=10
~~~

对应的 ingestion reason：

~~~text
auto
~~~

### 4.1 这是目标值，不是硬限制

Datadog 文档明确指出：

~~~text
10 traces/s
~~~

是一个动态目标，不是严格 Rate Limit。

流量突增时，短时间实际发送的 Trace 数量可能明显高于这个数字。

### 4.2 不同服务获得不同采样率

例如：

~~~text
Service A: 1000 req/s
Service B: 20 req/s
~~~

Agent 不一定给两者相同的 Sampling Rate。

它会根据流量动态调整，使整体尽量靠近目标 TPS。

因此可能出现：

~~~text
Service A -> 0.7%
Service B -> 100%
~~~

具体比例是动态计算的，不应把这个例子理解为固定算法。

### 4.3 低流量服务通常保留更多

Datadog 文档说明：

- 低流量应用可以应用 100% Sampling；
- 高流量应用会降低 Sampling Rate；
- 整体目标仍是每个 Agent 约 10 条完整 Trace/s。

这比全局固定 1% 或 10% 更适合流量差异大的微服务环境。

### 4.4 只适用于 Datadog SDK

需要特别注意：

> DD_APM_TARGET_TPS 控制 Datadog SDK 的默认采样机制，并不会自动控制 OpenTelemetry SDK 的采样。

这也是 Datadog Native 与 OTel Pipeline 的重要区别之一。

---

## 5. SDK Resource-based Sampling Rules

当默认 Automatic Sampling 不能表达业务优先级时，可以使用 SDK Sampling Rules。

常用配置：

~~~text
DD_TRACE_SAMPLING_RULES
~~~

例如：

~~~json
[
  {
    "service": "checkout-service",
    "resource": "GET /health",
    "sample_rate": 0.001
  },
  {
    "service": "checkout-service",
    "resource": "POST /payment",
    "sample_rate": 1.0
  },
  {
    "service": "checkout-service",
    "sample_rate": 0.2
  }
]
~~~

可以理解为：

~~~text
GET /health      -> 0.1%
POST /payment    -> 100%
其他 checkout    -> 20%
~~~

这仍然属于 **Head Sampling**，因为决定仍在 Trace Root 创建时发生。

对应：

~~~text
ingestion_reason: rule
~~~

Datadog 当前推荐优先使用：

~~~text
DD_TRACE_SAMPLING_RULES
~~~

而不是旧的全局：

~~~text
DD_TRACE_SAMPLE_RATE
~~~

后者在相关文档中已经标记为 deprecated，推荐使用一条全局 sampling rule 替代。

---

## 6. Sampling Rule Rate Limiter

SDK Sampling Rules 还有第二层控制：

~~~text
DD_TRACE_RATE_LIMIT
~~~

默认：

~~~text
100 traces / second / service instance
~~~

只有在使用 SDK sampling rules 或 global SDK sampling rate 时，这个 Rate Limiter 才有意义。

例如：

~~~text
POST /payment
10000 req/s

sample_rate = 1.0
DD_TRACE_RATE_LIMIT = 100
~~~

逻辑上：

~~~mermaid
flowchart LR
    A[10000 traces/s]
    B[Sampling Rule<br/>100%]
    C[Rate Limiter<br/>100 traces/s]
    D[发送]

    A --> B --> C --> D
~~~

因此：

> sample_rate=1.0 并不一定意味着无限制地 ingest 100% Trace。

使用 Agent 默认 automatic sampling 时，这个 SDK rule rate limiter 被忽略。

---

## 7. Sampling Decision 的传播

Head Sampling 要在 distributed trace 中有效，必须传播采样决定。

Datadog 自己的传播字段包括：

~~~text
x-datadog-sampling-priority
~~~

如果使用 W3C Trace Context，也会涉及：

~~~text
traceparent
sampled flag
tracestate
~~~

概念上：

~~~mermaid
flowchart LR
    A[checkout-service<br/>KEEP]
    B[payment-service<br/>KEEP]
    C[inventory-service<br/>KEEP]

    A --> B --> C
~~~

Sampling Decision 一旦被传播到下游，整个 Trace 才能保持一致。

如果不同服务独立决定 KEEP/DROP，容易产生：

~~~text
Partial Trace
~~~

因此在分布式 tracing 中，Sampling 不只是“概率问题”，还涉及 **Sampling Decision Propagation**。

---

## 8. Head Sampling 的天然缺陷

Head Sampling 的问题是：

> 做 Sampling Decision 时还不知道请求最终发生了什么。

例如：

~~~mermaid
flowchart LR
    A[POST /payment 开始]
    B[Head Sampling<br/>DROP]
    C[执行 8 秒]
    D[HTTP 500]

    A --> B --> C --> D
~~~

当请求最终变成：

~~~text
Error
Slow Trace
Rare Failure
~~~

原始 Head Sampling Decision 已经做完。

Datadog 因此提供了 Error Sampler、Rare Sampler、Single Span Sampling 等机制来补偿这种信息损失。

---

## 9. Error Sampling

对于没有被 Head Sampling 保留的 Trace，Datadog Agent 可以额外捕获包含 Error Span 的本地 Trace 片段。

默认上限：

~~~text
10 error traces / second / Agent
~~~

配置：

~~~bash
DD_APM_ERROR_TPS=10
~~~

对应：

~~~text
ingestion_reason: error
~~~

处理逻辑：

~~~mermaid
flowchart TD
    A[Trace]
    B{Head Sampling}
    C[Keep]
    D[Drop]
    E{包含 Error Span?}
    F[Error Sampler]
    G[真正丢弃]

    A --> B
    B -->|KEEP| C
    B -->|DROP| D
    D --> E
    E -->|是| F
    E -->|否| G
~~~

### 9.1 Error Sampler 的限制

Datadog 文档明确指出：

> Error Sampler 捕获的是 Agent 本地观察到的 error trace 片段。

如果一个分布式 Trace 跨越多个 Host/Agent：

~~~mermaid
flowchart LR
    A[Service A / Agent A]
    B[Service B / Agent B]
    C[Service C / Agent C<br/>Error]

    A --> B --> C
~~~

Agent C 可以保留包含 Error 的本地部分，但不能保证：

~~~text
A -> B -> C
~~~

整条 distributed trace 都被恢复。

因此：

~~~text
Error Sampling
!=
Distributed Tail Sampling
~~~

此外，默认情况下，被 SDK Sampling Rule 或 manual.drop 明确丢弃的 Span 不会被 Error Sampler 再捕获，除非启用对应 Agent feature。

---

## 10. Rare Sampling

Rare Sampler 用于补偿低频 service/resource 在 Head Sampling 中容易被遗漏的问题。

默认最大：

~~~text
5 rare traces / second / Agent
~~~

但该功能默认：

~~~text
Disabled
~~~

可通过：

~~~bash
DD_APM_ENABLE_RARE_SAMPLER=true
~~~

启用。

Datadog 会根据若干属性组合判断“rare”，相关属性包括：

- env；
- service；
- name；
- resource；
- error.type；
- http.status。

典型场景：

~~~text
GET /health    -> 1000000 req/day
POST /refund   -> 10 req/day
~~~

如果只依赖随机/比例 Head Sampling，低频的 refund 请求可能完全看不到。

Rare Sampling 的目标是：

> 给低频 service/resource 留下最低限度的诊断样本。

与 Error Sampler 一样，它也是 Agent Local Sampling，不能保证恢复完整 distributed trace。

---

## 11. Error / Rare Sampling 与 SDK Rules 的关系

Datadog 文档说明：

> 对已经设置 library sampling rules 的服务，Error Sampler 和 Rare Sampler 默认不会按照普通 automatic sampling 路径工作。

这是一个容易忽略的地方。

一旦通过 SDK Rules 显式控制：

~~~text
DD_TRACE_SAMPLING_RULES
~~~

就应该重新检查：

- Error Trace 是否仍被预期地保留；
- Rare Trace 是否仍然可见；
- Rule 本身是否已经覆盖业务需要。

不能假设：

~~~text
Head Rule + Error Sampler + Rare Sampler
~~~

在所有配置下都会自动叠加。

---

## 12. Single Span Sampling

Datadog 还支持：

~~~text
Single Span Sampling
~~~

用于：

> 整条 Trace 被 Head Sampling 丢弃时，仍然额外保留符合条件的某些 Span。

常见配置：

~~~text
DD_SPAN_SAMPLING_RULES
~~~

例如：

~~~json
[
  {
    "service": "payment-service",
    "name": "http.request",
    "sample_rate": 0.5,
    "max_per_second": 50
  }
]
~~~

概念上：

~~~mermaid
flowchart TD
    T[Trace: DROP]
    T --> A[普通 Span]
    T --> B[SQL Span]
    T --> C[Payment Span]
    C --> K[Single Span Sampling<br/>KEEP]
~~~

它适用于：

- 特定 Operation；
- 特定关键依赖；
- 特定类型 Span；
- 不值得保留完整 Trace，但仍希望分析某类 Span 的场景。

需要注意：

> Single Span Sampling 主要用于从已 Drop 的 Trace 中额外 KEEP 某些 Span，而不是从已经 KEEP 的完整 Trace 中继续删除 Span。

---

## 13. Manual Keep / Drop

部分 Datadog SDK 还支持应用代码主动控制 Sampling Priority，例如：

~~~text
manual.keep
manual.drop
~~~

适合业务已经明确知道重要性的情况。

例如：

~~~text
高价值支付交易
-> manual.keep

health check
-> manual.drop
~~~

概念上：

~~~mermaid
flowchart TD
    A[Trace]
    A --> B{业务条件}
    B -->|关键交易| C[Manual KEEP]
    B -->|明确无价值| D[Manual DROP]
~~~

这种方法的问题是侵入业务代码，同时要求 Sampling Decision 尽可能在 Trace 传播到下游之前完成，否则可能产生 Partial Trace。

---

## 14. Adaptive Sampling

Adaptive Sampling 是 Datadog 当前值得重点研究的机制之一。

它解决的问题不是：

~~~text
某一条 Trace 最终是不是 Error？
~~~

而是：

~~~text
如何自动控制每月 Trace Ingestion Volume？
~~~

用户配置一个月度 Trace Ingestion Target，例如：

~~~text
500 GB / month
~~~

Datadog 根据实际流量不断调整不同：

~~~text
environment
service
resource
~~~

组合的 sampling rate。

概念上：

~~~mermaid
flowchart LR
    B[月度 Ingestion Target]
    T[实际 Traffic]
    C[Adaptive Sampling Controller]

    B --> C
    T --> C

    C --> A1[/health<br/>低比例]
    C --> A2[/checkout<br/>中等比例]
    C --> A3[/payment<br/>较高比例]
~~~

### 14.1 动态重算

Datadog 当前文档说明：

~~~text
Sampling Rate 大约每 10 分钟重新计算
~~~

### 14.2 低流量可见性

Adaptive Sampling 会尽量保证：

~~~text
每个 service + resource + environment
至少每 5 分钟捕获一条 Trace
~~~

这用于避免低流量 endpoint 被完全采不到。

### 14.3 本质仍然是 Head Sampling

Adaptive Sampling 使用 Remote Configuration 和现有 Sampling Rules 机制动态改变 Rate。

因此：

~~~text
Adaptive Sampling
=
Feedback-controlled Head Sampling
~~~

它不是 Tail Sampling。

它在 Trace 开始时仍然不知道：

- 最终 Error；
- 最终 latency；
- 下游是否失败。

---

## 15. Sampling 配置优先级

当多个地方同时设置 Sampling Configuration 时，Datadog 当前的优先级从高到低为：

1. Remote Resource-based Sampling Rules；
2. Adaptive Sampling Rules；
3. Local DD_TRACE_SAMPLING_RULES；
4. Remote Global Sampling Rate；
5. Local DD_TRACE_SAMPLE_RATE；
6. Agent DD_APM_TARGET_TPS。

可以简化成三个规则：

~~~text
Tracer Settings > Agent Settings

Sampling Rules > Global Sampling Rate

Remote > Local
~~~

这对排查“为什么 Sampling Rate 没生效”非常重要。

例如：

~~~text
DD_APM_TARGET_TPS=20
~~~

没有生效，不一定是 Agent 配置错误，也可能是更高优先级的 Remote Rule 已经覆盖了它。

---

## 16. Ingestion Controls 与 APM Metrics

Datadog Native APM 一个非常重要的设计是：

> Ingestion Controls 不影响 APM Metrics 的计算。

Datadog 文档明确说明：

~~~text
APM Metrics are always calculated based on all traces
and are not impacted by ingestion controls.
~~~

因此可以出现：

~~~text
APM Metrics:
100% traffic

Trace Ingestion:
5% / 10% / 动态比例
~~~

这意味着可以同时获得：

- 比较完整的 Request Count；
- Error Count；
- Duration / Latency Metrics；
- 较小的 Trace Ingestion Volume。

概念上：

~~~mermaid
flowchart TD
    A[应用流量]
    A --> M[APM Metrics<br/>全量统计]
    A --> S[Ingestion Sampling]
    S --> T[部分 Trace]
~~~

这也是 Datadog 原生 APM Sampling 模型的重要价值。

---

## 17. 为什么不需要保存 100% Trace

假设服务有：

~~~text
100000 requests/s
~~~

监控层面需要知道：

~~~text
Request Rate
Error Rate
Latency Distribution
~~~

这些更适合用 Metrics 表示。

Trace 更适合：

~~~text
具体请求 Debug
Root Cause Analysis
调用链分析
异常案例定位
~~~

因此架构可以是：

~~~mermaid
flowchart TD
    A[100% Application Traffic]
    A --> M[APM Metrics<br/>趋势与监控]
    A --> S[Sampling]
    S --> T[代表性 Trace<br/>诊断]
~~~

这是一种：

~~~text
全量聚合统计
+
部分高价值明细
~~~

的数据模型。

---

## 18. OpenTelemetry SDK Head Sampling

OpenTelemetry SDK 自己也支持 Head Sampling，例如：

~~~text
TraceIdRatioBased
ParentBased
~~~

典型配置思想：

~~~text
ParentBased(
    TraceIdRatioBased(0.1)
)
~~~

这意味着在应用 SDK 内部只留下 10% Trace。

优点：

- 最早降低数据量；
- 降低 SDK → Collector 网络开销；
- 降低 Collector CPU/Memory；
- 实现简单。

但它带来一个对 Datadog 非常重要的影响：

> Datadog 或 Collector 从未看到被 SDK Drop 的 90% Trace。

因此如果下游 APM Metrics 根据这些 Span 计算，那么 Metrics 也只能基于已采样流量。

Datadog 官方 OpenTelemetry 文档明确提醒：

> SDK Head Sampling 会影响 APM Metrics，因为只有 sampled traces 会被发送到 Collector 或 Agent。

这和 Datadog Native 的 Ingestion Controls 模型不同。

---

## 19. OpenTelemetry Collector Tail Sampling

如果希望根据完整 Trace 的最终结果再采样，可以使用：

~~~text
OpenTelemetry Collector tail_sampling processor
~~~

它可以表达类似：

~~~text
Error Trace                -> 100%
Latency > 2s               -> 100%
特定 Attribute             -> 100%
普通请求                    -> 1%
~~~

概念上：

~~~mermaid
flowchart LR
    A[Service A]
    B[Service B]
    C[Service C]

    A --> G[Collector Gateway]
    B --> G
    C --> G

    G --> W[等待 Trace]
    W --> D{Tail Sampling Decision}

    D -->|Error| K1[KEEP]
    D -->|Slow| K2[KEEP]
    D -->|Normal| P[概率采样]
~~~

和 Head Sampling 相比，Tail Sampling 最大的优势是决策时已经知道更多信息。

但代价也明显：

- Collector 必须暂存 Span；
- Memory 使用更高；
- 决策延迟增加；
- Collector Scale-out 更复杂；
- Trace 必须正确聚合到同一个 Sampling 节点。

---

## 20. Tail Sampling 的关键：Trace Affinity

假设同一条 Trace 的 Span 被随机分散：

~~~mermaid
flowchart LR
    A[Span A] --> C1[Collector 1]
    B[Span B] --> C2[Collector 2]
    C[Span C] --> C3[Collector 3]
~~~

那么没有任何 Collector 拥有完整 Trace。

Tail Sampling 的判断就可能失真。

因此分布式 Tail Sampling 需要：

~~~text
trace_id-aware routing
~~~

使同一 Trace 的所有 Span 到达同一个 Sampling Collector。

概念上：

~~~mermaid
flowchart LR
    A[Span A]
    B[Span B]
    C[Span C]

    A --> R[Trace-aware Routing]
    B --> R
    C --> R

    R --> G[同一个 Gateway Collector]
~~~

Datadog OpenTelemetry 文档明确指出：

> Distributed Tail Sampling 应采用 Gateway Deployment，并保证同一 Trace 的所有 Span 路由到同一个 Gateway Collector。

---

## 21. OTel + Datadog：Sampling 前先计算 Trace Metrics

如果使用 Collector Tail Sampling，同时还希望 Datadog APM Metrics 尽可能基于完整流量，那么处理顺序至关重要。

推荐概念：

~~~mermaid
flowchart LR
    A[OTel SDK<br/>不提前丢 Trace]
    C[OTel Collector]
    SM[span_metrics]
    M[APM Trace Metrics]
    TS[tail_sampling]
    DD[Datadog]

    A --> C
    C --> SM --> M
    C --> TS --> DD
~~~

核心原则：

~~~text
span_metrics
必须看到 Sampling 之前的 Trace
~~~

即：

~~~text
span_metrics
-> sampling
~~~

而不是：

~~~text
sampling
-> span_metrics
~~~

Datadog 当前文档明确要求推荐的 span_metrics connector 在 Sampling Processor 之前收到所有 Trace，这样：

~~~text
hits
errors
duration
~~~

才能反映未采样流量。

这同样影响我们前面研究的：

~~~text
Peer Services
Inferred Services
Dependency Metrics
~~~

因为 Datadog 使用 span_metrics dimensions 推导 peer service、resource name、operation name 等信息。

---

## 22. Datadog Agent OTLP Probabilistic Sampling

当 OpenTelemetry SDK 直接把 OTLP Trace 发给 Datadog Agent 时，可以启用 Agent 侧 Probabilistic Sampler。

Datadog 当前要求：

~~~text
Agent >= 7.70.0
~~~

虽然该功能更早已经引入，但 Datadog 明确警告：

> 7.70.0 以前的 Agent 在 OTLP Ingest 中启用 probabilistic sampler 时，可能丢弃所有传入的 OTLP Trace。

配置示例：

~~~yaml
apm_config:
  probabilistic_sampler:
    enabled: true
    sampling_percentage: 10
    hash_seed: 22
~~~

环境变量示例：

~~~bash
DD_APM_PROBABILISTIC_SAMPLER_ENABLED=true
DD_APM_PROBABILISTIC_SAMPLER_SAMPLING_PERCENTAGE=10
~~~

这类 Sampling 基于 Trace ID 进行确定性概率决策。

---

## 23. Agent Probabilistic Sampling 与 SDK Sampling 的组合风险

Datadog 文档特别指出：

> Agent Probabilistic Sampler 会忽略 SDK 设置的 sampling priority。

因此下面这种结构需要谨慎：

~~~mermaid
flowchart LR
    SDK[OTel SDK<br/>Head Sampling]
    Agent[Datadog Agent<br/>Probabilistic Sampling]
    DD[Datadog]

    SDK --> Agent --> DD
~~~

因为可能形成两层独立 Sampling。

例如：

~~~text
SDK: KEEP
Agent Probabilistic Sampler: DROP
~~~

最终仍然会 Drop。

因此不建议把这两层简单叠加，并假设 SDK 的 KEEP Priority 会被 Agent 保留。

---

## 24. Retention Sampling

> Retention 已拆分为独立调研文档：[APM Trace 数据保留策略](./apm-trace-retention.md)。本节只保留 Sampling 文档所需的概要。

Trace 成功 Ingest 后，进入第二套采样系统：

~~~text
Retention
~~~

Retention Filter 决定哪些 Span 被索引并进入历史可查询数据集。具体 Retention Duration 取决于数据类型、Retention 机制和客户套餐；当前 Datadog Data Retention Periods 文档显示 APM Indexed Spans 通常为 15 或 30 天。

它不会改变：

~~~text
Agent / Collector 已经发送了多少数据
~~~

概念上：

~~~mermaid
flowchart LR
    I[Ingested Span]
    L[Live Search<br/>15 分钟]
    F[Retention Filters]
    X[Indexed<br/>长期保留]

    I --> L
    I --> F --> X
~~~

Datadog Trace Explorer 的 Live Search 对最近 15 分钟的 ingested spans 提供实时查询，在进入 retention/indexing 之前即可看到。

---

## 25. Intelligent Retention

Datadog 默认启用：

~~~text
Intelligent Retention Filter
~~~

它主要包括：

1. Diversity Sampling；
2. 1% Flat Sampling。

### 25.1 Diversity Sampling

Diversity Sampling 会尝试给不同组合保留具有代表性的 Trace。

主要维度包括：

~~~text
environment
service
operation
resource
~~~

同时考虑：

- latency 分布；
- Error；
- 不同 response status code。

Datadog 文档说明，它会周期性为不同 resource 保留代表性的 p75、p90、p95 latency 样本，以及 errors/status 的代表样本。

它的目标不是：

~~~text
统计学上的完全随机样本
~~~

而是：

~~~text
诊断多样性
~~~

### 25.2 1% Flat Sampling

Intelligent Retention 还包含一个：

~~~text
1% uniform sample
~~~

该采样基于：

~~~text
trace_id
~~~

因此同一条 Trace 的 Span 会得到一致的保留决策。

这个数据集更适合：

- General Health；
- Trend Analysis；
- Trace Queries。

Datadog 当前说明，Intelligent Retention 产生的 indexed spans 不计入 indexed span usage。

---

## 26. Custom Retention Filters

可以根据任意 Span Tag / Attribute 创建自定义 Retention Filter。

例如：

~~~text
status:error
~~~

或：

~~~text
resource_name:"POST /payment"
~~~

并设置：

~~~text
Retention Rate = 100%
~~~

也可以设置：

~~~text
Retention Rate = 10%
~~~

Datadog 会从匹配的 Span 中均匀选择对应比例。

Retention Filters 有顺序：

> 一个 Span 一旦命中了前面的 Retention Filter 并被 KEEP/DROP，后续 Filter 不会再处理它。

因此 Filter Order 本身也是 Retention Policy 的组成部分。

---

## 27. Datadog APM 的双层 Sampling Pipeline

综合起来，可以把 Datadog Sampling 理解成：

~~~mermaid
flowchart TD
    A[Application Traffic]

    A --> H[Head Sampling]

    H -->|KEEP| AG[Agent / Collector]
    H -->|DROP| R[Error / Rare / Single Span 等补偿]

    R --> AG

    AG --> M[APM Metrics]
    AG --> I[Datadog Ingestion]

    I --> L[Live Search<br/>15 分钟]

    I --> IR[Intelligent Retention]
    I --> CR[Custom Retention Filters]

    IR --> X[Indexed Data<br/>15 天]
    CR --> X
~~~

因此需要把两个问题分开问：

### 问题 A

~~~text
我们希望发送多少 Trace 到 Datadog？
~~~

这是：

~~~text
Ingestion Sampling
~~~

### 问题 B

~~~text
已经发送的数据里，我们希望哪些 Trace 在 Live Search 窗口之后仍然可以历史查询？
~~~

这是：

~~~text
Retention Sampling
~~~

---

## 28. Head Sampling 与 Tail Sampling 对比

| 维度 | Head Sampling | Tail Sampling |
|---|---|---|
| 决策时间 | Trace 开始时 | Trace 聚合后 |
| 是否知道最终 Error | 否 | 是 |
| 是否知道最终 Latency | 否 | 是 |
| 应用到 Collector 前即可降低流量 | 是 | 否 |
| Collector Memory 成本 | 低 | 高 |
| 分布式部署复杂度 | 低 | 高 |
| Trace Affinity 要求 | 较低 | 高 |
| Datadog Native 主模型 | 是 | 不是主要原生路径 |
| OTel SDK | 支持 | 不支持完整 Tail Decision |
| OTel Collector | 可做概率/其他 Processor | 支持 Tail Sampling |

---

## 29. Ingestion 与 Retention 对比

| 维度 | Ingestion | Retention |
|---|---|---|
| 主要发生位置 | SDK / Agent / Collector | Datadog Backend |
| 控制目标 | 发多少 Trace | 长期索引多少 |
| 影响 Ingestion Volume | 是 | 否 |
| 影响最近 15 分钟 Live Search 数据 | 是 | 否 |
| 影响历史 Indexed Search | 间接 | 直接 |
| 典型机制 | Head / Tail / Probabilistic | Intelligent / Custom Filters |

可用于观察使用量的 Datadog Metrics 包括：

~~~text
datadog.estimated_usage.apm.ingested_bytes
datadog.estimated_usage.apm.ingested_spans
datadog.estimated_usage.apm.ingested_traces
datadog.estimated_usage.apm.indexed_spans
~~~

---

## 30. Datadog Native 与 OpenTelemetry 的关键区别

### 30.1 Datadog Native

Datadog 文档保证：

~~~text
Ingestion Controls
不会影响 APM Metrics
~~~

因此逻辑上可以获得：

~~~text
全量 APM Metrics
+
部分 Trace Ingestion
~~~

这非常适合大流量生产系统。

### 30.2 OTel SDK Head Sampling

如果：

~~~text
OTel SDK
先 Drop 90%
~~~

那么后面的：

~~~text
Collector
Datadog Agent
Datadog Backend
~~~

都不可能再看到这 90%。

因此通过这些 Trace 计算的 APM Metrics 也会受采样影响。

### 30.3 OTel Collector Tail Sampling

更合理的高级模型是：

~~~mermaid
flowchart LR
    A[OTel SDK<br/>尽量不提前丢]
    C[Collector]

    A --> C

    C --> SM[span_metrics]
    SM --> M[APM Metrics]

    C --> TS[tail_sampling]
    TS --> DD[Datadog Trace Ingestion]
~~~

这样可以同时实现：

~~~text
Sampling 前计算 Metrics
+
根据完整 Trace 做 Tail Sampling
~~~

---

## 31. 对 Inferred Services 的影响

Sampling 与前面的 Inferred Services 调研直接相关。

Datadog 的推荐 span_metrics dimensions 会用于推导：

- Peer Services；
- Host Tags；
- Operation Names；
- Resource Names。

因此：

~~~mermaid
flowchart TD
    A[100% Trace]
    A --> SM[span_metrics]
    SM --> M[APM Metrics]
    SM --> P[Peer / Inferred Service Stats]

    A --> S[Sampling]
    S --> T[Sampled Traces]
~~~

如果 Sampling 位于 span_metrics 之前：

~~~text
Sampling
-> span_metrics
~~~

那么：

~~~text
Trace Metrics
Dependency Metrics
Peer Service Stats
~~~

都会只基于 Sampled Traffic。

如果：

~~~text
span_metrics
-> Sampling
~~~

则这些聚合可以基于 Sampling 前的数据。

---

## 32. 生产策略建议框架

这里不定义唯一“正确”的采样率，而是给出不同架构下的设计框架。

### 32.1 Datadog Native

可以从下面的组合开始评估：

~~~text
Automatic / Adaptive Head Sampling
+
关键 Resource Sampling Rules
+
Error Sampling
+
按需启用 Rare Sampling
+
必要的 Single Span Sampling
+
Retention Filters
~~~

例如：

~~~text
/health
-> 极低 Sampling Rate

/payment
-> 高 Sampling Rate

其他 Endpoint
-> Adaptive Sampling
~~~

### 32.2 OTel + Datadog，简单部署

~~~text
OTel SDK
-> Datadog Agent OTLP
-> Datadog
~~~

此时需要明确选择：

- SDK Head Sampling；
- Agent Probabilistic Sampling；
- 或尽量不在 SDK 过早采样。

不要无意识地叠加多个独立 Sampler。

### 32.3 OTel + Datadog，高级 Sampling

推荐研究：

~~~mermaid
flowchart LR
    Apps[OTel Applications]
    Gateway[OTel Gateway]

    Apps --> Gateway

    Gateway --> Stats[span_metrics]
    Stats --> M[Datadog APM Metrics]

    Gateway --> Tail[tail_sampling]
    Tail --> DD[Datadog]
~~~

Tail Policy 可以实验：

~~~text
Error                        100%
Latency > 2s                 100%
关键业务 transaction         100%
Rare Resource                 50%
普通流量                       1%
~~~

这些比例应该根据实际流量、成本目标和诊断要求测量，而不是直接作为生产默认值。

---

## 33. Observability-Lab 建议实验

建议创建：

~~~text
labs/datadog-apm-sampling/
~~~

统一 workload：

~~~mermaid
flowchart LR
    L[Load Generator]
    A[checkout-service]
    B[payment-service]
    DB[(PostgreSQL)]
    Q[Kafka]

    L --> A
    A --> B
    A --> DB
    A --> Q
~~~

生成四类流量：

1. 高频正常请求；
2. 低频正常请求；
3. Error 请求；
4. Slow 请求。

### 33.1 实验矩阵

| 实验 | Sampling 策略 | 主要观察 |
|---|---|---|
| A | Datadog Automatic Sampling | 默认 10 TPS 行为 |
| B | Resource Sampling Rules | Endpoint 优先级 |
| C | Error Sampler | 被 Head Drop 的 Error 可见性 |
| D | Rare Sampler | 低频 Resource 可见性 |
| E | Single Span Sampling | Drop Trace 中保留关键 Span |
| F | Adaptive Sampling | 月度目标下 Rate 调整 |
| G | OTel SDK 10% Head Sampling | APM Metrics 偏差 |
| H | OTel Collector Tail Sampling | Error / Slow Trace 完整性 |
| I | span_metrics before Tail Sampling | 全量 Metrics |
| J | span_metrics after Sampling | Sampled Metrics |
| K | Agent OTLP Probabilistic Sampling | Trace-ID 概率采样 |
| L | Retention Filters | Ingested vs Indexed 差异 |

### 33.2 需要记录的指标

每轮实验记录：

~~~text
Requests Generated
Spans Generated
Traces Ingested
Spans Ingested
Error Traces Ingested
Slow Traces Ingested
Rare Resource Traces Ingested
Indexed Spans
APM Request Count
APM Error Count
Latency Metrics
Collector CPU
Collector Memory
Agent CPU
Agent Memory
~~~

并计算：

~~~text
Trace Ingestion Ratio
Error Preservation Ratio
Slow Trace Preservation Ratio
Rare Trace Preservation Ratio
Metric Error %
Ingested Bytes / Request
Indexed Spans / Request
~~~

---

## 34. 重点验证假设

建议将下面几个问题作为实验结论，而不是仅依赖产品文档。

### H1：Datadog Native Ingestion Sampling 不影响 APM Metrics

验证：

~~~text
真实请求数量
vs
APM Hits
vs
Ingested Trace Count
~~~

### H2：OTel SDK Head Sampling 会影响 Datadog APM Metrics

对比：

~~~text
OTel SDK 100%
vs
OTel SDK 10%
~~~

### H3：span_metrics 位于 Tail Sampling 之前可以保持更完整的 APM Metrics

对比：

~~~text
span_metrics -> tail_sampling
~~~

和：

~~~text
tail_sampling -> span_metrics
~~~

### H4：Error Sampler 不能恢复完整 Distributed Trace

构造：

~~~text
Service A -> B -> C(error)
~~~

检查 Datadog 中最终保存的 Trace 是否完整。

### H5：Tail Sampling 能稳定保留 Error / Slow Trace，但 Collector Resource Cost 更高

测量：

- Trace Buffer；
- Memory；
- Decision Wait；
- Throughput；
- Dropped Span。

### H6：Inferred Services / Dependency Metrics 会受到 Sampling Position 的影响

将 Peer Metrics 放在 Sampling 前后比较：

- Request Count；
- Error Rate；
- Peer Entity Coverage。

---

## 35. 当前结论

Datadog APM Sampling 更准确的理解不是：

> “Datadog 使用多少百分比采样？”

而是：

> **Datadog 用多个阶段控制 Trace 的 Ingestion、诊断覆盖率和长期 Retention，同时尽量把聚合 APM Metrics 与 Trace 明细采样解耦。**

可以将整体策略压缩成：

~~~text
应用流量
    ↓
Head Sampling
    ↓
Error / Rare / Single Span 等补偿
    ↓
APM Metrics + Trace Ingestion
    ↓
Live Search
    ↓
Intelligent / Custom Retention
    ↓
长期 Indexed Trace
~~~

OpenTelemetry 场景则增加了一个非常重要的架构选择：

~~~text
在哪里 Sampling？
~~~

如果在 SDK 太早 Sampling：

~~~text
低网络成本
低 Collector 成本
但下游无法恢复完整 Metrics 和 Trace
~~~

如果在 Collector Tail Sampling：

~~~text
能够根据 Error / Latency / Attribute 决策
能够在 Sampling 前计算 Trace Metrics
但需要更高 Collector 资源和 Trace-aware Routing
~~~

因此，对 Datadog + OpenTelemetry 的生产架构来说，Sampling 不应该只讨论“采样率”，而应该同时讨论：

- Sampling Decision 在哪一层发生；
- Decision 是否能够看到完整 Trace；
- Sampling 前是否完成 Metrics Aggregation；
- 是否保持 Distributed Trace 完整；
- 是否影响 Peer / Inferred Service 统计；
- Ingestion 与 Retention 是否被混为一谈。

---

## 参考资料

### Datadog APM

- Trace Pipeline  
  https://docs.datadoghq.com/tracing/trace_pipeline/
- Ingestion Mechanisms  
  https://docs.datadoghq.com/tracing/trace_pipeline/ingestion_mechanisms/
- Ingestion Controls  
  https://docs.datadoghq.com/tracing/trace_pipeline/ingestion_controls/
- Adaptive Sampling  
  https://docs.datadoghq.com/tracing/trace_pipeline/adaptive_sampling/
- Trace Retention  
  https://docs.datadoghq.com/tracing/trace_pipeline/trace_retention/
- Trace Sampling Use Cases  
  https://docs.datadoghq.com/tracing/guide/ingestion_sampling_use_cases/
- Diversity Sampling / Retention Policy  
  https://docs.datadoghq.com/tracing/guide/leveraging_diversity_sampling/
- Trace Explorer  
  https://docs.datadoghq.com/tracing/trace_explorer/

### Datadog + OpenTelemetry

- Ingestion Sampling with OpenTelemetry  
  https://docs.datadoghq.com/opentelemetry/ingestion_sampling/
- OpenTelemetry Trace Metrics  
  https://docs.datadoghq.com/opentelemetry/integrations/trace_metrics/

### OpenTelemetry

- Sampling  
  https://opentelemetry.io/docs/concepts/sampling/
- Collector Tail Sampling Processor  
  https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/tailsamplingprocessor
