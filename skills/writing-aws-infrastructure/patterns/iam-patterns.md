# IAM Patterns

## Contents

- [Scoped Lambda execution role (CloudFormation)](#scoped-lambda-execution-role-cloudformation)
- [Scoped Lambda execution role (CDK)](#scoped-lambda-execution-role-cdk)
- [Cross-account assume-role chain](#cross-account-assume-role-chain)
- [Permission boundaries for developer sandboxes](#permission-boundaries-for-developer-sandboxes)
- [Service-linked roles](#service-linked-roles)
- [Condition keys reference](#condition-keys-reference)

## Scoped Lambda execution role (CloudFormation)

The role grants only the specific DynamoDB actions on one table and CloudWatch Logs access for its own log group. The trust policy pins to the exact function ARN.

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Parameters:
  Environment:
    Type: String
    AllowedValues: [dev, staging, prod]

Resources:
  ProcessOrderRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: !Sub "process-order-${Environment}"
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
            Condition:
              ArnLike:
                aws:SourceArn: !Sub "arn:aws:lambda:${AWS::Region}:${AWS::AccountId}:function:process-order-${Environment}"
      Policies:
        - PolicyName: DynamoDBAccess
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - Sid: ReadWriteOrdersTable
                Effect: Allow
                Action:
                  - dynamodb:GetItem
                  - dynamodb:PutItem
                  - dynamodb:UpdateItem
                  - dynamodb:Query
                Resource:
                  - !GetAtt OrdersTable.Arn
                  - !Sub "${OrdersTable.Arn}/index/*"
        - PolicyName: CloudWatchLogs
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - Sid: WriteLogs
                Effect: Allow
                Action:
                  - logs:CreateLogStream
                  - logs:PutLogEvents
                Resource: !Sub "arn:aws:logs:${AWS::Region}:${AWS::AccountId}:log-group:/aws/lambda/process-order-${Environment}:*"
```

## Scoped Lambda execution role (CDK)

CDK `grant*` methods generate scoped policies automatically. Use them over hand-written policy statements.

```typescript
import { Stack, StackProps } from "aws-cdk-lib";
import { Construct } from "constructs";
import { Table } from "aws-cdk-lib/aws-dynamodb";
import { NodejsFunction } from "aws-cdk-lib/aws-lambda-nodejs";
import { Runtime, Architecture } from "aws-cdk-lib/aws-lambda";
import { Queue } from "aws-cdk-lib/aws-sqs";
import { Secret } from "aws-cdk-lib/aws-secretsmanager";

export class OrderServiceStack extends Stack {
  constructor(scope: Construct, id: string, props: StackProps) {
    super(scope, id, props);

    const ordersTable = new Table(this, "OrdersTable", { /* ... */ });
    const dlq = new Queue(this, "OrderDLQ");
    const orderQueue = new Queue(this, "OrderQueue", {
      deadLetterQueue: { queue: dlq, maxReceiveCount: 3 },
    });
    const dbSecret = Secret.fromSecretNameV2(this, "DbSecret", "prod/db");

    const processOrder = new NodejsFunction(this, "ProcessOrder", {
      entry: "src/handlers/process-order.ts",
      runtime: Runtime.NODEJS_22_X,
      architecture: Architecture.ARM_64,
    });

    // Each grant generates a scoped IAM statement
    ordersTable.grantReadWriteData(processOrder);
    orderQueue.grantConsumeMessages(processOrder);
    dbSecret.grantRead(processOrder);
  }
}
```

## Cross-account assume-role chain

Account A (111111111111) assumes a role in Account B (222222222222) to read an S3 bucket.

**Account B — trust policy on the target role:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowAccountAAssume",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::111111111111:role/data-pipeline-role"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "acme-cross-account-2024",
          "aws:PrincipalOrgID": "o-abc123"
        }
      }
    }
  ]
}
```

**Account B — permissions on the target role:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadSharedBucket",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::shared-data-bucket",
        "arn:aws:s3:::shared-data-bucket/exports/*"
      ]
    }
  ]
}
```

**Account A — permission to assume:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AssumeAccountBRole",
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::222222222222:role/shared-data-reader"
    }
  ]
}
```

Both sides must grant access — Account A must be allowed to call `sts:AssumeRole`, and Account B's trust policy must allow Account A's principal.

## Permission boundaries for developer sandboxes

Developers can create IAM roles but cannot exceed their own boundary. Prevents privilege escalation.

**Boundary policy (managed policy):**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowedServices",
      "Effect": "Allow",
      "Action": [
        "s3:*",
        "dynamodb:*",
        "lambda:*",
        "logs:*",
        "sqs:*",
        "sns:*"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": ["eu-west-1", "eu-central-1"]
        }
      }
    },
    {
      "Sid": "DenyBoundaryDeletion",
      "Effect": "Deny",
      "Action": [
        "iam:DeleteRolePermissionsBoundary",
        "iam:DeleteUserPermissionsBoundary"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DenyBoundaryPolicyModification",
      "Effect": "Deny",
      "Action": [
        "iam:CreatePolicyVersion",
        "iam:DeletePolicy",
        "iam:DeletePolicyVersion",
        "iam:SetDefaultPolicyVersion"
      ],
      "Resource": "arn:aws:iam::*:policy/developer-boundary"
    }
  ]
}
```

**Developer role — limited IAM creation with forced boundary:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "CreateRolesWithBoundary",
      "Effect": "Allow",
      "Action": [
        "iam:CreateRole",
        "iam:AttachRolePolicy",
        "iam:PutRolePolicy"
      ],
      "Resource": "arn:aws:iam::*:role/sandbox-*",
      "Condition": {
        "StringEquals": {
          "iam:PermissionsBoundary": "arn:aws:iam::*:policy/developer-boundary"
        }
      }
    },
    {
      "Sid": "DenyCreateWithoutBoundary",
      "Effect": "Deny",
      "Action": [
        "iam:CreateRole",
        "iam:CreateUser"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "iam:PermissionsBoundary": "arn:aws:iam::*:policy/developer-boundary"
        }
      }
    }
  ]
}
```

## Service-linked roles

AWS-managed roles for services that need to act on your behalf. You cannot modify their permissions.

| Service | Role Name | Created By |
|---------|-----------|------------|
| ECS | `AWSServiceRoleForECS` | First ECS service creation |
| Auto Scaling | `AWSServiceRoleForAutoScaling` | First ASG creation |
| ElastiCache | `AWSServiceRoleForElastiCache` | First cluster creation |
| RDS | `AWSServiceRoleForRDS` | First DB instance creation |
| Organizations | `AWSServiceRoleForOrganizations` | Enabling Organizations |

Create explicitly in CloudFormation when you need the role before the service uses it:

```yaml
EcsServiceLinkedRole:
  Type: AWS::IAM::ServiceLinkedRole
  Properties:
    AWSServiceName: ecs.amazonaws.com
    Description: "ECS service-linked role"
```

## Condition keys reference

| Condition Key | Use Case | Example Value |
|---------------|----------|---------------|
| `aws:SourceArn` | Restrict which resource can assume a role | `arn:aws:lambda:eu-west-1:111111111111:function:my-func` |
| `aws:PrincipalOrgID` | Allow only principals from your AWS Organization | `o-abc123` |
| `aws:SourceVpc` | Restrict API calls to originate from a specific VPC | `vpc-0abc123` |
| `aws:SourceIp` | Restrict to specific IP ranges (use for human users, not services) | `203.0.113.0/24` |
| `aws:RequestedRegion` | Limit resource creation to approved regions | `eu-west-1` |
| `aws:PrincipalTag/Department` | ABAC — match principal's tag against policy condition | `engineering` |
| `s3:prefix` | Scope S3 ListBucket to a key prefix | `uploads/` |
| `dynamodb:LeadingKeys` | Restrict DynamoDB queries to specific partition keys | `${aws:PrincipalTag/UserId}` |
| `sts:ExternalId` | Prevent confused-deputy in cross-account access | `acme-cross-account-2024` |
| `ec2:ResourceTag/Environment` | Restrict EC2 actions to tagged resources | `prod` |

**ABAC example** — allow access only to resources tagged with the caller's department:

```json
{
  "Sid": "AbacDepartmentScope",
  "Effect": "Allow",
  "Action": ["s3:GetObject", "s3:PutObject"],
  "Resource": "arn:aws:s3:::shared-data/*",
  "Condition": {
    "StringEquals": {
      "s3:ExistingObjectTag/Department": "${aws:PrincipalTag/Department}"
    }
  }
}