# GCP Cloud Logging

Forward GCP Cloud Logging to Firetiger via a logging sink → Pub/Sub topic → Cloud Function forwarder. The
ingest endpoint is `https://ingest.cloud.firetiger.com`; use the Step 1 credentials `$USERNAME` / `$PASSWORD`.
Requires `which gcloud` and a chosen `$REGION`.

```bash
# Current project
PROJECT=$(gcloud config get-value project)

# Enable required APIs
gcloud services enable cloudfunctions.googleapis.com pubsub.googleapis.com logging.googleapis.com \
  run.googleapis.com cloudbuild.googleapis.com artifactregistry.googleapis.com eventarc.googleapis.com

# Pub/Sub topic
gcloud pubsub topics create firetiger-cloud-logs

# Logging sink → topic
gcloud logging sinks create firetiger-cloud-logs \
  pubsub.googleapis.com/projects/$PROJECT/topics/firetiger-cloud-logs

# Grant the sink's writer identity publish permission
SINK_SA=$(gcloud logging sinks describe firetiger-cloud-logs --format='value(writerIdentity)')
gcloud pubsub topics add-iam-policy-binding firetiger-cloud-logs \
  --member="$SINK_SA" --role="roles/pubsub.publisher"

# Deploy the forwarder Cloud Function
gcloud functions deploy firetiger-cloud-logs-forwarder \
  --gen2 --runtime=python313 \
  --trigger-topic=firetiger-cloud-logs \
  --entry-point=process_log_entry \
  --set-env-vars="FT_EXPORTER_ENDPOINT=https://ingest.cloud.firetiger.com,FT_EXPORTER_BASIC_AUTH_USERNAME=$USERNAME,FT_EXPORTER_BASIC_AUTH_PASSWORD=$PASSWORD" \
  --source=gs://firetiger-public/ingest/gcp/cloud-logging/function.zip \
  --region=$REGION
```

By default the sink forwards all project logs. Add a `--log-filter='...'` to the `gcloud logging sinks create`
command to scope which logs are exported.

## Delivery options

Firetiger accepts GCP logs two ways:
- **Pub/Sub pull** — the forwarder function above, driven by the logging sink → Pub/Sub topic (shown here).
- **HTTP push** — a Log Router sink or Cloud Function can POST Cloud Logging `LogEntry` payloads directly to
  `https://ingest.cloud.firetiger.com/gcp/cloud-logging` with the Basic auth header, skipping the pull subscriber.

Choose HTTP push for the simplest setup; use the Pub/Sub path when you want buffering/replay in front of
Firetiger. For app traces/metrics, add the OTLP SDK (`firetiger-instrument`).

## Next steps

- **Verify** — after some traffic, run `SHOW TABLES;` with `firetiger-query` (data can lag a few minutes right after setup).
- **Query GCP without ingesting** — GCP is also a **pull-based** connection (Cloud Monitoring metrics via PromQL) you query in place; see [connections.md](connections.md).
- **Other sources** — back to [ingest-sources.md](ingest-sources.md) for the full drain menu.
