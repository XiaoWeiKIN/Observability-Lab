# Learning & Research Roadmap

## Phase 1 — Foundations

### Outcomes
- Explain traces, metrics, logs, resources, baggage, context, and exemplars.
- Explain propagation across HTTP/gRPC/message queues.
- Read OTLP payloads and semantic-convention attributes.

### Labs
- Instrument one service manually.
- Add zero-code instrumentation to the same service.
- Export OTLP to a local Collector and debug exporter.

---

## Phase 2 — OpenTelemetry Collector

### Topics
- receivers
- processors
- exporters
- connectors
- extensions
- resource detection
- memory limiter and batching
- filtering and transformation
- tail sampling

### Questions
- Where should sampling happen?
- Where should PII scrubbing happen?
- What changes when the Collector runs as sidecar, DaemonSet, or gateway?
- How do backpressure and retries affect application behavior?

---

## Phase 3 — Datadog interoperability

### Compare

| Dimension | OTel-first | Datadog-native | Hybrid |
|---|---|---|---|
| Instrumentation | OTel SDK/auto | Datadog SDK | OTel API + Datadog components |
| Transport | OTLP | Datadog protocol / OTLP where supported | Mixed |
| Processing | OTel Collector | Agent / DDOT | Collector + Agent/DDOT |
| Portability | Higher | Lower | Medium |
| Datadog feature depth | Validate per feature | Usually deepest | Validate per feature |

Do not treat the table as a verdict. Verify feature compatibility for the specific language, signal, and deployment.

---

## Phase 4 — Production engineering

- telemetry SLOs
- collector self-observability
- buffering and retry behavior
- multi-tenant routing
- redaction / PII policies
- cardinality budgets
- sampling policies
- cost attribution
- schema migration
- incident workflows

---

## Phase 5 — Frontier topics

- OpenTelemetry eBPF / zero-code instrumentation
- OpenTelemetry Profiles
- continuous profiling + trace correlation
- GenAI / agent semantic conventions
- LLM evaluations and quality signals
- observability for inference infrastructure / GPUs
- adaptive telemetry and dynamic sampling
- observability pipelines as policy enforcement points
