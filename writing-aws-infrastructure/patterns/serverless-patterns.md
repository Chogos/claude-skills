# Serverless Patterns

## Contents

- [Lambda + API Gateway HTTP API](#lambda--api-gateway-http-api)
- [Lambda + SQS with DLQ](#lambda--sqs-with-dlq)
- [Lambda + EventBridge rules](#lambda--eventbridge-rules)
- [Step Functions orchestration](#step-functions-orchestration)
- [Lambda Powertools](#lambda-powertools)
- [Cold start mitigation](#cold-start-mitigation)

## Lambda + API Gateway HTTP API

HTTP APIs (v2) are cheaper and faster than REST APIs (v1). Use REST APIs only when you need request validation, WAF integration, or usage plans.

```typescript
import { Stack, StackProps, Duration, CfnOutput } from "aws-cdk-lib";
import { Construct } from "constructs";
import { HttpApi, HttpMethod, CorsHttpMethod } from "aws-cdk-lib/aws-apigatewayv2";
import { HttpLambdaIntegration } from "aws-cdk-lib/aws-apigatewayv2-integrations";
import { NodejsFunction } from "aws-cdk-lib/aws-lambda-nodejs";
import { Runtime, Architecture, Tracing } from "aws-cdk-lib/aws-lambda";
import { RetentionDays } from "aws-cdk-lib/aws-logs";

export class ApiStack extends Stack {
  constructor(scope: Construct, id: string, props: StackProps) {
    super(scope, id, props);

    const getOrders = new NodejsFunction(this, "GetOrders", {
      entry: "src/handlers/get-orders.ts",
      handler: "handler",
      runtime: Runtime.NODEJS_22_X,
      architecture: Architecture.ARM_64,
      memorySize: 512,
      timeout: Duration.seconds(10),
      tracing: Tracing.ACTIVE,
      logRetention: RetentionDays.TWO_WEEKS,
      bundling: { minify: true, sourceMap: true },
    });

    const createOrder = new NodejsFunction(this, "CreateOrder", {
      entry: "src/handlers/create-order.ts",
      handler: "handler",
      runtime: Runtime.NODEJS_22_X,
      architecture: Architecture.ARM_64,
      memorySize: 512,
      timeout: Duration.seconds(10),
      tracing: Tracing.ACTIVE,
      logRetention: RetentionDays.TWO_WEEKS,
      bundling: { minify: true, sourceMap: true },
    });

    const httpApi = new HttpApi(this, "OrdersApi", {
      apiName: "orders-api",
      corsPreflight: {
        allowOrigins: ["https://app.example.com"],
        allowMethods: [CorsHttpMethod.GET, CorsHttpMethod.POST],
        allowHeaders: ["Authorization", "Content-Type"],
        maxAge: Duration.hours(1),
      },
    });

    httpApi.addRoutes({
      path: "/orders",
      methods: [HttpMethod.GET],
      integration: new HttpLambdaIntegration("GetOrdersIntegration", getOrders),
    });

    httpApi.addRoutes({
      path: "/orders",
      methods: [HttpMethod.POST],
      integration: new HttpLambdaIntegration("CreateOrderIntegration", createOrder),
    });

    new CfnOutput(this, "ApiUrl", { value: httpApi.apiEndpoint });
  }
}
```

## Lambda + SQS with DLQ

Always configure a dead-letter queue. Failed messages must go somewhere observable.

```typescript
import { Stack, StackProps, Duration } from "aws-cdk-lib";
import { Construct } from "constructs";
import { Queue, DeadLetterQueue } from "aws-cdk-lib/aws-sqs";
import { SqsEventSource } from "aws-cdk-lib/aws-lambda-event-sources";
import { NodejsFunction } from "aws-cdk-lib/aws-lambda-nodejs";
import { Runtime, Architecture, Tracing } from "aws-cdk-lib/aws-lambda";
import { Alarm, ComparisonOperator } from "aws-cdk-lib/aws-cloudwatch";
import { SnsAction } from "aws-cdk-lib/aws-cloudwatch-actions";

export class QueueProcessorStack extends Stack {
  constructor(scope: Construct, id: string, props: StackProps) {
    super(scope, id, props);

    const dlq = new Queue(this, "OrderDLQ", {
      queueName: "order-processing-dlq",
      retentionPeriod: Duration.days(14),
    });

    const orderQueue = new Queue(this, "OrderQueue", {
      queueName: "order-processing",
      visibilityTimeout: Duration.seconds(180), // 6x Lambda timeout
      retentionPeriod: Duration.days(7),
      deadLetterQueue: {
        queue: dlq,
        maxReceiveCount: 3,
      },
    });

    const processor = new NodejsFunction(this, "OrderProcessor", {
      entry: "src/handlers/process-order.ts",
      handler: "handler",
      runtime: Runtime.NODEJS_22_X,
      architecture: Architecture.ARM_64,
      memorySize: 512,
      timeout: Duration.seconds(30),
      tracing: Tracing.ACTIVE,
      reservedConcurrentExecutions: 10,
    });

    processor.addEventSource(new SqsEventSource(orderQueue, {
      batchSize: 10,
      maxBatchingWindow: Duration.seconds(5),
      reportBatchItemFailures: true, // partial batch failure support
    }));

    // Alarm on DLQ depth — messages in DLQ need investigation
    new Alarm(this, "DLQAlarm", {
      metric: dlq.metricApproximateNumberOfMessagesVisible(),
      threshold: 1,
      evaluationPeriods: 1,
      comparisonOperator: ComparisonOperator.GREATER_THAN_OR_EQUAL_TO_THRESHOLD,
      alarmDescription: "Messages in order processing DLQ",
    });
  }
}
```

- Set `visibilityTimeout` to at least 6x the Lambda timeout.
- Enable `reportBatchItemFailures` — without it, one failed message retries the entire batch.
- `reservedConcurrentExecutions` throttles the consumer to protect downstream systems.

## Lambda + EventBridge rules

Decouple producers from consumers. EventBridge routes events by pattern matching.

```typescript
import { Stack, StackProps, Duration } from "aws-cdk-lib";
import { Construct } from "constructs";
import { EventBus, Rule } from "aws-cdk-lib/aws-events";
import { LambdaFunction } from "aws-cdk-lib/aws-events-targets";
import { NodejsFunction } from "aws-cdk-lib/aws-lambda-nodejs";
import { Runtime, Architecture } from "aws-cdk-lib/aws-lambda";
import { Queue } from "aws-cdk-lib/aws-sqs";

export class EventProcessingStack extends Stack {
  constructor(scope: Construct, id: string, props: StackProps) {
    super(scope, id, props);

    const bus = new EventBus(this, "OrderEventBus", {
      eventBusName: "orders",
    });

    const dlq = new Queue(this, "EventDLQ");

    const notifyShipping = new NodejsFunction(this, "NotifyShipping", {
      entry: "src/handlers/notify-shipping.ts",
      runtime: Runtime.NODEJS_22_X,
      architecture: Architecture.ARM_64,
      memorySize: 256,
      timeout: Duration.seconds(10),
    });

    new Rule(this, "OrderCompletedRule", {
      eventBus: bus,
      eventPattern: {
        source: ["com.acme.orders"],
        detailType: ["OrderCompleted"],
        detail: {
          status: ["PAID"],
        },
      },
      targets: [
        new LambdaFunction(notifyShipping, {
          deadLetterQueue: dlq,
          retryAttempts: 2,
          maxEventAge: Duration.hours(1),
        }),
      ],
    });
  }
}
```

- Use a custom event bus per domain — default bus mixes with AWS service events.
- Always set `retryAttempts` and `deadLetterQueue` on targets.
- Keep event detail-type stable — consumers depend on it for routing.

## Step Functions orchestration

Use Standard workflows for long-running orchestrations (up to 1 year). Use Express workflows for high-volume, short-duration flows (up to 5 minutes).

```typescript
import { Stack, StackProps, Duration } from "aws-cdk-lib";
import { Construct } from "constructs";
import {
  StateMachine, Wait, WaitTime, Choice, Condition,
  Succeed, Fail, TaskInput, LogLevel, StateMachineType
} from "aws-cdk-lib/aws-stepfunctions";
import { LambdaInvoke } from "aws-cdk-lib/aws-stepfunctions-tasks";
import { NodejsFunction } from "aws-cdk-lib/aws-lambda-nodejs";
import { Runtime, Architecture } from "aws-cdk-lib/aws-lambda";
import { LogGroup, RetentionDays } from "aws-cdk-lib/aws-logs";

export class OrderWorkflowStack extends Stack {
  constructor(scope: Construct, id: string, props: StackProps) {
    super(scope, id, props);

    const validateOrder = this.createFunction("ValidateOrder", "validate-order");
    const processPayment = this.createFunction("ProcessPayment", "process-payment");
    const shipOrder = this.createFunction("ShipOrder", "ship-order");

    const validateStep = new LambdaInvoke(this, "ValidateStep", {
      lambdaFunction: validateOrder,
      resultSelector: { "valid.$": "$.Payload.valid" },
      retryOnServiceExceptions: true,
    });

    const paymentStep = new LambdaInvoke(this, "PaymentStep", {
      lambdaFunction: processPayment,
      retryOnServiceExceptions: true,
    }).addRetry({
      errors: ["PaymentProviderTimeout"],
      interval: Duration.seconds(5),
      maxAttempts: 3,
      backoffRate: 2,
    });

    const shipStep = new LambdaInvoke(this, "ShipStep", {
      lambdaFunction: shipOrder,
      retryOnServiceExceptions: true,
    });

    const definition = validateStep.next(
      new Choice(this, "IsValid?")
        .when(Condition.booleanEquals("$.valid", true),
          paymentStep.next(shipStep).next(new Succeed(this, "OrderComplete")))
        .otherwise(new Fail(this, "ValidationFailed", {
          cause: "Order validation failed",
        }))
    );

    const logGroup = new LogGroup(this, "WorkflowLogs", {
      retention: RetentionDays.TWO_WEEKS,
    });

    new StateMachine(this, "OrderWorkflow", {
      definitionBody: DefinitionBody.fromChainable(definition),
      stateMachineType: StateMachineType.STANDARD,
      timeout: Duration.minutes(30),
      tracingEnabled: true,
      logs: {
        destination: logGroup,
        level: LogLevel.ERROR,
        includeExecutionData: true,
      },
    });
  }

  private createFunction(id: string, entry: string): NodejsFunction {
    return new NodejsFunction(this, id, {
      entry: `src/handlers/${entry}.ts`,
      runtime: Runtime.NODEJS_22_X,
      architecture: Architecture.ARM_64,
      memorySize: 256,
      timeout: Duration.seconds(30),
    });
  }
}
```

| Feature | Standard | Express |
|---------|----------|---------|
| Max duration | 1 year | 5 minutes |
| Pricing | Per state transition | Per execution + duration |
| Execution semantics | Exactly-once | At-least-once (async) or at-most-once (sync) |
| History | Full execution history | CloudWatch Logs only |
| Use case | Order workflows, approvals | Data transforms, IoT ingestion |

## Lambda Powertools

Use Lambda Powertools for structured logging, distributed tracing, and custom metrics. Available for Python, TypeScript, Java, .NET.

```typescript
// src/handlers/process-order.ts
import { Logger } from "@aws-lambda-powertools/logger";
import { Tracer } from "@aws-lambda-powertools/tracer";
import { Metrics, MetricUnit } from "@aws-lambda-powertools/metrics";
import middy from "@middy/core";
import { injectLambdaContext } from "@aws-lambda-powertools/logger/middleware";
import { captureLambdaHandler } from "@aws-lambda-powertools/tracer/middleware";
import { logMetrics } from "@aws-lambda-powertools/metrics/middleware";

const logger = new Logger({ serviceName: "order-service" });
const tracer = new Tracer({ serviceName: "order-service" });
const metrics = new Metrics({ serviceName: "order-service", namespace: "Acme" });

const lambdaHandler = async (event: SQSEvent): Promise<SQSBatchResponse> => {
  const failures: SQSBatchItemFailure[] = [];

  for (const record of event.Records) {
    try {
      const order = JSON.parse(record.body);
      logger.appendKeys({ orderId: order.id });
      logger.info("Processing order");

      // Business logic here
      await processOrder(order);

      metrics.addMetric("OrderProcessed", MetricUnit.Count, 1);
    } catch (error) {
      logger.error("Failed to process order", { error });
      metrics.addMetric("OrderFailed", MetricUnit.Count, 1);
      failures.push({ itemIdentifier: record.messageId });
    }
  }

  return { batchItemFailures: failures };
};

export const handler = middy(lambdaHandler)
  .use(injectLambdaContext(logger, { clearState: true }))
  .use(captureLambdaHandler(tracer))
  .use(logMetrics(metrics));
```

Set env vars in the Lambda configuration:

```typescript
environment: {
  POWERTOOLS_SERVICE_NAME: "order-service",
  POWERTOOLS_METRICS_NAMESPACE: "Acme",
  POWERTOOLS_LOG_LEVEL: "INFO",
  POWERTOOLS_LOGGER_SAMPLE_RATE: "0.1", // sample 10% of debug logs
}
```

## Cold start mitigation

Cold starts add 100ms–2s depending on runtime, package size, and VPC attachment.

**Strategies (ordered by impact):**

1. **ARM64 (`arm64`)** — Graviton cold starts are ~15% faster than x86.
2. **Minimize package size** — tree-shake, minify, exclude dev deps. Use `bundling: { minify: true }` in CDK.
3. **Avoid VPC unless required** — VPC-attached Lambdas used to have 10s+ cold starts. Hyperplane (since 2019) reduced this to ~1s, but it still adds latency.
4. **SnapStart (Java only)** — snapshots initialized execution environment. Reduces cold start from ~5s to ~200ms.
5. **Provisioned concurrency** — pre-warmed execution environments. Eliminates cold starts entirely at a fixed cost.

```typescript
import { Alias } from "aws-cdk-lib/aws-lambda";

const fn = new NodejsFunction(this, "CriticalApi", {
  entry: "src/handlers/critical-api.ts",
  runtime: Runtime.NODEJS_22_X,
  architecture: Architecture.ARM_64,
  memorySize: 1024,
  currentVersionOptions: {
    removalPolicy: RemovalPolicy.RETAIN,
  },
});

const alias = new Alias(this, "ProdAlias", {
  aliasName: "prod",
  version: fn.currentVersion,
  provisionedConcurrentExecutions: 5,
});
```

- Provisioned concurrency costs ~$0.015/GB-hour — use only for latency-sensitive paths.
- Scale provisioned concurrency with Application Auto Scaling based on utilization.
- Keep initialization code (SDK clients, DB connections) outside the handler — reused across invocations.