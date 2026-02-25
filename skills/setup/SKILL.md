---
name: firetiger-setup
description: "Add OpenTelemetry instrumentation to applications for Firetiger observability"
user_invocable: true
user_invocable_description: "Instrument your application with OpenTelemetry for Firetiger"
---

# OpenTelemetry Instrumentation for Firetiger

You are an expert at instrumenting applications with OpenTelemetry to send telemetry data to Firetiger.

## Framework Detection

First, detect the application's technology stack by examining:
- `package.json` → Node.js/TypeScript
- `requirements.txt`, `pyproject.toml`, `setup.py` → Python
- `go.mod` → Go
- `Cargo.toml` → Rust

## Node.js / TypeScript

### Install Dependencies

```bash
npm install @opentelemetry/api \
  @opentelemetry/sdk-node \
  @opentelemetry/auto-instrumentations-node \
  @opentelemetry/exporter-trace-otlp-proto \
  @opentelemetry/exporter-logs-otlp-proto \
  @opentelemetry/exporter-metrics-otlp-proto
```

### Create Instrumentation File

Create `instrumentation.ts` (or `instrumentation.js`):

```typescript
import { NodeSDK } from '@opentelemetry/sdk-node';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-proto';
import { OTLPLogExporter } from '@opentelemetry/exporter-logs-otlp-proto';
import { OTLPMetricExporter } from '@opentelemetry/exporter-metrics-otlp-proto';
import { PeriodicExportingMetricReader } from '@opentelemetry/sdk-metrics';

const sdk = new NodeSDK({
  serviceName: process.env.OTEL_SERVICE_NAME || 'my-service',
  traceExporter: new OTLPTraceExporter(),
  metricReader: new PeriodicExportingMetricReader({
    exporter: new OTLPMetricExporter(),
  }),
  logRecordProcessor: new SimpleLogRecordProcessor(new OTLPLogExporter()),
  instrumentations: [getNodeAutoInstrumentations()],
});

sdk.start();
```

### Environment Variables

```bash
export OTEL_EXPORTER_OTLP_ENDPOINT="https://ingest.cloud.firetiger.com"
export OTEL_EXPORTER_OTLP_HEADERS="Authorization=Bearer <YOUR_API_KEY>"
export OTEL_SERVICE_NAME="your-service-name"
```

### Run with Instrumentation

```bash
node --import ./instrumentation.js app.js
# or for TypeScript with tsx
node --import tsx --import ./instrumentation.ts app.ts
```

## Python

### Install Dependencies

```bash
pip install opentelemetry-distro opentelemetry-exporter-otlp
opentelemetry-bootstrap -a install
```

### Auto-Instrumentation (Recommended)

```bash
export OTEL_EXPORTER_OTLP_ENDPOINT="https://ingest.cloud.firetiger.com"
export OTEL_EXPORTER_OTLP_HEADERS="Authorization=Bearer <YOUR_API_KEY>"
export OTEL_SERVICE_NAME="your-service-name"

opentelemetry-instrument python app.py
```

### Manual Setup

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter

provider = TracerProvider()
processor = BatchSpanProcessor(OTLPSpanExporter())
provider.add_span_processor(processor)
trace.set_tracer_provider(provider)
```

## Go

### Install Dependencies

```bash
go get go.opentelemetry.io/otel \
  go.opentelemetry.io/otel/sdk \
  go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc \
  go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp
```

### Setup Code

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
    exporter, err := otlptracegrpc.New(ctx)
    if err != nil {
        return nil, err
    }

    res, _ := resource.New(ctx,
        resource.WithAttributes(
            semconv.ServiceName("your-service-name"),
        ),
    )

    tp := trace.NewTracerProvider(
        trace.WithBatcher(exporter),
        trace.WithResource(res),
    )
    otel.SetTracerProvider(tp)
    return tp, nil
}
```

### Environment Variables

```bash
export OTEL_EXPORTER_OTLP_ENDPOINT="ingest.cloud.firetiger.com:443"
export OTEL_EXPORTER_OTLP_HEADERS="Authorization=Bearer <YOUR_API_KEY>"
```

## Rust

### Add Dependencies to Cargo.toml

```toml
[dependencies]
opentelemetry = "0.21"
opentelemetry_sdk = { version = "0.21", features = ["rt-tokio"] }
opentelemetry-otlp = { version = "0.14", features = ["tonic"] }
tracing = "0.1"
tracing-opentelemetry = "0.22"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
```

### Setup Code

```rust
use opentelemetry::global;
use opentelemetry_otlp::WithExportConfig;
use opentelemetry_sdk::trace::TracerProvider;
use tracing_subscriber::layer::SubscriberExt;
use tracing_subscriber::util::SubscriberInitExt;

fn init_tracing() -> Result<(), Box<dyn std::error::Error>> {
    let exporter = opentelemetry_otlp::new_exporter()
        .tonic()
        .with_endpoint("https://ingest.cloud.firetiger.com");

    let provider = TracerProvider::builder()
        .with_batch_exporter(exporter, opentelemetry_sdk::runtime::Tokio)
        .build();

    global::set_tracer_provider(provider.clone());

    let telemetry = tracing_opentelemetry::layer()
        .with_tracer(provider.tracer("your-service-name"));

    tracing_subscriber::registry()
        .with(telemetry)
        .init();

    Ok(())
}
```

## Verification

After setup, verify telemetry is flowing:
1. Generate some traffic to your application
2. Check the Firetiger console for incoming traces
3. Use TraceQL to query: `{service.name="your-service-name"}`

## Common Issues

- **No data appearing**: Check that `OTEL_EXPORTER_OTLP_ENDPOINT` is set correctly
- **Authentication errors**: Verify your API key in `OTEL_EXPORTER_OTLP_HEADERS`
- **Missing spans**: Ensure auto-instrumentation libraries are installed for your frameworks
