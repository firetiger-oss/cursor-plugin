# Go instrumentation

Assumes the `OTEL_EXPORTER_OTLP_*` env vars from the SKILL.md are set (Basic auth header from
`get_ingest_credentials`).

**Note:** the gRPC exporter endpoint is `host:port` (no scheme) — `ingest.cloud.firetiger.com:443`.

```bash
go get go.opentelemetry.io/otel \
  go.opentelemetry.io/otel/sdk \
  go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc \
  go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp
```

```go
package main

import (
    "context"

    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc"
    "go.opentelemetry.io/otel/sdk/resource"
    "go.opentelemetry.io/otel/sdk/trace"
    semconv "go.opentelemetry.io/otel/semconv/v1.21.0"
)

func initTracer(ctx context.Context) (*trace.TracerProvider, error) {
    // Endpoint and auth headers come from OTEL_EXPORTER_OTLP_* env vars.
    exporter, err := otlptracegrpc.New(ctx)
    if err != nil {
        return nil, err
    }

    res, _ := resource.New(ctx, resource.WithAttributes(
        semconv.ServiceName("your-service-name"),
    ))

    tp := trace.NewTracerProvider(
        trace.WithBatcher(exporter),
        trace.WithResource(res),
    )
    otel.SetTracerProvider(tp)
    return tp, nil
}
```

Call `initTracer` at the start of `main`, and defer `tp.Shutdown(ctx)` so buffered spans flush on exit:

```go
func main() {
    ctx := context.Background()
    tp, err := initTracer(ctx)
    if err != nil {
        log.Fatal(err)
    }
    defer tp.Shutdown(ctx)
    // ... start your server, wrapping handlers with otelhttp.NewHandler(...)
}
```

Environment:

```bash
export OTEL_EXPORTER_OTLP_ENDPOINT="ingest.cloud.firetiger.com:443"
export OTEL_EXPORTER_OTLP_HEADERS="Authorization=Basic <base64(username:password)>"
```

## Next steps

- **Verify** — generate traffic, then query `"opentelemetry/traces/your-service-name"` with `firetiger-query` (`SHOW TABLES;` first).
- **Custom spans & log correlation** — instrument business-critical operations and join logs to traces in [custom-spans.md](custom-spans.md).
- **Full onboarding** — connect integrations and create a monitoring agent with `firetiger-setup`.
