# Datadog 调研

## 调研文档

- [推断服务（Inferred Services）](./inferred-services.md) — 调研 Datadog 如何根据出站 Span 推导远端依赖，包括 Peer Identity、实体解析、APM Metrics、OpenTelemetry 映射、Sampling 与 Cardinality。
- [APM 采样策略](./apm-sampling.md) — 调研 Datadog APM 的 Ingestion Sampling、Automatic/Adaptive Sampling、Error/Rare/Single Span Sampling，以及 OpenTelemetry Head/Tail Sampling 对 APM Metrics 和 Trace 完整性的影响。
- [APM Trace 数据保留策略](./apm-trace-retention.md) — 调研 Intelligent Retention、Diversity/1% Flat Sampling、Span-level/Trace-level Retention、Trace Queries、Monitor 数据集、Retention 时长与 Indexed Span 成本。

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
