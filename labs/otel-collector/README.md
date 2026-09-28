# Lab: Minimal OpenTelemetry Collector

Goal: receive OTLP over gRPC/HTTP and inspect telemetry locally.

## Run

Use the Collector Contrib image with `otel-collector-config.yaml`.

```bash
docker run --rm \
  -p 4317:4317 \
  -p 4318:4318 \
  -v "$PWD/otel-collector-config.yaml:/etc/otelcol-contrib/config.yaml" \
  otel/opentelemetry-collector-contrib:latest
```

Then point an instrumented application at:

```text
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318
```

For reproducible research, replace `latest` with a pinned version before committing benchmark results.
