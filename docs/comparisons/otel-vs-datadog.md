# OpenTelemetry vs Datadog: Research Framework

This is not a winner/loser comparison. OpenTelemetry is primarily an open observability framework/specification/ecosystem, while Datadog is a commercial observability platform that can consume OpenTelemetry data and also provides its own SDKs, agents, and products.

## Dimensions to compare

| Dimension | Questions |
|---|---|
| Instrumentation | Manual, auto, zero-code, eBPF; language coverage |
| Signal coverage | Traces, metrics, logs, profiles |
| Transport | OTLP support, proprietary protocols |
| Processing | Collector, Agent, DDOT; transformations, sampling |
| Correlation | Trace-log-metric-profile linkage |
| Portability | Backend switching cost |
| Feature compatibility | Which platform features are available with pure OTel? |
| Operations | CPU/memory overhead, rollout, config management |
| Governance | PII scrubbing, routing, tenancy, schema control |
| Cost | Ingestion, retention, custom metrics/cardinality, sampling |
| Developer UX | Debuggability, local workflows, onboarding |
| Kubernetes | Operator, injection, daemon/gateway patterns |
| AI workloads | GenAI spans, evaluations, agent/tool workflows |

## Experiment protocol

1. Keep workload constant.
2. Record versions.
3. Keep traffic profile reproducible.
4. Capture CPU/memory overhead.
5. Capture generated span/metric/log volume.
6. Compare data loss and correlation behavior during failure.
7. Record feature gaps without generalizing beyond the tested setup.
