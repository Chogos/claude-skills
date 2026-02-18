---
name: writing-aws-infrastructure
description: AWS infrastructure best practices. Use when creating, modifying, or reviewing CloudFormation templates, CDK stacks, IAM policies, Lambda functions, ECS task definitions, VPC configurations, or any AWS resource definitions.
---

# AWS Infrastructure Best Practices

## IAM Policies

Never use `*` in Action or Resource. Every policy must scope to the minimum actions and resources required.

### Least privilege

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadInvoiceBucket",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::acme-invoices",
        "arn:aws:s3:::acme-invoices/*"
      ],
      "Condition": {
        "StringEquals": {
          "aws:PrincipalOrgID": "o-abc123"
        }
      }
    }
  ]
}
```

### Role design

- One IAM role per workload — never share roles across services.
- Pin trust policies to specific accounts and services:
  ```json
  {
    "Effect": "Allow",
    "Principal": { "Service": "lambda.amazonaws.com" },
    "Action": "sts:AssumeRole",
    "Condition": {
      "ArnLike": {
        "aws:SourceArn": "arn:aws:lambda:eu-west-1:111111111111:function:my-func"
      }
    }
  }
  ```
- Use permission boundaries to cap developer-created roles.
- Prefer AWS managed policies for common patterns (e.g., `AmazonDynamoDBReadOnlyAccess`), then tighten with inline.

### Policy structure

- Always include `Sid` — readable identifier per statement.
- Deny-first: explicit `Deny` statements override any `Allow`.
- Managed policies over inline for reusability and 6144-char inline limit.
- Max 10 managed policies per role (default). Request increase if needed.

## S3

Block public access at the account level. Repeat at the bucket level for defense-in-depth.

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Resources:
  InvoiceBucket:
    Type: AWS::S3::Bucket
    DeletionPolicy: Retain
    Properties:
      BucketName: !Sub "${AWS::AccountId}-invoices"
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: aws:kms
              KMSMasterKeyID: !Ref BucketKey
      VersioningConfiguration:
        Status: Enabled
      LifecycleConfiguration:
        Rules:
          - Id: TransitionToIA
            Status: Enabled
            Transitions:
              - StorageClass: STANDARD_IA
                TransitionInDays: 30
              - StorageClass: GLACIER
                TransitionInDays: 90
      LoggingConfiguration:
        DestinationBucketName: !Ref AccessLogsBucket
        LogFilePrefix: invoices/
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true

  InvoiceBucketPolicy:
    Type: AWS::S3::BucketPolicy
    Properties:
      Bucket: !Ref InvoiceBucket
      PolicyDocument:
        Statement:
          - Sid: DenyInsecureTransport
            Effect: Deny
            Principal: "*"
            Action: "s3:*"
            Resource:
              - !GetAtt InvoiceBucket.Arn
              - !Sub "${InvoiceBucket.Arn}/*"
            Condition:
              Bool:
                "aws:SecureTransport": "false"
```

- SSE-S3 (`AES256`) for default encryption, SSE-KMS when you need key rotation control or cross-account access.
- Enable versioning on all buckets holding business data.
- Access logging to a dedicated bucket — never log to the same bucket.

## Lambda

Pin the runtime version. Use `provided.al2023` for custom runtimes.

```typescript
// CDK — Lambda with best practices
import { Duration, RemovalPolicy } from "aws-cdk-lib";
import { Runtime, Architecture, Tracing } from "aws-cdk-lib/aws-lambda";
import { NodejsFunction } from "aws-cdk-lib/aws-lambda-nodejs";
import { RetentionDays } from "aws-cdk-lib/aws-logs";
import { Secret } from "aws-cdk-lib/aws-secretsmanager";

const apiSecret = Secret.fromSecretNameV2(this, "ApiSecret", "prod/api-key");

const processOrder = new NodejsFunction(this, "ProcessOrder", {
  entry: "src/handlers/process-order.ts",
  handler: "handler",
  runtime: Runtime.NODEJS_22_X,
  architecture: Architecture.ARM_64,
  memorySize: 512,
  timeout: Duration.seconds(30),
  tracing: Tracing.ACTIVE,
  reservedConcurrentExecutions: 100,
  environment: {
    TABLE_NAME: ordersTable.tableName,
    REGION: this.region,
  },
  logRetention: RetentionDays.TWO_WEEKS,
  bundling: {
    minify: true,
    sourceMap: true,
  },
});

apiSecret.grantRead(processOrder);
ordersTable.grantReadWriteData(processOrder);
```

- **Memory**: start at 512 MB, profile with Lambda Power Tuning, then right-size.
- **Timeout**: set to expected P99 + buffer, never max (900s) unless truly needed.
- **Secrets**: read from Secrets Manager or SSM Parameter Store at cold start, cache in-memory. Never bake into env vars or code.
- **Layers**: shared deps and extensions. Max 5 layers, 250 MB unzipped total.
- **Reserved concurrency**: protect downstream services from Lambda burst scaling.

## ECS / Fargate

### Task definitions

```json
{
  "family": "api-service",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::111111111111:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::111111111111:role/api-service-task-role",
  "containerDefinitions": [
    {
      "name": "api",
      "image": "111111111111.dkr.ecr.eu-west-1.amazonaws.com/api:1.2.3",
      "portMappings": [{ "containerPort": 8080, "protocol": "tcp" }],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/api-service",
          "awslogs-region": "eu-west-1",
          "awslogs-stream-prefix": "api"
        }
      },
      "secrets": [
        {
          "name": "DB_PASSWORD",
          "valueFrom": "arn:aws:secretsmanager:eu-west-1:111111111111:secret:prod/db-password"
        }
      ],
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 60
      }
    }
  ]
}
```

- Pin image tags to immutable digests or semver — never `latest`.
- Always `awsvpc` network mode on Fargate.
- Separate `executionRoleArn` (ECR pull, log writes) from `taskRoleArn` (app-level AWS access).
- Inject secrets via `secrets` block — they resolve at task start, not baked into the image.

### Service config

- **Circuit breaker**: `deploymentCircuitBreaker: { enable: true, rollback: true }`.
- **Deployment**: `maximumPercent: 200`, `minimumHealthyPercent: 100` for zero-downtime rolling deploys.
- **Health check grace period**: at least as long as your container startup time — prevents premature task kills.
- **Platform version**: pin to `LATEST` or specific version (`1.4.0`).

## Networking

### VPC design

- Minimum 2 AZs, prefer 3 for production.
- Three subnet tiers: public (ALB, NAT GW), private (compute), isolated (databases).
- CIDR planning: `/16` VPC, `/20` subnets. Leave room for expansion.
- One NAT Gateway per AZ for HA. Single NAT Gateway acceptable for dev/staging.

### Security groups

- Stateful — return traffic is auto-allowed.
- Reference by security group ID, never by CIDR, for intra-VPC rules:
  ```yaml
  EcsSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: ECS tasks
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 8080
          ToPort: 8080
          SourceSecurityGroupId: !Ref AlbSecurityGroup
  ```
- One security group per role (ALB, app, database). Avoid catch-all groups.
- No `0.0.0.0/0` ingress except ALB ports 80/443.

### VPC endpoints

- **Gateway endpoints** (free): S3, DynamoDB. Always add to route tables.
- **Interface endpoints**: ECR (`ecr.api`, `ecr.dkr`), Secrets Manager, CloudWatch Logs, STS.
- Interface endpoints incur hourly + data charges — evaluate cost vs NAT Gateway savings.

## CloudFormation / CDK

### CloudFormation templates

- Use `Parameters` for environment-specific values. Default nothing that varies per environment.
- `DeletionPolicy: Retain` on stateful resources (S3, RDS, DynamoDB).
- `Fn::Sub` over `Fn::Join` for readability:
  ```yaml
  Value: !Sub "arn:aws:s3:::${BucketName}/*"
  ```
- Organize: Parameters → Mappings → Conditions → Resources → Outputs.
- Export outputs sparingly — cross-stack refs create tight coupling. Prefer SSM parameters.

### CDK conventions

- One construct per file. File name matches construct: `api-gateway.ts` → `ApiGateway`.
- Define a `Props` interface extending `StackProps` for every stack:
  ```typescript
  interface ApiStackProps extends StackProps {
    vpcId: string;
    certificateArn: string;
  }
  ```
- `RemovalPolicy.RETAIN` on stateful resources. `RemovalPolicy.DESTROY` only in dev stacks.
- Apply tags at the `App` level:
  ```typescript
  Tags.of(app).add("Project", "acme");
  Tags.of(app).add("Environment", props.environment);
  ```
- Run `cdk-nag` in CI. Add suppressions with justification, never disable globally.

## Cost Optimization

- **Tagging**: every resource must have `Project`, `Environment`, `Owner`. Use tag policies in AWS Organizations.
- **Cost visibility**: enable Cost Explorer, set up AWS Budgets with alerts at 80%/100% of expected spend. Use Trusted Advisor for idle resource detection.
- **Right-sizing**: use Compute Optimizer recommendations for EC2, Lambda, and ECS. Review monthly — workloads change.
- **S3**: lifecycle rules to transition to IA at 30 days, Glacier at 90. Enable Intelligent-Tiering for unpredictable access patterns.
- **Lambda**: right-size memory with Power Tuning. Use ARM (`arm64`) for ~20% cost reduction. Minimize cold starts to reduce wasted execution time.
- **Fargate Spot**: use for fault-tolerant workloads (queue processors, batch jobs). Not for user-facing services.
- **NAT Gateway**: $0.045/hr per gateway. Consolidate to one for non-prod. Use VPC endpoints to reduce NAT traffic for AWS service calls.
- **VPC endpoints**: gateway endpoints (S3, DynamoDB) are free — always enable. Interface endpoints are ~$7/mo per AZ — cost-effective when they replace significant NAT traffic.
- **Reserved capacity**: Savings Plans for predictable Lambda/Fargate usage. Compute Savings Plans are more flexible than EC2 Reserved Instances.
- **Database**: use Aurora Serverless v2 for variable workloads. DynamoDB on-demand for unpredictable traffic, provisioned + auto-scaling once patterns stabilize.

## Encryption

### At rest

- KMS customer-managed keys (CMK) for data you control. AWS-managed keys for everything else.
- Enable default encryption on S3, EBS, RDS, DynamoDB, SQS, SNS.
- DynamoDB: AWS-owned key (free, default) or CMK (for cross-account or compliance).
- RDS: encryption must be enabled at creation — cannot encrypt existing unencrypted instances.

### In transit

- TLS 1.2 minimum everywhere. TLS 1.3 where supported.
- S3: enforce via `aws:SecureTransport` condition in bucket policy.
- ALB: use `ELBSecurityPolicy-TLS13-1-2-2021-06` or newer.
- Internal services: TLS between all components — ALB → ECS, ECS → RDS, Lambda → any endpoint.

### Key rotation

- Enable automatic rotation for CMKs (annual by default, configurable).
- Secrets Manager: configure rotation Lambda for database credentials (30-90 day cycle).
- Never embed key material in code or CloudFormation templates.

## New infrastructure workflow

```
- [ ] Define resource requirements and naming convention
- [ ] Choose region(s) and multi-AZ strategy
- [ ] Design VPC layout (CIDR, subnets, connectivity)
- [ ] Write IAM roles with least privilege
- [ ] Define encryption strategy (KMS keys, TLS)
- [ ] Write CloudFormation/CDK with DeletionPolicy on stateful resources
- [ ] Add monitoring: CloudWatch alarms, dashboards, log groups
- [ ] Add tagging: Project, Environment, Owner on all resources
- [ ] Run validation loop (below)
- [ ] Peer review the template
- [ ] Deploy to staging, verify, then production
```

## Validation loop

1. **Lint**: `cfn-lint template.yaml` or `cdk synth` — catch syntax errors, invalid property values, missing required fields.
2. **Security scan**: `cfn_nag_scan --input-path template.yaml` or `cdk-nag` — flag overly permissive policies, missing encryption, public access.
3. **Policy-as-code**: `checkov -f template.yaml` — broader compliance checks (CIS, SOC2, HIPAA benchmarks).
4. **IAM analysis**: `aws accessanalyzer validate-policy --policy-document file://policy.json` — detect unused permissions, overly broad resources, missing conditions.
5. Repeat until all tools report clean. Suppress findings only with written justification.

## Deep-dive references

- **IAM patterns**: See [patterns/iam-patterns.md](patterns/iam-patterns.md) for scoped roles, cross-account access, permission boundaries, condition keys
- **Serverless patterns**: See [patterns/serverless-patterns.md](patterns/serverless-patterns.md) for Lambda + API Gateway, SQS, EventBridge, Step Functions, Powertools
- **Networking patterns**: See [patterns/networking-patterns.md](patterns/networking-patterns.md) for VPC layouts, security group chains, VPC endpoints, transit gateway
- **Database patterns**: See [patterns/database-patterns.md](patterns/database-patterns.md) for RDS, Aurora, DynamoDB, ElastiCache best practices
- **Monitoring patterns**: See [patterns/monitoring-patterns.md](patterns/monitoring-patterns.md) for CloudWatch alarms, dashboards, CloudTrail, Config, alerting
- **Service limits**: See [service-limits-cheatsheet.md](service-limits-cheatsheet.md) for default quotas across Lambda, S3, IAM, VPC, ECS, CloudFormation

## Official references

- [AWS Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html) — pillars: operational excellence, security, reliability, performance, cost, sustainability
- [IAM Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html) — least privilege, MFA, role design, access analyzer
- [CloudFormation Best Practices](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/best-practices.html) — template organization, change sets, stack policies