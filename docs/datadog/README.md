# Datadog Research Notes

## Areas

- APM
- Infrastructure Monitoring
- Logs
- Metrics
- RUM
- Continuous Profiler
- Universal Service Monitoring
- OpenTelemetry / DDOT
- Agent Observability
- Fleet management
- Cost and usage controls

## OpenTelemetry interoperability matrix

Track this per language and signal rather than assuming one global answer.

| Scenario | Instrumentation | Processing | Backend |
|---|---|---|---|
| OTel-native | OTel SDK / auto | Upstream OTel Collector | Datadog |
| DDOT | OTel or Datadog instrumentation | Datadog Distribution of OTel Collector | Datadog |
| Datadog-native | Datadog SDK | Datadog Agent / DDOT | Datadog |
| Hybrid | OTel API + Datadog SDK or mixed libraries | Agent / DDOT / Collector | Datadog |

## Evaluation questions

- Which Datadog features are available for each setup?
- Which setup preserves vendor-neutral instrumentation?
- Where are sampling and enrichment performed?
- Which attributes/tags are transformed at ingestion?
- What are the operational differences between upstream Collector and DDOT?
- What is required for logs-traces-metrics correlation?
