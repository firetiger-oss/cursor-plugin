# Vercel log & trace drains

Forward Vercel logs and traces to Firetiger via the Vercel API. The ingest endpoint is
`https://ingest.cloud.firetiger.com`; auth uses the Step 1 credentials as `$AUTH_HEADER` =
`base64(username:password)`.

## Prerequisites

- `which vercel` succeeds.
- A Vercel token (check `vercel whoami`, or ask the user for one).

## Find the project and team

```bash
curl -H "Authorization: Bearer $TOKEN" "https://api.vercel.com/v9/projects"
```

Grab the `$PROJECT_ID` for the project being onboarded.

## Create the logs drain

```bash
curl -X POST "https://api.vercel.com/v1/drains" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"name":"Send logs to Firetiger","projects":"some","projectIds":["'$PROJECT_ID'"],
       "schemas":{"log":{"version":"v1"}},
       "delivery":{"type":"http","endpoint":"https://ingest.cloud.firetiger.com/vercel/logs","encoding":"json",
                   "headers":{"Authorization":"Basic '$AUTH_HEADER'"}},
       "filter":{"version":"v2","filter":{"type":"basic","log":{"sources":["lambda","edge"]},
                 "deployment":{"environments":["production"]}}}}'
```

## Create the traces drain

```bash
curl -X POST "https://api.vercel.com/v1/drains" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"name":"Send traces to Firetiger","projects":"some","projectIds":["'$PROJECT_ID'"],
       "schemas":{"trace":{"version":"v1"}},
       "delivery":{"type":"otlphttp","endpoint":{"traces":"https://ingest.cloud.firetiger.com/v1/traces"},"encoding":"json",
                   "headers":{"Authorization":"Basic '$AUTH_HEADER'"}}}'
```

The logs drain targets production `lambda` and `edge` sources; adjust the `filter` block to include other
environments or sources as needed.

## Next steps

- **Verify** — after some traffic, run `SHOW TABLES;` with `firetiger-query` (data can lag a few minutes right after setup).
- **App-level traces** — drains capture platform logs/traces; for spans from your own code add the OTLP SDK via `firetiger-instrument`.
- **Other sources & integrations** — back to [ingest-sources.md](ingest-sources.md) for more drains, or [connections.md](connections.md) to connect GitHub/Slack and pull-based sources.
