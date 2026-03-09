# Monitoring Patterns

CloudWatch alarms, dashboards, audit logging, and alerting pipelines.

## CloudWatch Alarms

### Composite alarms

Combine multiple alarms to reduce noise. Only fire when multiple conditions are true:

```yaml
HighErrorRate:
  Type: AWS::CloudWatch::Alarm
  Properties:
    AlarmName: !Sub "${AWS::StackName}-high-error-rate"
    MetricName: 5XXError
    Namespace: AWS/ApplicationELB
    Statistic: Sum
    Period: 300
    EvaluationPeriods: 2
    Threshold: 10
    ComparisonOperator: GreaterThanThreshold
    Dimensions:
      - Name: LoadBalancer
        Value: !Ref ALBFullName

HighLatency:
  Type: AWS::CloudWatch::Alarm
  Properties:
    AlarmName: !Sub "${AWS::StackName}-high-latency"
    MetricName: TargetResponseTime
    Namespace: AWS/ApplicationELB
    ExtendedStatistic: p99
    Period: 300
    EvaluationPeriods: 2
    Threshold: 2
    ComparisonOperator: GreaterThanThreshold
    Dimensions:
      - Name: LoadBalancer
        Value: !Ref ALBFullName

ServiceDegraded:
  Type: AWS::CloudWatch::CompositeAlarm
  Properties:
    AlarmName: !Sub "${AWS::StackName}-service-degraded"
    AlarmRule: !Sub >-
      ALARM("${HighErrorRate}") AND ALARM("${HighLatency}")
    AlarmActions:
      - !Ref AlertsTopic
```

### Anomaly detection

Detect unusual patterns without static thresholds:

```yaml
AnomalyDetector:
  Type: AWS::CloudWatch::AnomalyDetector
  Properties:
    MetricName: Invocations
    Namespace: AWS/Lambda
    Stat: Sum
    Dimensions:
      - Name: FunctionName
        Value: !Ref MyFunction

InvocationAnomaly:
  Type: AWS::CloudWatch::Alarm
  Properties:
    AlarmName: !Sub "${AWS::StackName}-invocation-anomaly"
    Metrics:
      - Id: m1
        MetricStat:
          Metric:
            MetricName: Invocations
            Namespace: AWS/Lambda
            Dimensions:
              - Name: FunctionName
                Value: !Ref MyFunction
          Period: 300
          Stat: Sum
      - Id: ad1
        Expression: ANOMALY_DETECTION_BAND(m1, 2)
    ThresholdMetricId: ad1
    ComparisonOperator: LessThanLowerOrGreaterThanUpperThreshold
    EvaluationPeriods: 3
    AlarmActions:
      - !Ref AlertsTopic
```

### Essential alarms per service

| Service | Metric | Threshold | Reasoning |
|---------|--------|-----------|-----------|
| ALB | `5XXError` | > 1% of requests | Backend failures |
| ALB | `TargetResponseTime` (p99) | > 2s | Latency degradation |
| ECS | `CPUUtilization` | > 80% for 10min | Scale up or investigate |
| ECS | `MemoryUtilization` | > 85% for 10min | OOM risk |
| Lambda | `Errors` | > 5% of invocations | Function failures |
| Lambda | `Duration` (p99) | > 80% of timeout | Approaching timeout |
| RDS | `FreeStorageSpace` | < 10GB | Disk pressure |
| RDS | `DatabaseConnections` | > 80% of max | Connection exhaustion |
| DynamoDB | `ThrottledRequests` | > 0 sustained | Capacity issue |
| SQS | `ApproximateAgeOfOldestMessage` | > 5min | Consumer falling behind |

## CloudWatch Dashboards

### Service-level dashboard structure

```typescript
// CDK — service dashboard
new cloudwatch.Dashboard(this, "ServiceDashboard", {
  dashboardName: `${props.serviceName}-${props.environment}`,
  widgets: [
    // Row 1: Traffic overview
    [
      new cloudwatch.GraphWidget({
        title: "Request Rate",
        left: [albRequestCount],
        width: 8,
      }),
      new cloudwatch.GraphWidget({
        title: "Error Rate (5xx)",
        left: [alb5xxRate],
        width: 8,
      }),
      new cloudwatch.GraphWidget({
        title: "Latency (p50, p95, p99)",
        left: [latencyP50, latencyP95, latencyP99],
        width: 8,
      }),
    ],
    // Row 2: Compute
    [
      new cloudwatch.GraphWidget({
        title: "ECS CPU / Memory",
        left: [ecsCpu],
        right: [ecsMemory],
        width: 12,
      }),
      new cloudwatch.GraphWidget({
        title: "Task Count",
        left: [runningTasks, desiredTasks],
        width: 12,
      }),
    ],
    // Row 3: Dependencies
    [
      new cloudwatch.GraphWidget({
        title: "RDS Connections / CPU",
        left: [rdsConnections],
        right: [rdsCpu],
        width: 12,
      }),
      new cloudwatch.GraphWidget({
        title: "SQS Queue Depth",
        left: [sqsVisible, sqsInFlight],
        width: 12,
      }),
    ],
  ],
});
```

Group dashboards by service, not by AWS resource type. Each dashboard should answer: "Is this service healthy right now?"

## Audit and Compliance

### CloudTrail

- Enable in all regions (including unused ones — detect unauthorized activity).
- Store in a centralized S3 bucket with SSE-KMS encryption.
- Enable log file validation for tamper detection.
- Set up CloudWatch Logs integration for real-time alerting on sensitive API calls.

Critical events to monitor:
- `ConsoleLogin` without MFA
- `CreateUser`, `AttachUserPolicy`, `CreateAccessKey` — IAM changes
- `StopLogging` — someone disabling CloudTrail
- `AuthorizeSecurityGroupIngress` with `0.0.0.0/0` — public security group rules

### AWS Config

- Enable in all regions. Record all resource types.
- Managed rules for common compliance checks:

```yaml
S3BucketPublicAccessProhibited:
  Type: AWS::Config::ConfigRule
  Properties:
    ConfigRuleName: s3-bucket-public-access-prohibited
    Source:
      Owner: AWS
      SourceIdentifier: S3_BUCKET_PUBLIC_READ_PROHIBITED

EncryptedVolumes:
  Type: AWS::Config::ConfigRule
  Properties:
    ConfigRuleName: encrypted-volumes
    Source:
      Owner: AWS
      SourceIdentifier: ENCRYPTED_VOLUMES
```

## Alerting Pipeline

```
CloudWatch Alarm → SNS Topic → Multiple subscribers
                                ├── Lambda → Slack/Teams
                                ├── Email (oncall@company.com)
                                └── PagerDuty/OpsGenie integration
```

```yaml
AlertsTopic:
  Type: AWS::SNS::Topic
  Properties:
    TopicName: !Sub "${AWS::StackName}-alerts"
    KmsMasterKeyId: alias/aws/sns

EmailSubscription:
  Type: AWS::SNS::Subscription
  Properties:
    TopicArn: !Ref AlertsTopic
    Protocol: email
    Endpoint: oncall@company.com

SlackForwarder:
  Type: AWS::SNS::Subscription
  Properties:
    TopicArn: !Ref AlertsTopic
    Protocol: lambda
    Endpoint: !GetAtt SlackNotifierFunction.Arn
```

- Use separate SNS topics for `critical` (page) and `warning` (ticket) severity.
- Include runbook links in alarm descriptions.
- Test alerting pipeline regularly — an untested alert is a false sense of security.
