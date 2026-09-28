# Frontier Observability Topics

## 1. eBPF / zero-code instrumentation
Research:
- protocol coverage
- TLS visibility constraints
- context propagation
- kernel/runtime overhead
- Kubernetes service discovery
- relationship to language auto-instrumentation

## 2. OpenTelemetry Profiles
Research:
- signal maturity
- pprof/JFR/linux_perf compatibility
- trace/profile correlation
- storage and aggregation implications

## 3. GenAI / Agent observability
Research:
- agent, workflow, tool, retrieval, embedding, LLM spans
- token and cost attribution
- evaluation signals
- prompt/output privacy
- model/provider portability
- OpenTelemetry GenAI semantic conventions

## 4. Telemetry economics
Research:
- cardinality explosions
- dynamic/adaptive sampling
- pre-aggregation
- routing by tenant/service/signal
- observability cost per request/service/team

## 5. Telemetry governance
Research:
- PII detection/redaction
- attribute allow/deny lists
- schema migration
- policy-as-code in Collector pipelines
