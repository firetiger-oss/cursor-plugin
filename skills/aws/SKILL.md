---
name: firetiger-aws
description: "Set up AWS integrations for Firetiger - CloudWatch Logs, ECS events"
user_invocable: true
user_invocable_description: "Configure AWS integrations to send CloudWatch logs and ECS events to Firetiger"
---

# AWS Integrations for Firetiger

You are an expert at configuring AWS services to forward telemetry data to Firetiger.

## CloudWatch Logs Integration

Forward CloudWatch Logs to Firetiger using subscription filters and Lambda.

### Architecture

```
CloudWatch Logs → Subscription Filter → Lambda → Firetiger OTLP Endpoint
```

### Terraform Configuration

```hcl
# Lambda function to forward logs
resource "aws_lambda_function" "firetiger_forwarder" {
  function_name = "firetiger-log-forwarder"
  runtime       = "nodejs20.x"
  handler       = "index.handler"
  role          = aws_iam_role.lambda_role.arn
  timeout       = 30
  memory_size   = 256

  environment {
    variables = {
      FIRETIGER_ENDPOINT = "https://ingest.cloud.firetiger.com/v1/logs"
      FIRETIGER_API_KEY  = var.firetiger_api_key
    }
  }

  filename         = data.archive_file.lambda_zip.output_path
  source_code_hash = data.archive_file.lambda_zip.output_base64sha256
}

# IAM role for Lambda
resource "aws_iam_role" "lambda_role" {
  name = "firetiger-forwarder-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "lambda.amazonaws.com"
      }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "lambda_basic" {
  role       = aws_iam_role.lambda_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

# Permission for CloudWatch Logs to invoke Lambda
resource "aws_lambda_permission" "cloudwatch" {
  statement_id  = "AllowCloudWatchLogs"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.firetiger_forwarder.function_name
  principal     = "logs.amazonaws.com"
  source_arn    = "${aws_cloudwatch_log_group.app_logs.arn}:*"
}

# Subscription filter
resource "aws_cloudwatch_log_subscription_filter" "firetiger" {
  name            = "firetiger-forwarder"
  log_group_name  = aws_cloudwatch_log_group.app_logs.name
  filter_pattern  = ""  # Empty pattern = all logs
  destination_arn = aws_lambda_function.firetiger_forwarder.arn
}
```

### Lambda Forwarder Code

```javascript
// index.js
const zlib = require('zlib');
const https = require('https');

exports.handler = async (event) => {
  const payload = Buffer.from(event.awslogs.data, 'base64');
  const parsed = JSON.parse(zlib.gunzipSync(payload).toString('utf8'));

  const logs = parsed.logEvents.map(logEvent => ({
    timestamp: logEvent.timestamp * 1000000, // Convert to nanoseconds
    body: logEvent.message,
    attributes: {
      'aws.cloudwatch.log_group': parsed.logGroup,
      'aws.cloudwatch.log_stream': parsed.logStream,
      'aws.cloudwatch.owner': parsed.owner,
    },
    resource: {
      'service.name': parsed.logGroup.split('/').pop(),
      'cloud.provider': 'aws',
      'cloud.region': process.env.AWS_REGION,
    }
  }));

  const body = JSON.stringify({ logs });

  await fetch(process.env.FIRETIGER_ENDPOINT, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${process.env.FIRETIGER_API_KEY}`,
    },
    body,
  });

  return { statusCode: 200 };
};
```

## ECS Task State Change Events

Capture ECS task lifecycle events for observability.

### Architecture

```
ECS → EventBridge → Lambda → Firetiger OTLP Endpoint
```

### Terraform Configuration

```hcl
# EventBridge rule for ECS task state changes
resource "aws_cloudwatch_event_rule" "ecs_task_state" {
  name        = "firetiger-ecs-task-state"
  description = "Capture ECS task state changes"

  event_pattern = jsonencode({
    source      = ["aws.ecs"]
    detail-type = ["ECS Task State Change"]
  })
}

# Target the Lambda forwarder
resource "aws_cloudwatch_event_target" "ecs_to_firetiger" {
  rule      = aws_cloudwatch_event_rule.ecs_task_state.name
  target_id = "firetiger-forwarder"
  arn       = aws_lambda_function.firetiger_ecs_forwarder.arn
}

# Lambda for ECS events
resource "aws_lambda_function" "firetiger_ecs_forwarder" {
  function_name = "firetiger-ecs-forwarder"
  runtime       = "nodejs20.x"
  handler       = "index.handler"
  role          = aws_iam_role.lambda_role.arn
  timeout       = 30

  environment {
    variables = {
      FIRETIGER_ENDPOINT = "https://ingest.cloud.firetiger.com/v1/logs"
      FIRETIGER_API_KEY  = var.firetiger_api_key
    }
  }

  filename         = data.archive_file.ecs_lambda_zip.output_path
  source_code_hash = data.archive_file.ecs_lambda_zip.output_base64sha256
}

# Permission for EventBridge to invoke Lambda
resource "aws_lambda_permission" "eventbridge" {
  statement_id  = "AllowEventBridge"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.firetiger_ecs_forwarder.function_name
  principal     = "events.amazonaws.com"
  source_arn    = aws_cloudwatch_event_rule.ecs_task_state.arn
}
```

### ECS Event Forwarder Code

```javascript
// index.js
exports.handler = async (event) => {
  const detail = event.detail;

  const log = {
    timestamp: Date.now() * 1000000,
    body: JSON.stringify({
      event: 'ECS Task State Change',
      lastStatus: detail.lastStatus,
      desiredStatus: detail.desiredStatus,
      stoppedReason: detail.stoppedReason,
    }),
    severity: detail.lastStatus === 'STOPPED' ? 'WARN' : 'INFO',
    attributes: {
      'aws.ecs.task_arn': detail.taskArn,
      'aws.ecs.task_definition_arn': detail.taskDefinitionArn,
      'aws.ecs.cluster_arn': detail.clusterArn,
      'aws.ecs.last_status': detail.lastStatus,
      'aws.ecs.desired_status': detail.desiredStatus,
      'aws.ecs.stopped_reason': detail.stoppedReason || '',
    },
    resource: {
      'service.name': detail.group?.replace('service:', '') || 'ecs-task',
      'cloud.provider': 'aws',
      'cloud.region': event.region,
    }
  };

  await fetch(process.env.FIRETIGER_ENDPOINT, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${process.env.FIRETIGER_API_KEY}`,
    },
    body: JSON.stringify({ logs: [log] }),
  });

  return { statusCode: 200 };
};
```

## CloudFormation Alternative

For CloudFormation users, equivalent resources:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Firetiger CloudWatch Logs Integration

Parameters:
  FiretigerApiKey:
    Type: String
    NoEcho: true

Resources:
  ForwarderFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: firetiger-log-forwarder
      Runtime: nodejs20.x
      Handler: index.handler
      Timeout: 30
      MemorySize: 256
      Environment:
        Variables:
          FIRETIGER_ENDPOINT: https://ingest.cloud.firetiger.com/v1/logs
          FIRETIGER_API_KEY: !Ref FiretigerApiKey
      Role: !GetAtt LambdaRole.Arn
      Code:
        ZipFile: |
          // Lambda code here

  LambdaRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

  SubscriptionFilter:
    Type: AWS::Logs::SubscriptionFilter
    Properties:
      LogGroupName: !Ref LogGroupName
      FilterPattern: ''
      DestinationArn: !GetAtt ForwarderFunction.Arn
```

## Verification

After deployment:

1. **Check Lambda logs** for any errors in the forwarder
2. **Query in Firetiger**:
   ```traceql
   { resource.cloud.provider = "aws" }
   ```
3. **Search logs**:
   ```
   aws.cloudwatch.log_group = "/aws/lambda/my-function"
   ```

## Troubleshooting

- **No logs appearing**: Check Lambda execution role has CloudWatch Logs read permissions
- **Auth errors**: Verify FIRETIGER_API_KEY is correctly set
- **Throttling**: Increase Lambda concurrency or add batching
- **Missing fields**: Ensure log format matches expected OTLP structure
