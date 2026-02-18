# AWS Service Limits Cheatsheet

Default quotas for commonly used services. Check [Service Quotas console](https://console.aws.amazon.com/servicequotas/) for current account values.

## Lambda

| Limit | Default | Adjustable | Notes |
|-------|---------|------------|-------|
| Concurrent executions | 1,000 per region | Yes | Shared across all functions; request increase early for production |
| Function memory | 128 MB – 10,240 MB | No | 1 MB increments |
| Function timeout | 900 seconds (15 min) | No | |
| Deployment package (zip) | 50 MB compressed, 250 MB unzipped | No | Use container images for larger packages (up to 10 GB) |
| Container image size | 10 GB | No | |
| Environment variables | 4 KB total | No | Aggregate size of all key-value pairs |
| Layers per function | 5 | No | |
| Layer size | 250 MB unzipped (total across all layers) | No | Shared with deployment package limit |
| Burst concurrency | 500–3,000 (varies by region) | No | Initial burst, then +500/min |
| Provisioned concurrency | Equals unreserved account concurrency | Yes | |
| Ephemeral storage (/tmp) | 512 MB – 10,240 MB | No | Configurable per function |
| Function resource policy | 20 KB | No | |

## S3

| Limit | Default | Adjustable | Notes |
|-------|---------|------------|-------|
| Buckets per account | 100 | Yes | Up to 1,000 via request |
| Object size | 5 TB | No | |
| Single PUT size | 5 GB | No | Use multipart upload for objects > 100 MB |
| Multipart upload parts | 10,000 | No | Each part: 5 MB – 5 GB |
| Request rate per prefix | 5,500 GET/s, 3,500 PUT/s | No | Distribute across prefixes for higher throughput |
| Bucket policy size | 20 KB | No | |
| Object tags per object | 10 | No | |
| Lifecycle rules per bucket | 1,000 | No | |
| S3 Access Points per region | 10,000 | Yes | |

## IAM

| Limit | Default | Adjustable | Notes |
|-------|---------|------------|-------|
| Managed policies per account | 1,500 | Yes | |
| Managed policies per role/user | 10 | Yes | Up to 20 |
| Inline policy size | 2,048 chars (user), 10,240 chars (role/group) | No | Use managed policies for larger documents |
| Roles per account | 1,000 | Yes | |
| Instance profiles per account | 1,000 | Yes | |
| Role trust policy size | 2,048 chars | No | |
| Groups per account | 300 | Yes | |
| Users per account | 5,000 | Yes | |
| Access keys per user | 2 | No | |
| MFA devices per user | 8 | No | |
| Role session duration | 1–12 hours | No | Default 1 hour |
| Policy versions per managed policy | 5 | No | Delete old versions before creating new ones |

## VPC

| Limit | Default | Adjustable | Notes |
|-------|---------|------------|-------|
| VPCs per region | 5 | Yes | Commonly increased to 20+ |
| Subnets per VPC | 200 | Yes | |
| IPv4 CIDR blocks per VPC | 5 | Yes | Up to 50 |
| Security groups per VPC | 2,500 | Yes | |
| Inbound/outbound rules per SG | 60 | Yes | Inbound + outbound counted separately |
| Security groups per ENI | 5 | Yes | Up to 16 |
| Network ACLs per VPC | 200 | Yes | |
| Rules per NACL | 20 | Yes | Inbound and outbound counted separately |
| Elastic IPs per region | 5 | Yes | |
| NAT Gateways per AZ | 5 | Yes | |
| VPC endpoints per region | 50 gateway, 50 interface | Yes | |
| VPC peering connections per VPC | 50 | Yes | Up to 125 |
| Route tables per VPC | 200 | Yes | |
| Routes per route table | 50 | Yes | Up to 1,000 |
| Internet Gateways per region | 5 | Yes | Tied to VPC count |

## ECS

| Limit | Default | Adjustable | Notes |
|-------|---------|------------|-------|
| Clusters per region | 10,000 | Yes | |
| Services per cluster | 5,000 | Yes | |
| Tasks per service | 5,000 | Yes | |
| Container instances per cluster | 5,000 | Yes | EC2 launch type only |
| Task definition revisions | No limit | No | Old revisions are deregistered, not deleted |
| Containers per task definition | 10 | No | |
| Task definition size | 64 KB | No | |
| Fargate On-Demand tasks per region | 500 | Yes | Request increase well before launch |
| Fargate Spot tasks per region | 500 | Yes | |
| ECS Exec sessions | 100 per container instance | No | |
| Target groups per service | 5 | No | |

## CloudFormation

| Limit | Default | Adjustable | Notes |
|-------|---------|------------|-------|
| Stacks per region | 2,000 | Yes | |
| Resources per stack | 500 | No | Use nested stacks or break into multiple stacks |
| Outputs per stack | 200 | No | |
| Parameters per stack | 200 | No | |
| Mappings per template | 200 | No | |
| Template body size (S3) | 1 MB | No | |
| Template body size (inline) | 51,200 bytes | No | |
| Stack sets per account | 100 | Yes | |
| Stack instances per stack set | 2,000 | Yes | |
| Nested stack depth | 5 levels | No | |
| Custom resource timeout | 1 hour | No | |
| Change set quota per stack | 10 | No | Delete unused change sets |
| Exports per region | 200 | No | Prefer SSM parameters over exports |

## API Gateway

| Limit | Default | Adjustable | Notes |
|-------|---------|------------|-------|
| HTTP APIs per region | 100 | Yes | |
| REST APIs per region | 600 | Yes | |
| Routes per HTTP API | 300 | Yes | |
| Resources per REST API | 300 | No | |
| Stages per API | 10 | Yes | |
| Authorizers per API | 10 | Yes | |
| Throttle (account level) | 10,000 req/s, 5,000 burst | Yes | Shared across all APIs in the region |
| Throttle per-route (HTTP API) | 10,000 req/s | Yes | |
| Payload size | 10 MB | No | Both HTTP API and REST API |
| WebSocket message size | 128 KB (send), 32 KB (receive) | No | |
| Custom domain names per region | 120 | Yes | |
| Integration timeout | 30 seconds (HTTP API), 29 seconds (REST API) | No | |
| Cache size (REST API only) | 0.5 – 237 GB | No | Not available on HTTP API |

## SQS

| Limit | Default | Adjustable | Notes |
|-------|---------|------------|-------|
| Queues per region | 1,000 (standard), 1,000 (FIFO) | Yes | |
| Message size | 256 KB | No | Use S3 for larger payloads via Extended Client Library |
| Message retention | 1 minute – 14 days | No | Default 4 days |
| Visibility timeout | 0 seconds – 12 hours | No | Set to ≥6x Lambda timeout |
| Inflight messages | 120,000 (standard), 20,000 (FIFO) | No | Messages received but not yet deleted |
| Batch size | 10 messages | No | Per `SendMessageBatch` / `ReceiveMessage` |
| FIFO throughput | 300 msg/s (without batching), 3,000 msg/s (with batching) | No | Per message group for ordering |
| FIFO high throughput | 30,000 msg/s | No | Requires `DeduplicationScope: messageGroup` |
| Long polling wait time | 0 – 20 seconds | No | Use 20s to reduce empty responses and cost |
| Delay queue | 0 – 15 minutes | No | Per-queue or per-message |

## SNS

| Limit | Default | Adjustable | Notes |
|-------|---------|------------|-------|
| Topics per region | 100,000 | Yes | |
| Subscriptions per topic | 12,500,000 | Yes | |
| Message size | 256 KB | No | Use S3 for larger payloads |
| Publish rate | Varies by region | Yes | ~30,000/s for standard topics |
| FIFO topic throughput | 300 publishes/s | No | Per message group |
| Filter policies per topic | 200 | Yes | One per subscription |
| Filter policy size | 150 KB | No | |
| SMS spending limit | $1.00/month | Yes | Set via account attributes |
| Mobile push platform applications | 1,000 | No | |
| Delivery retries (HTTP/S) | 100,185 over 23 days | No | Exponential backoff built-in |
