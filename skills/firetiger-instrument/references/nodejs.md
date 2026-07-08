# Node.js & Next.js instrumentation

All snippets assume the `OTEL_EXPORTER_OTLP_*` env vars from the SKILL.md are set (Basic auth header from
`get_ingest_credentials`).

## Next.js

Next.js has first-class OpenTelemetry support via `@vercel/otel`.

```bash
npm install @vercel/otel @opentelemetry/sdk-logs @opentelemetry/api-logs @opentelemetry/instrumentation
```

Create `instrumentation.ts` in the **project root** (or `src/` if you use a src folder):

```typescript
import { registerOTel } from '@vercel/otel'

export function register() {
  registerOTel({ serviceName: 'your-service-name' })
}
```

The `OTEL_EXPORTER_OTLP_ENDPOINT` / `OTEL_EXPORTER_OTLP_HEADERS` env vars drive the exporter. Next.js loads
`instrumentation.ts` before app code automatically — no `--import` flag needed.

## Node.js / TypeScript (non-Next.js)

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

Create `instrumentation.ts` — it MUST load before any application code:

```typescript
import { NodeSDK } from '@opentelemetry/sdk-node';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-proto';
import { OTLPLogExporter } from '@opentelemetry/exporter-logs-otlp-proto';
import { OTLPMetricExporter } from '@opentelemetry/exporter-metrics-otlp-proto';
import { PeriodicExportingMetricReader } from '@opentelemetry/sdk-metrics';
import { BatchSpanProcessor } from '@opentelemetry/sdk-trace-base';
import { SimpleLogRecordProcessor } from '@opentelemetry/sdk-logs';

const traceExporter = new OTLPTraceExporter({ compression: 'gzip' });

const sdk = new NodeSDK({
  serviceName: process.env.OTEL_SERVICE_NAME || 'my-service',
  spanProcessor: new BatchSpanProcessor(traceExporter, {
    maxQueueSize: 2048,
    maxExportBatchSize: 512,
    scheduledDelayMillis: 5000,
  }),
  metricReader: new PeriodicExportingMetricReader({ exporter: new OTLPMetricExporter() }),
  logRecordProcessor: new SimpleLogRecordProcessor(new OTLPLogExporter()),
  instrumentations: [getNodeAutoInstrumentations()],
});

sdk.start();

// Flush buffered spans before exit
process.on('SIGTERM', () => { sdk.shutdown().then(() => process.exit(0)); });
```

Run with the instrumentation loaded first:

```bash
node --import ./instrumentation.js app.js
# TypeScript with tsx:
node --import tsx --import ./instrumentation.ts app.ts
```

## Simpler traces-only setup

If you only need traces, `@opentelemetry/exporter-trace-otlp-http` plus auto-instrumentations is enough:

```bash
npm install @opentelemetry/api @opentelemetry/sdk-node \
  @opentelemetry/auto-instrumentations-node @opentelemetry/exporter-trace-otlp-http
```

```typescript
import { NodeSDK } from '@opentelemetry/sdk-node'
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node'
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http'

const sdk = new NodeSDK({
  serviceName: process.env.OTEL_SERVICE_NAME || 'my-service',
  traceExporter: new OTLPTraceExporter({
    url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT,
    headers: {
      Authorization: process.env.OTEL_EXPORTER_OTLP_HEADERS?.replace('Authorization=', '') || '',
    },
  }),
  instrumentations: [getNodeAutoInstrumentations()],
})

sdk.start()
```

## Next steps

- **Verify** — generate traffic, then query `"opentelemetry/traces/your-service-name"` with `firetiger-query` (`SHOW TABLES;` first).
- **Custom spans & log correlation** — instrument business-critical operations and join logs to traces in [custom-spans.md](custom-spans.md).
- **Full onboarding** — connect integrations and create a monitoring agent with `firetiger-setup`.
