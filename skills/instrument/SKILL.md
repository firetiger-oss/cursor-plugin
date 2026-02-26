---
name: firetiger-instrument
description: "Add OpenTelemetry instrumentation to applications for Firetiger observability"
user_invocable: true
user_invocable_description: "Instrument your application with OpenTelemetry for Firetiger"
---

# OpenTelemetry Instrumentation for Firetiger

You are an expert at instrumenting applications with OpenTelemetry to send telemetry data to Firetiger.

## Critical: Initialization Order

**OpenTelemetry MUST be initialized before any other imports.** If instrumented libraries (Express, HTTP, database clients) are imported before OpenTelemetry initializes, auto-instrumentation will not work. Always load the instrumentation file first.

## Framework Detection

Detect the application's technology stack by examining:
- `package.json` → Node.js/TypeScript
- `next.config.js` or `next.config.ts` → Next.js (use `@vercel/otel`)
- `requirements.txt`, `pyproject.toml`, `setup.py` → Python
- `go.mod` → Go
- `Cargo.toml` → Rust

## Next.js Applications

For Next.js apps, use the `@vercel/otel` package for simplified setup.

### Install Dependencies

```bash
npm install @vercel/otel @opentelemetry/sdk-logs @opentelemetry/api-logs @opentelemetry/instrumentation
```

### Create Instrumentation File

Create `instrumentation.ts` in the **root directory** (or `src/` if using src folder):

```typescript
import { registerOTel } from '@vercel/otel'

export function register() {
  registerOTel({ serviceName: 'your-service-name' })
}
```

### Environment Variables

```bash
export OTEL_EXPORTER_OTLP_ENDPOINT="https://ingest.cloud.firetiger.com"
export OTEL_EXPORTER_OTLP_HEADERS="Authorization=Bearer <YOUR_API_KEY>"
```

## Node.js / TypeScript (Non-Next.js)

### Install Dependencies

```bash
npm install @opentelemetry/api \
  @opentelemetry/sdk-node \
  @opentelemetry/auto-instrumentations-node \
  @opentelemetry/exporter-trace-otlp-proto \
  @opentelemetry/exporter-logs-otlp-proto \
  @opentelemetry/exporter-metrics-otlp-proto \
  @opentelemetry/sdk-metrics \
  @opentelemetry/sdk-logs
```

### Create Instrumentation File

Create `instrumentation.ts` (or `instrumentation.js`). This file MUST be loaded before any application code:

```typescript
import { NodeSDK } from '@opentelemetry/sdk-node';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-proto';
import { OTLPLogExporter } from '@opentelemetry/exporter-logs-otlp-proto';
import { OTLPMetricExporter } from '@opentelemetry/exporter-metrics-otlp-proto';
import { PeriodicExportingMetricReader } from '@opentelemetry/sdk-metrics';
import { BatchSpanProcessor } from '@opentelemetry/sdk-trace-base';
import { SimpleLogRecordProcessor } from '@opentelemetry/sdk-logs';

const traceExporter = new OTLPTraceExporter({
  compression: 'gzip', // Reduce bandwidth usage
});

const sdk = new NodeSDK({
  serviceName: process.env.OTEL_SERVICE_NAME || 'my-service',
  spanProcessor: new BatchSpanProcessor(traceExporter, {
    maxQueueSize: 2048,
    maxExportBatchSize: 512,
    scheduledDelayMillis: 5000,
  }),
  metricReader: new PeriodicExportingMetricReader({
    exporter: new OTLPMetricExporter(),
  }),
  logRecordProcessor: new SimpleLogRecordProcessor(new OTLPLogExporter()),
  instrumentations: [getNodeAutoInstrumentations()],
});

sdk.start();

// Graceful shutdown
process.on('SIGTERM', () => {
  sdk.shutdown().then(() => process.exit(0));
});
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
opentelemetry-bootstrap -a install  # Installs instrumentations for detected libraries
```

### Auto-Instrumentation (Recommended)

```bash
export OTEL_EXPORTER_OTLP_ENDPOINT="https://ingest.cloud.firetiger.com"
export OTEL_EXPORTER_OTLP_HEADERS="Authorization=Bearer <YOUR_API_KEY>"
export OTEL_SERVICE_NAME="your-service-name"

opentelemetry-instrument python app.py
```

### Manual Setup (for more control)

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.resources import Resource
from opentelemetry.semconv.resource import ResourceAttributes

resource = Resource.create({
    ResourceAttributes.SERVICE_NAME: "your-service-name",
    ResourceAttributes.DEPLOYMENT_ENVIRONMENT: "production",
})

provider = TracerProvider(resource=resource)
processor = BatchSpanProcessor(OTLPSpanExporter(
    endpoint="https://ingest.cloud.firetiger.com",
    headers={"Authorization": "Bearer <YOUR_API_KEY>"},
))
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

## Adding Custom Spans

Auto-instrumentation covers common libraries, but add manual spans for business-critical operations.

### Node.js Custom Spans

```typescript
import { trace, SpanStatusCode } from '@opentelemetry/api';

const tracer = trace.getTracer('your-service-name');

async function processOrder(orderId: string, customerId: string) {
  return await tracer.startActiveSpan('processOrder', async (span) => {
    // Add business context as attributes
    span.setAttribute('order.id', orderId);
    span.setAttribute('customer.id', customerId);

    try {
      const result = await executeOrder(orderId);
      span.addEvent('order.completed', { 'order.total': result.total });
      return result;
    } catch (error) {
      span.setStatus({ code: SpanStatusCode.ERROR, message: error.message });
      span.recordException(error);
      throw error;
    } finally {
      span.end();
    }
  });
}
```

### Python Custom Spans

```python
from opentelemetry import trace

tracer = trace.get_tracer("your-service-name")

def process_order(order_id: str, customer_id: str):
    with tracer.start_as_current_span("process_order") as span:
        span.set_attribute("order.id", order_id)
        span.set_attribute("customer.id", customer_id)

        try:
            result = execute_order(order_id)
            span.add_event("order.completed", {"order.total": result.total})
            return result
        except Exception as e:
            span.set_status(trace.StatusCode.ERROR, str(e))
            span.record_exception(e)
            raise
```

## Log Correlation

Inject trace context into logs to correlate logs with traces in Firetiger.

### Node.js with Winston

```typescript
import winston from 'winston';
import { trace } from '@opentelemetry/api';

const logger = winston.createLogger({
  format: winston.format.combine(
    winston.format((info) => {
      const span = trace.getActiveSpan();
      if (span) {
        const context = span.spanContext();
        info.trace_id = context.traceId;
        info.span_id = context.spanId;
      }
      return info;
    })(),
    winston.format.json()
  ),
  transports: [new winston.transports.Console()],
});
```

### Python with structlog

```python
import structlog
from opentelemetry import trace

def add_trace_context(logger, method_name, event_dict):
    span = trace.get_current_span()
    if span.is_recording():
        ctx = span.get_span_context()
        event_dict["trace_id"] = format(ctx.trace_id, "032x")
        event_dict["span_id"] = format(ctx.span_id, "016x")
    return event_dict

structlog.configure(processors=[add_trace_context, structlog.processors.JSONRenderer()])
```

## Best Practices

### Semantic Conventions

Use OpenTelemetry semantic conventions for consistent attribute naming:
- `http.method`, `http.status_code`, `http.route` for HTTP
- `db.system`, `db.statement`, `db.name` for databases
- `rpc.service`, `rpc.method` for RPC calls
- Use `app.` prefix for custom business attributes

### Performance Optimization

1. **Use BatchSpanProcessor** in production (not SimpleSpanProcessor)
2. **Enable compression** (`gzip`) to reduce bandwidth
3. **Configure appropriate batch sizes** based on your traffic volume
4. **Set queue limits** to prevent memory issues under load

### What to Instrument

1. Start with auto-instrumentation for broad coverage
2. Add manual spans for:
   - Business-critical operations (checkout, payment)
   - Custom code not covered by auto-instrumentation
   - Operations you need to measure for SLOs

### Attribute Guidelines

- Only include relevant attributes for each span
- Avoid high-cardinality attributes (user IDs as attribute values are fine, but not as attribute names)
- Never include sensitive data (passwords, tokens, PII) in attributes

## Verification

After setup, verify telemetry is flowing:
1. Generate some traffic to your application
2. Check the Firetiger console for incoming traces
3. Use SQL to query: `SELECT * FROM traces WHERE service_name = 'your-service-name' LIMIT 10`

## Common Issues

- **No data appearing**: Check that `OTEL_EXPORTER_OTLP_ENDPOINT` is set correctly
- **Authentication errors**: Verify your API key in `OTEL_EXPORTER_OTLP_HEADERS`
- **Missing spans**: Ensure auto-instrumentation libraries are installed for your frameworks
- **Spans not connected**: Verify context propagation is working (W3C Trace Context headers)
- **High memory usage**: Reduce batch sizes or enable sampling
