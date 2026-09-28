# Datadog APM Trace Retention（Trace 数据保留策略）调研

> 调研日期：2026-09-28  
> 范围：Datadog APM Trace Retention，包括 Intelligent Retention、Diversity Sampling、1% Flat Sampling、Default/Custom Retention Filters、Span-level/Trace-level Retention、Trace Queries、Trace Analytics Monitor、Retention 成本与数据生命周期。

## 1. 摘要

Datadog APM 的 Trace Retention 是一套独立于 Ingestion Sampling 的数据生命周期策略。

最重要的概念区分：

~~~text
Ingestion Sampling
=
决定哪些 Trace / Span 进入 Datadog

Retention
=
决定已经进入 Datadog 的 Span 中，
哪些被索引并成为长期可查询数据
~~~

Retention 不会降低上游 Ingestion Volume。它发生在 Span 已经 Ingest 之后，主要控制：

- 哪些 Span 可以在历史 Trace Explorer 中搜索；
- 哪些完整 Trace 可以用于 Trace Queries；
- 哪些业务 Trace 被长期保留；
- Indexed Spans 使用量；
- 数据集是偏诊断型还是比例代表型。

Datadog 默认提供一个始终启用的 **Intelligent Retention Filter**，由：

1. Diversity Sampling；
2. 1% Flat Sampling；

组成。

除此之外，还有：

- Default Retention Filters；
- Custom Retention Filters。

整体模型：

~~~mermaid
flowchart TD
    A[Ingested Trace / Span]

    A --> L[Live Search<br/>约 15 分钟]

    A --> IR[Intelligent Retention]
    A --> DR[Default Retention Filters]
    A --> CR[Custom Retention Filters]

    IR --> DS[Diversity Sampling]
    IR --> FS[1% Flat Sampling]

    DS --> X[Indexed Data]
    FS --> X
    DR --> X
    CR --> X

    X --> E[Trace Explorer]
    X --> Q[Trace Queries]
    X --> D[Dashboard / Notebook]
~~~

需要特别注意：

> **“被 Ingest”与“被 Indexed”是两个不同状态。**

---

## 2. Retention 在 Trace Pipeline 中的位置

Datadog 的 Trace Pipeline 可以拆成：

~~~mermaid
flowchart LR
    A[Application]
    B[Ingestion Sampling]
    C[Datadog Ingestion]
    D[Live Search]
    E[Retention]
    F[Indexed Data]

    A --> B --> C
    C --> D
    C --> E --> F
~~~

### 2.1 Ingestion

控制：

~~~text
哪些 Trace / Span 被发送到 Datadog
~~~

影响：

- ingested traces；
- ingested spans；
- ingested bytes；
- 最近数据的可见范围；
- Ingestion 使用量。

### 2.2 Retention

控制：

~~~text
已经进入 Datadog 的 Span 中，
哪些被 Index
~~~

影响：

- 历史查询能力；
- Trace Explorer 中超过 Live Search 时间窗口后的数据；
- Trace Queries；
- Indexed Span Usage。

因此：

~~~text
Retention Filter
不会降低
Ingested Bytes / Ingested Spans
~~~

如果目标是降低 Ingestion Volume，应修改 Ingestion Sampling，而不是 Retention。

---

## 3. Live Search 与 Indexed Search

Datadog 会短期保留 Ingested Trace/Span，使它们可以通过 Live Search 查询。

当前文档中的典型时间窗口：

~~~text
15 minutes
~~~

超过这个实时窗口以后，能够继续搜索的数据取决于 Retention。

概念上：

~~~mermaid
flowchart LR
    I[Ingested Span]

    I --> L[Live Search<br/>短期完整可见]
    I --> R[Retention Decision]

    R -->|Index| H[Historical Search]
    R -->|Not Indexed| D[退出长期可搜索数据集]
~~~

所以如果某个 Span：

~~~text
已经 Ingest
但是没有被任何 Retention 机制 Index
~~~

那么它可能在刚产生时可以通过 Live Search 找到，但之后不会继续存在于长期历史搜索数据集中。

---

## 4. Intelligent Retention Filter

Datadog 的 **Intelligent Retention Filter** 对服务始终启用，不要求用户为每个 Service/Resource 自己建立大量 Retention Filter。

它由：

~~~mermaid
flowchart TD
    IR[Intelligent Retention]

    IR --> DS[Diversity Sampling]
    IR --> FS[1% Flat Sampling]
~~~

组成。

这两套数据集解决的问题不同：

| 机制 | 主要目标 | 数据分布 |
|---|---|---|
| Diversity Sampling | 保证诊断覆盖面 | 有意偏向 Error / High Latency / Rare Resource |
| 1% Flat Sampling | 提供比例更接近总体的数据集 | Uniform / proportionally representative |

Datadog 当前文档说明：

> Intelligent Retention 产生的 indexed spans 不计入 indexed span usage，因此不会增加这部分计费使用量。

这意味着 Datadog 默认维护了一套基础的历史诊断数据集。

---

## 5. Diversity Sampling

Diversity Sampling 的目标不是随机抽出“平均请求”。

它的目标是：

> **让不同 Environment、Service、Operation、Resource，以及不同性能和错误形态都有至少一些可诊断样本。**

Datadog 会扫描 Service Entry Spans，并针对每个：

~~~text
environment
+
service
+
operation
+
resource
~~~

组合保留代表数据。

当前文档描述包括：

- 最多每 15 分钟至少保留一个 Span 以及关联 Trace；
- 保留高延迟的 p75、p90、p95 代表样本；
- 保留具有错误多样性的 Error，例如不同 4xx、5xx。

概念上：

~~~mermaid
flowchart TD
    T[某 Resource 的全部请求]

    T --> N[Normal]
    T --> P75[p75]
    T --> P90[p90]
    T --> P95[p95]
    T --> E4[4xx Error]
    T --> E5[5xx Error]
    T --> R[低流量样本]

    P75 --> K[诊断样本]
    P90 --> K
    P95 --> K
    E4 --> K
    E5 --> K
    R --> K
~~~

它确保即使某个 endpoint 流量很低，也更有机会在 Service/Resource 页面找到一个历史示例 Trace。

---

## 6. Diversity Sampling 是有偏数据集

这是 Retention 中最重要的分析约束之一。

Diversity Sampling **不是 uniform sampling**。

它明确偏向：

~~~text
Error
High Latency
Rare / Low-throughput Resource
~~~

因此不能直接把这批数据当作真实请求分布。

假设真实流量：

~~~text
Success = 99%
Error   = 1%
~~~

Diversity Sampling 可能为了保留不同 Error，而让历史样本中 Error 占比远高于 1%。

因此下面这种计算可能是错误的：

~~~text
Diversity Dataset 中：
error traces / all traces

≠

真实系统 Error Rate
~~~

### 6.1 正确用途

更适合：

- 找一个典型慢请求；
- 找不同错误类型的例子；
- 找低流量 Endpoint 示例；
- 进行 Root Cause Analysis；
- 从 Service/Resource 页面进入具体 Trace。

### 6.2 不适合直接做比例推断

不适合直接用于：

- Error Rate 估计；
- Endpoint Traffic Share；
- 用户请求类型比例；
- 业务事件概率。

如果要做比例型分析，应优先：

- 使用 APM Metrics；
- 或使用 1% Flat Sampling 数据集；
- 或使用明确设计的 Custom Retention 数据集。

---

## 7. 1% Flat Sampling

Intelligent Retention 的第二部分是：

~~~text
1% Flat Sampling
~~~

它包含两条相关路径。

### 7.1 基于 trace_id 的 Uniform 1%

Datadog 对 Ingested Spans 做约 1% 的 Uniform Sampling，并基于：

~~~text
trace_id
~~~

做一致决策。

因此同一 Trace 中的所有 Span 会共享同一个 Sampling Decision。

~~~mermaid
flowchart LR
    T[trace_id]

    T --> S1[Span A]
    T --> S2[Span B]
    T --> S3[Span C]

    T --> D{1% Decision}
    D -->|KEEP| K[整条 Trace 的相关 Span 一致保留]
~~~

这批数据的特点是：

> **Uniform，并且在统计意义上更接近 Ingested Traffic 的比例分布。**

Datadog 建议它用于：

- General System Health；
- Trend Analysis；
- System-wide Analysis；
- Trace Queries。

### 7.2 一个重要限制

因为只有约 1%，低流量服务或 endpoint 在短时间范围内可能完全没有样本。

所以：

~~~text
Flat Sampling
更适合总体趋势

Diversity Sampling
更适合保证诊断覆盖
~~~

两者是互补关系。

---

## 8. RUM Session 关联的 1% Sampling

Intelligent Retention 还会保留：

> 与约 1% 已 Ingest RUM Session 关联的 Trace。

这类 Sampling 基于：

~~~text
session_id
~~~

而不是 trace_id。

因此同一个 RUM Session 关联的 Backend Traces 会共享一致的索引决策。

~~~mermaid
flowchart LR
    R[RUM Session]

    R --> A[Frontend Action]
    R --> T1[Backend Trace 1]
    R --> T2[Backend Trace 2]
    R --> T3[Backend Trace 3]

    R --> D{Session Sampling}
    D -->|KEEP| K[关联 Trace 一起保留]
~~~

这支持：

~~~text
Frontend RUM
↔
Backend APM
~~~

联合诊断。

---

## 9. retained_by：为什么这条 Span 被留下

所有 Retained Span 都可以通过：

~~~text
retained_by
~~~

识别其保留原因。

Datadog 当前主要值包括：

| retained_by | 含义 |
|---|---|
| retained_by:diversity_sampling | Intelligent Retention 的 Diversity Sampling |
| retained_by:flat_sampled | Intelligent Retention 的 1% Flat Sampling |
| retained_by:retention_filter | Span-level Custom/Default Retention Filter |
| retained_by:trace_retention_filter | 配置了 Trace Rate 的 Trace-level Retention Filter |

对于：

~~~text
retained_by:flat_sampled
~~~

还可以继续使用：

~~~text
@retention_reason:rum
~~~

识别基于 RUM Session 的保留；

或者：

~~~text
@retention_reason:trace
~~~

识别基于 trace_id 的 Uniform Sampling。

这组字段对实验和成本分析非常重要，因为它把：

~~~text
“为什么我还能查到这条数据？”
~~~

变成可查询属性，而不是黑盒推断。

---

## 10. Default Retention Filters

除了 Intelligent Retention，Datadog 根据启用的产品创建若干默认 Retention Filters。

当前文档列出的主要类型包括：

### 10.1 Error Default

默认 Query：

~~~text
status:error
~~~

用于长期保留 Error Span。

Query 和 Retention Rate 可以配置。

例如：

~~~text
status:error env:production
~~~

只针对 Production Error。

### 10.2 App and API Protection Default

如果使用 App and API Protection，会确保具有应用安全影响的 Trace 数据得到保留。

### 10.3 Synthetics Default

如果使用 Synthetic Monitoring，则保留 Synthetic API / Browser Test 对应的 Trace，以便跨 Synthetics 与 APM 分析。

### 10.4 Dynamic Instrumentation Default

如果使用 Dynamic Instrumentation，则保留动态创建的 Span。

整体上：

~~~mermaid
flowchart TD
    D[Default Retention Filters]

    D --> E[Error]
    D --> S[Security]
    D --> Y[Synthetics]
    D --> DI[Dynamic Instrumentation]
~~~

这些 Default Filter 同样属于 Retention Policy，需要纳入治理，而不能只关注用户自己创建的 Custom Filter。

---

## 11. Custom Retention Filters

Custom Retention Filter 用来表达：

> **哪些业务 Trace / Span 对组织来说值得长期查询。**

可以根据任意 Span Tag / Attribute 创建 Filter。

例如：

### Error

~~~text
status:error
~~~

### Payment

~~~text
service:payment-service
~~~

### 高价值交易

~~~text
@transaction_amount:>10000
~~~

### 高延迟业务请求

~~~text
env:prod resource_name:"POST /checkout" @duration:>2s
~~~

Custom Filter 通常应该反映：

- 业务重要性；
- 调试价值；
- Compliance / Audit 需要；
- 高风险操作；
- 特定用户路径；
- 特定生产环境异常。

---

## 12. Span-level Retention

Standard Retention Filter 属于 Span-level Retention。

例如：

~~~text
service:payment-service
~~~

如果匹配到：

~~~text
payment-service span
~~~

则只有匹配 Query 的 Span 会被索引。

~~~mermaid
flowchart LR
    C[checkout Span]
    P[payment Span<br/>匹配]
    DB[postgres Span]
    K[kafka Span]

    C --> P
    P --> DB
    P --> K

    P --> I[Indexed]
~~~

此时：

~~~text
payment Span
~~~

可以在长期 Span Explorer 中搜索。

但：

~~~text
checkout
postgres
kafka
~~~

不因为属于同一 Trace 就自动成为 Indexed Span。

---

## 13. Trace-level Retention

Trace-level Retention Filter 增加了：

~~~text
Trace Retention Rate
~~~

当某个 Span 匹配 Retention Query，并且 Trace Rate 命中时：

> 整条 Trace 的所有 Span 都被索引。

~~~mermaid
flowchart LR
    C[checkout]
    P[payment<br/>Query 命中]
    DB[postgres]
    K[kafka]

    C --> P
    P --> DB
    P --> K

    P --> R{Trace Rate 命中}
    R --> I[完整 Trace Indexed]
~~~

这样：

~~~text
checkout
payment
postgres
kafka
~~~

都可以作为历史 Indexed Data 被查询。

Trace-level Retention 是：

~~~text
Trace Queries
~~~

能够查询完整 Trace 的重要来源之一。

---

## 14. Span Rate 与 Trace Rate 的两阶段机制

Custom Retention Filter 可以配置：

~~~text
Span Rate
~~~

以及可选的：

~~~text
Trace Rate
~~~

执行顺序是：

~~~text
Span Rate
先执行

Trace Rate
只作用于 Span Rate 已选择的 Trace
~~~

例如：

~~~text
Query: service:my-service

Span Rate = 50%
Trace Rate = 10%
~~~

流程：

~~~mermaid
flowchart TD
    A[匹配 Query 的 Trace]
    A --> S{Span Rate 50%}

    S -->|未选中| D[不由该 Filter Index]
    S -->|选中| I[匹配 Span 被 Index]

    I --> T{Trace Rate 10%}

    T -->|未选中| P[只保留匹配 Span]
    T -->|选中| F[完整 Trace 所有 Span 都 Index]
~~~

这意味着 Trace Rate 会放大 Indexed Span Volume。

---

## 15. Trace-level Retention 的成本放大效应

Datadog 文档给出的示例非常有代表性。

假设：

~~~text
平均每条 Trace = 100 Spans
其中匹配 service:my-service = 5 Spans
~~~

只使用 Span-level Retention：

~~~text
可能 Index 5 个 Span
~~~

如果 Trace-level Retention 命中：

~~~text
另外 95 个 Span
也需要被 Index
~~~

因此：

> **Trace Rate 对 Indexed Spans Usage 的影响可能远高于直觉。**

在设计 Retention 时不能只看：

~~~text
Trace Rate = 10%
~~~

还需要看：

~~~text
Average Spans per Trace
Matched Spans per Trace
Full Trace Expansion Factor
~~~

可以定义：

~~~text
Trace Expansion Factor
≈
Average Spans per Trace
/
Average Directly Matched Spans per Trace
~~~

这不是 Datadog 官方计费公式，而是用于容量分析的工程估算指标。

---

## 16. Retention Filter 的顺序是策略本身

Retention Filters 按列表顺序串行执行。

Datadog 明确说明：

> 如果一个 Span 已经匹配前面的 Filter，并被 Keep 或 Drop，后面的 Custom Retention Filter 不会再处理这个 Span。

所以 Filter Order 会改变实际索引结果。

例如：

### 配置 A

~~~text
1. service:checkout -> 1%
2. status:error     -> 100%
~~~

### 配置 B

~~~text
1. status:error     -> 100%
2. service:checkout -> 1%
~~~

两者可能产生完全不同的长期数据集。

因此 Retention Policy 不只是：

~~~text
Query + Rate
~~~

而是：

~~~text
Ordered Rules
~~~

这和 Firewall Rules、Routing Rules、Sampling Rules 类似，需要显式治理优先级。

---

## 17. 可视化完整 Trace 不等于所有 Span 都被 Index

这是使用 Trace Explorer 时很容易误解的一点。

假设只有：

~~~text
payment Span
~~~

被 Span-level Retention Index。

打开这个 Span 时，Datadog 可以显示关联 Trace Context，让用户看到：

~~~text
checkout
  -> payment
       -> postgres
       -> kafka
~~~

但是：

~~~text
能在 Trace UI 中看到上下文
~~~

不等于：

~~~text
所有 Span 都被 Indexed
~~~

没有被 Index 的 Span 不一定能通过历史 Span Query 独立搜索。

因此要区分：

- Trace visualization context；
- Indexed searchable span；
- 完整 Trace-level indexing。

---

## 18. Trace Queries 使用的数据集

Datadog 当前明确说明：

> Trace Queries 基于 Intelligent Retention Filter 索引的数据，以及 Trace-level Retention Filters 产生的完整 Trace 数据。

这意味着 Trace Queries 并不是对：

~~~text
所有曾经 Ingest 的 Trace
~~~

执行查询。

它依赖 Retention Dataset。

概念上：

~~~mermaid
flowchart TD
    I[Ingested Trace]

    I --> IR[Intelligent Retention<br/>完整 Trace 数据集]
    I --> TR[Trace-level Retention]

    IR --> Q[Trace Queries]
    TR --> Q
~~~

因此设计复杂 Trace Query 时，需要先确认：

> Query 所需的数据是否实际存在于 Retention Dataset 中。

---

## 19. Intelligent Retention 与 Trace Analytics Monitor 的区别

这是 Datadog Retention 中一个非常重要、容易忽略的限制：

> **Intelligent Retention Filter 索引的 Span 不参与 APM Trace Analytics Monitor 的评估。**

也就是说：

~~~text
Diversity Sampling
+
1% Flat Sampling
~~~

可以用于 Trace Explorer、分析和 Trace Queries，但不能假设它们会驱动 Trace Analytics Monitor。

如果需要针对某种 Span 建立 Trace Analytics Monitor，例如：

~~~text
status:error
@payment_method:card
~~~

应确保这些数据通过：

~~~text
Custom / Default Retention Filter
~~~

进入 Monitor 可评估的数据集。

另外，Trace-level Retention 间接索引的 Span——即自身不匹配 Query、只是因为属于完整 Trace 而被 Index 的 Span——也不会被 Trace Analytics Monitor 评估。

---

## 20. Intelligent Retention 数据为什么“免费”但不能替代 Custom Retention

Intelligent Retention 的价值很高：

- 自动覆盖各种 Service/Resource；
- 自动保留 Slow/Error 样本；
- 提供 1% Uniform 数据；
- 不计入 Indexed Span Usage。

但它不能替代 Custom Retention。

因为业务可能需要：

~~~text
100% 保留关键 Payment Error
100% 保留安全相关操作
100% 保留高价值订单
50% 保留某种新功能请求
~~~

Intelligent Retention 不理解业务重要性。

可以把两者的职责理解成：

~~~mermaid
flowchart LR
    IR[Intelligent Retention]
    CR[Custom Retention]

    IR --> A[系统基础诊断覆盖]
    CR --> B[业务明确保留策略]
~~~

---

## 21. Retention Duration：15 天、30 天与套餐差异

Datadog 不同文档中会看到不同 Retention 数字，需要区分数据类型和套餐。

当前 Data Retention Periods 文档对 APM 给出的信息是：

| 数据类型 | 默认保留期 |
|---|---|
| Errors | 15 天 |
| Indexed Spans | 15 或 30 天，取决于 Customer Plan |
| Service / Resource Statistics | 30 天 |
| Viewed Traces | Account 生命周期内保留 |

同时，Trace Retention 页面：

- Custom Retention 工作流通常以“保留 15 天”描述；
- Diversity Sampling 当前文档描述其数据保留 30 天。

因此仓库中不应该把：

~~~text
Indexed Trace 永远 = 15 天
~~~

作为统一结论。

更准确的表达是：

> **具体 Retention Duration 取决于数据类型、Retention 机制和客户套餐；设计生产策略时应以当前账户 Plan 和 Data Retention Periods 页面为准。**

---

## 22. APM Metrics 与 Trace Retention 的时间尺度不同

APM Trace Metrics 与 Raw Trace/Span 的生命周期并不相同。

Datadog APM Metrics 记录：

- Request Count；
- Error Count；
- Latency。

它们基于聚合数据，而不是依赖长期保留每一个 Raw Span。

因此可以形成：

~~~mermaid
flowchart TD
    A[Application Traffic]

    A --> M[APM Metrics]
    A --> I[Trace Ingestion]

    I --> R[Retention]
    R --> X[Indexed Raw Trace]

    M --> MM[长期趋势 / Monitor / SLO]
    X --> D[具体 Trace Debug]
~~~

这说明长期可观测性不是：

~~~text
把所有 Raw Trace 永久保存
~~~

而是：

~~~text
长期保留聚合 Metrics
+
有策略地保留诊断级 Raw Trace
~~~

---

## 23. Datadog APM 的数据分层模型

从数据生命周期角度，可以把 APM 数据划分成四类。

| 数据层 | 主要用途 | 选择方式 | 主要 Usage Driver |
|---|---|---|---|
| APM Trace Metrics | Monitor / SLO / Trend | 聚合 | APM Metrics |
| Ingested Trace | 近期 Debug / Live Search | Ingestion Sampling | Ingested Bytes/Spans |
| Intelligent Retention | 基础历史诊断 / Trace Queries | Datadog 自动选择 | 不计入 Indexed Span Usage |
| Custom Indexed Data | 业务级历史搜索 | 用户定义 Retention | Indexed Spans |

可以理解为：

~~~mermaid
flowchart TD
    R[Raw Application Traffic]

    R --> M[Metrics Tier<br/>长期聚合]
    R --> I[Ingested Trace Tier<br/>短期细节]

    I --> IR[Intelligent Diagnostic Tier]
    I --> CR[Business Indexed Tier]

    IR --> H[Historical Trace Analysis]
    CR --> H
~~~

这比简单的：

~~~text
Metrics vs Traces
~~~

更接近 Datadog APM 实际的数据管理模型。

---

## 24. Ingested Usage 与 Indexed Usage

两个很重要的 Usage Metrics：

~~~text
datadog.estimated_usage.apm.ingested_bytes
datadog.estimated_usage.apm.indexed_spans
~~~

分别代表不同控制面。

### Ingested Bytes

主要受：

- Trace Ingestion Sampling；
- Trace 数量；
- Span 数量；
- Span Payload Size；

影响。

### Indexed Spans

主要受：

- Custom/Default Retention Filters；
- Span Rate；
- Trace Rate；
- 完整 Trace 展开后的 Span 数量；

影响。

因此成本优化也应该分开：

~~~text
Ingestion Cost Optimization
!=
Indexing Cost Optimization
~~~

例如：

> 把 Custom Retention Rate 从 100% 调到 10%，可能显著降低 Indexed Span Usage，但不会减少 Agent 已经发送到 Datadog 的 Ingested Bytes。

---

## 25. Retention Policy 的设计问题

设计 Retention 时，不应该只问：

> “保存多少百分比？”

更应该问：

### 25.1 哪些数据只需要 Metrics？

例如：

- 高频 Health Check；
- 正常静态请求；
- 重复度极高的普通流量。

### 25.2 哪些数据需要至少有诊断样本？

可以依赖：

- Diversity Sampling；
- Flat Sampling。

### 25.3 哪些业务事件必须长期可搜索？

例如：

- Payment Failure；
- 高价值订单；
- 安全敏感操作；
- 关键 API Failure；
- 生产事故相关路径。

适合 Custom Retention。

### 25.4 哪些场景必须完整保留整条 Trace？

例如需要：

~~~text
A -> B -> C -> Database -> Queue
~~~

做跨服务 Trace Query 或完整根因分析。

适合 Trace-level Retention。

---

## 26. 一个建议的生产 Retention 分层

下面不是 Datadog 官方唯一推荐配置，而是用于设计的策略框架。

~~~text
Layer 1: APM Metrics
100% 聚合统计

Layer 2: Intelligent Retention
系统自动诊断样本

Layer 3: Custom Span Retention
关键 Service / Error / Business Span

Layer 4: Custom Trace Retention
极关键业务路径的完整 Trace
~~~

概念图：

~~~mermaid
flowchart TD
    A[全部应用请求]

    A --> M[APM Metrics<br/>全量聚合]
    A --> ING[Ingested Trace]

    ING --> INT[Intelligent Retention]
    ING --> SP[Custom Span Retention]
    ING --> TR[Custom Trace Retention]

    INT --> D1[一般诊断]
    SP --> D2[关键 Span 历史查询]
    TR --> D3[关键完整 Trace]
~~~

---

## 27. Retention Policy 示例

假设支付系统有：

~~~text
GET /health
GET /product
POST /checkout
POST /payment
POST /refund
~~~

可以设计：

### 系统层

依赖 Intelligent Retention：

~~~text
Diversity Sampling
+
1% Flat Sampling
~~~

### Error

Default / Custom Filter：

~~~text
status:error env:prod
Span Rate = 100%
~~~

### Payment

~~~text
service:payment-service env:prod
Span Rate = 20%
Trace Rate = 5%
~~~

### 高价值交易

~~~text
@transaction_amount:>10000
Span Rate = 100%
Trace Rate = 100%
~~~

### Health Check

不额外建立高 Retention Filter。

这样可以分别满足：

- 系统基础诊断；
- Error 历史查询；
- Payment 分析；
- 高价值交易完整追踪；
- 控制 Indexed Span Volume。

具体比例必须根据真实流量和成本测量调整。

---

## 28. Retention Anti-patterns

### 28.1 所有 Span 100% Index

~~~text
所有 Service
Span Rate = 100%
Trace Rate = 100%
~~~

对于大规模系统会产生大量 Indexed Spans，通常失去了 Sampling/Retention 分层的意义。

### 28.2 用 Diversity Dataset 计算真实 Error Rate

错误，因为它本身对 Error/Latency 有偏。

### 28.3 只看 Filter Query，不看 Filter Order

可能导致高优先级 Filter 提前处理 Span，使下游规则永远无法命中。

### 28.4 Trace Rate 设置很小就认为成本一定很小

错误。完整 Trace 可能包含几十或几百 Span。

### 28.5 认为 UI 能看到完整 Trace 就代表所有 Span 都 Indexed

错误。Visual Context 与 Searchable Indexed Span 是两个概念。

### 28.6 认为 Retention 可以降低 Ingestion Cost

错误。Retention 发生在 Ingestion 之后。

---

## 29. 与 Sampling Strategy 文档的关系

Sampling 和 Retention 应分别回答两个问题。

### Sampling

~~~text
哪些 Trace 值得进入 Datadog？
~~~

关注：

- Head Sampling；
- Adaptive Sampling；
- Error/Rare Sampling；
- OTel Tail Sampling；
- Ingestion Cost；
- Trace 完整性。

### Retention

~~~text
进入 Datadog 后，
哪些数据值得长期可搜索？
~~~

关注：

- Intelligent Retention；
- Diversity / Flat Dataset；
- Span-level / Trace-level Index；
- Trace Queries；
- Indexed Cost；
- 业务数据生命周期。

因此完整链路是：

~~~mermaid
flowchart LR
    A[Application]
    S[Sampling]
    I[Ingestion]
    R[Retention]
    H[Historical Indexed Data]

    A --> S --> I --> R --> H
~~~

---

## 30. 与 Inferred Services 的关系

Retention 一般不会改变已经基于 Trace 计算出来的：

- APM Metrics；
- Peer Service Metrics；
- Inferred Service Metrics。

但是会影响：

> 后续是否还有具体 Trace 样本可以用于验证某条 dependency edge。

例如：

~~~mermaid
flowchart LR
    C[checkout-service]
    P[postgres inferred dependency]

    C --> P
~~~

Service/Dependency Metric 可以长期存在，但如果相关 Trace 没有被 Retention：

~~~text
你可能看到“这个依赖变慢了”
但历史上没有足够的具体 Trace 可进一步 Debug
~~~

所以 Sampling、Metrics 与 Retention 是互补的：

~~~text
Metrics
告诉你哪里异常

Retention
决定你是否还有历史明细可以深入调查
~~~

---

## 31. Observability-Lab 建议实验

建议后续创建：

~~~text
labs/datadog-trace-retention/
~~~

统一 workload：

~~~mermaid
flowchart LR
    L[Load Generator]
    C[checkout-service]
    P[payment-service]
    DB[(PostgreSQL)]

    L --> C
    C --> P
    P --> DB
~~~

生成：

- 高频正常请求；
- 低频 endpoint；
- Error；
- p75/p90/p95 慢请求；
- 高价值 transaction。

### 实验 A：Diversity Sampling

验证：

- 低流量 Resource 是否有样本；
- Error 类型覆盖；
- p75/p90/p95 Trace；
- retained_by。

### 实验 B：Flat Sampling

验证：

- retained_by:flat_sampled；
- @retention_reason:trace；
- 样本比例；
- 低流量短窗口缺失。

### 实验 C：RUM-linked Retention

如果接入 RUM：

- @retention_reason:rum；
- 同一 session 关联 Trace 的一致性。

### 实验 D：Span-level Filter

~~~text
service:payment-service
Span Rate = 50%
~~~

观察哪些 Span 可独立搜索。

### 实验 E：Trace-level Filter

~~~text
service:payment-service
Span Rate = 50%
Trace Rate = 10%
~~~

比较 Indexed Span 数量增长。

### 实验 F：Filter Ordering

交换：

~~~text
status:error -> 100%
service:checkout -> 1%
~~~

顺序，比较结果。

### 实验 G：Monitor Dataset

验证：

- Intelligent Retention Data；
- Custom Retention Data；

在 Trace Analytics Monitor 中的差异。

---

## 32. 实验需要记录的数据

每次实验建议记录：

~~~text
Requests Generated
Traces Ingested
Spans Ingested
Spans Indexed

retained_by Distribution
retention_reason Distribution

Trace Count in Trace Queries
Span Count in Span Explorer

Indexed Spans Usage
Ingested Bytes

Average Spans per Trace
Matched Spans per Trace
~~~

建议计算：

~~~text
Index Ratio
=
Indexed Spans / Ingested Spans
~~~

以及实验性的：

~~~text
Trace Expansion Factor
=
完整 Trace 平均 Span 数
/
Query 直接命中的平均 Span 数
~~~

用于理解 Trace-level Retention 的 Index 放大程度。

---

## 33. 当前结论

Datadog APM 的 Retention 不应该被理解成：

> “Sampling 后再随机留一点 Trace。”

更准确的模型是：

> **Retention 是对已经进入 Datadog 的 Trace 数据进行分层索引和长期数据集构建的机制。**

Datadog 默认同时维护两类互补数据：

~~~text
Diversity Sampling
=
诊断覆盖优先

1% Flat Sampling
=
比例代表性优先
~~~

再通过：

~~~text
Default Retention
+
Custom Span-level Retention
+
Custom Trace-level Retention
~~~

表达业务级长期数据策略。

从架构角度，Datadog APM 的完整数据生命周期更接近：

~~~mermaid
flowchart TD
    A[100% Application Traffic]

    A --> M[APM Metrics<br/>聚合趋势]
    A --> S[Ingestion Sampling]

    S --> I[Ingested Trace<br/>近期细节]

    I --> IR[Intelligent Retention]
    I --> CR[Custom Retention]

    IR --> H[历史诊断数据]
    CR --> B[业务关键历史数据]

    H --> Q[Trace Explorer / Trace Queries]
    B --> Q
~~~

因此设计生产环境 Datadog APM 策略时，至少要分别回答：

1. **哪些 Trace 应该进入 Datadog？**
2. **哪些 Trace/Span 应该长期被 Index？**
3. **哪些数据需要完整 Trace？**
4. **哪些数据只需要一个代表性 Span？**
5. **哪些统计应该依赖 Metrics，而不是 Retained Trace？**
6. **哪些业务数据值得承担额外 Indexed Span 成本？**

只有把 Ingestion、Metrics 和 Retention 分开设计，Trace Pipeline 才是完整的。

---

## 参考资料

### Datadog

- Trace Retention  
  https://docs.datadoghq.com/tracing/trace_pipeline/trace_retention/
- The Trace Pipeline  
  https://docs.datadoghq.com/tracing/trace_pipeline/
- Trace Explorer  
  https://docs.datadoghq.com/tracing/trace_explorer/
- Trace Queries Source Data  
  https://docs.datadoghq.com/tracing/guide/trace_queries_dataset/
- APM Metrics  
  https://docs.datadoghq.com/tracing/metrics/
- Data Retention Periods  
  https://docs.datadoghq.com/data_security/data_retention_periods/
- APM Retention Filters API  
  https://docs.datadoghq.com/api/latest/apm-retention-filters/
