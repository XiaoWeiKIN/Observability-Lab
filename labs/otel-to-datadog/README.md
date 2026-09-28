# Lab: OpenTelemetry → Datadog

Goal: compare multiple supported integration paths instead of assuming they are equivalent.

## Variants

1. OTel SDK → upstream OpenTelemetry Collector → Datadog via OTLP.
2. OTel SDK → DDOT Collector → Datadog.
3. Datadog SDK using OpenTelemetry APIs where supported.
4. Datadog-native instrumentation → Agent/DDOT.

## Record for each run

- language/runtime version
- SDK version
- Collector/DDOT/Agent version
- configuration
- traces/metrics/logs received
- service/resource metadata
- propagation behavior
- feature compatibility
- CPU/memory overhead
- ingest volume
- sampling behavior

## Secrets

Never commit `DD_API_KEY`. Use environment variables, secret stores, or local `.env` files excluded by `.gitignore`.

## Reference

Check the current Datadog OpenTelemetry documentation before implementing the exporter path because recommended configurations can change.
