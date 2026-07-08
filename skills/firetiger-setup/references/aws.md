# AWS CloudWatch logs

Forward CloudWatch logs to Firetiger by deploying the onboarding CloudFormation stack (ingest + IAM role). The
ingest endpoint is `https://ingest.cloud.firetiger.com`; use the Step 1 credentials `$USERNAME` / `$PASSWORD`.
Requires `which aws` and a chosen `$REGION`.

## Deploy the onboarding stack

```bash
aws cloudformation create-stack \
  --stack-name firetiger-cloudwatch-logs \
  --template-url https://firetiger-public-$REGION.s3.$REGION.amazonaws.com/ingest/aws/cloudwatch/logs/ingest-and-iam-onboarding.yaml \
  --parameters \
    ParameterKey=FiretigerEndpoint,ParameterValue=https://ingest.cloud.firetiger.com \
    ParameterKey=FiretigerUsername,ParameterValue=$USERNAME \
    ParameterKey=FiretigerPassword,ParameterValue=$PASSWORD \
    ParameterKey=FiretigerExternalId,ParameterValue=$(uuidgen) \
  --capabilities CAPABILITY_NAMED_IAM \
  --region $REGION
```

## Wait, then read the outputs

```bash
aws cloudformation wait stack-create-complete --stack-name firetiger-cloudwatch-logs --region $REGION
aws cloudformation describe-stacks --stack-name firetiger-cloudwatch-logs \
  --query 'Stacks[0].Outputs' --region $REGION
```

The outputs include the created IAM Role ARN — surface it in the summary so the user can confirm the
cross-account role in the Firetiger dashboard if prompted.

## Other AWS sources

CloudWatch Logs is the common case, but Firetiger ingests several other AWS streams (all Basic auth on the
`https://ingest.cloud.firetiger.com` host, typically delivered via **Kinesis Firehose**, which also supports a dedicated Firehose
auth mode):

| Source | Endpoint | Delivery |
|--------|----------|----------|
| CloudWatch Logs | `/aws/cloudwatch/logs` | CloudWatch subscription → Firehose |
| ALB access logs | `/AWSLogs/...` | S3 access-log delivery |
| CloudFront logs | `/aws/cloudfront/kinesis` | Kinesis Firehose |
| ECS task state changes | `/aws/eventbridge/ecs-task-state-change`, `/aws/events` | EventBridge rule |
| Kafka / MSK broker logs | `/aws/kinesis/kafka/logs`, `/aws/kinesis/msk/logs` | Kinesis Firehose |

Point a Firehose delivery stream (HTTP endpoint destination) or EventBridge API destination at the relevant
path with the Basic auth header. For richer app telemetry, prefer the OTLP SDK (`firetiger-instrument`)
alongside these infra logs.

## Next steps

- **Verify** — after some traffic, run `SHOW TABLES;` with `firetiger-query` (data can lag a few minutes right after setup).
- **Query AWS without ingesting** — AWS is also a **pull-based** connection (CloudWatch metrics, cost data) you query in place; see [connections.md](connections.md).
- **Other sources** — back to [ingest-sources.md](ingest-sources.md) for the full drain menu.
