# Python instrumentation

Assumes the `OTEL_EXPORTER_OTLP_*` env vars from the SKILL.md are set (Basic auth header from
`get_ingest_credentials`).

## Auto-instrumentation (recommended)

```bash
pip install opentelemetry-distro opentelemetry-exporter-otlp
opentelemetry-bootstrap -a install   # installs instrumentations for detected libraries
```

```bash
export OTEL_EXPORTER_OTLP_ENDPOINT="https://ingest.cloud.firetiger.com"
export OTEL_EXPORTER_OTLP_HEADERS="Authorization=Basic <base64(username:password)>"
export OTEL_SERVICE_NAME="your-service-name"

opentelemetry-instrument python app.py
```

`opentelemetry-instrument` wraps your process and initializes the SDK before your code runs — no code changes
required for auto-instrumented libraries (Flask, Django, FastAPI, requests, psycopg, etc.).

## Manual setup (more control)

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
    headers={"Authorization": "Basic <base64(username:password)>"},
))
provider.add_span_processor(processor)
trace.set_tracer_provider(provider)
```

Call this at the very top of your entrypoint, before importing instrumented libraries.

## Next steps

- **Verify** — generate traffic, then query `"opentelemetry/traces/your-service-name"` with `firetiger-query` (`SHOW TABLES;` first).
- **Custom spans & log correlation** — instrument business-critical operations and join logs to traces in [custom-spans.md](custom-spans.md).
- **Full onboarding** — connect integrations and create a monitoring agent with `firetiger-setup`.
