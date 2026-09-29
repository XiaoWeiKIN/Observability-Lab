# Datadog 调研

## 调研文档

- [推断服务（Inferred Services）](./inferred-services.md) — 调研 Datadog 如何根据出站 Span 推导远端依赖，包括 Peer Identity、实体解析、APM Metrics、OpenTelemetry 映射、Sampling 与 Cardinality。
- [APM 采样策略](./apm-sampling.md) — 调研 Datadog APM 的 Ingestion Sampling、Automatic/Adaptive Sampling、Error/Rare/Single Span Sampling，以及 OpenTelemetry Head/Tail Sampling 对 APM Metrics 和 Trace 完整性的影响。
- [APM Trace 数据保留策略](./apm-trace-retention.md) — 调研 Intelligent Retention、Diversity/1% Flat Sampling、Span-level/Trace-level Retention、Trace Queries、Monitor 数据集、Retention 时长与 Indexed Span 成本。
- [APM Trace Pipeline](./apm-trace-pipeline.md) — 从数据生命周期角度分析 Instrumentation、Sampling、Ingestion、Trace Metrics、Processing、Retention、查询模型、Usage/Cost，以及 OpenTelemetry → Datadog 的语义映射。
- [APM Trace Metrics 与 Service Entry Span](./apm-trace-metrics-service-entry-span.md) — 深入分析 Trace Metrics、Service Entry Span、Measured Span、Primary Operation、Metric Namespace、Cardinality，以及 OpenTelemetry SpanKind → Datadog APM Stats 的映射。

## 调研留存约定

- 所有形成明确结论、架构分析或实验设计的调研，都应保存为仓库中的 Markdown 文档，而不是只保留在聊天记录中。
- 文档正文使用简体中文；协议名、属性名、配置项和必要的英文技术术语保持原文。
- 新调研优先补充到已有主题文档；如果形成独立知识域，则新建文档并在本索引中登记。
- 事实、产品行为和推断应明确区分；可验证的结论应附官方文档或实验依据。

## 调研方向

- APM
- Infrastructure Monitoring
- Logs
- Metrics
- RUM
- Continuous Profiler
- Universal Service Monitoring
- OpenTelemetry / DDOT
- Agent Observability
- Fleet Management
- 成本与用量治理

## OpenTelemetry 互操作矩阵

不同语言、Signal 和部署方式应该分别验证，不能假设它们具有完全一致的行为。

| 场景 | Instrumentation | 数据处理 | Backend |
|---|---|---|---|
| OTel Native | OTel SDK / Auto Instrumentation | Upstream OTel Collector | Datadog |
| DDOT | OTel 或 Datadog Instrumentation | Datadog Distribution of OTel Collector | Datadog |
| Datadog Native | Datadog SDK | Datadog Agent / DDOT | Datadog |
| Hybrid | OTel API + Datadog SDK 或混合 Library | Agent / DDOT / Collector | Datadog |

## 主要调研问题

- 每种接入方式可以使用哪些 Datadog 能力？
- 哪种接入方式能够保留 vendor-neutral instrumentation？
- Sampling 与数据增强分别发生在哪一层？
- Ingestion 过程中哪些 Attributes / Tags 会被转换？
- Upstream Collector 与 DDOT 在运行和数据模型上有哪些差异？
- Logs、Traces、Metrics 之间的关联需要满足哪些条件？
- Sampling 发生在 SDK、Agent 还是 Collector，会如何影响 Trace 完整性和 APM Metrics？
- Ingestion Sampling 与 Retention Sampling 应如何分别控制成本和诊断覆盖率？
