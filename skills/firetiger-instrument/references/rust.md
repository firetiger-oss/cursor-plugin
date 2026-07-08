# Rust instrumentation

Uses `tracing` + `tracing-opentelemetry` + the OTLP exporter. Assumes the `OTEL_EXPORTER_OTLP_*` env vars
from the SKILL.md are set (Basic auth header from `get_ingest_credentials`).

```toml
[dependencies]
opentelemetry = "0.21"
opentelemetry_sdk = { version = "0.21", features = ["rt-tokio"] }
opentelemetry-otlp = { version = "0.14", features = ["tonic"] }
tracing = "0.1"
tracing-opentelemetry = "0.22"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
```

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

Call `init_tracing()` early in `main` (inside the Tokio runtime). Instrument functions with
`#[tracing::instrument]` or emit spans with `tracing::span!` — they flow through the OpenTelemetry layer to
Firetiger.

## Next steps

- **Verify** — generate traffic, then query `"opentelemetry/traces/your-service-name"` with `firetiger-query` (`SHOW TABLES;` first).
- **Custom spans & log correlation** — the [custom-spans.md](custom-spans.md) patterns (Node/Python shown) map directly onto `tracing` spans and fields.
- **Full onboarding** — connect integrations and create a monitoring agent with `firetiger-setup`.
