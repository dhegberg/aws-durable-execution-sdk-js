# Lambda Function Structure

This chapter covers how a consumer structures a Lambda function to use the Durable Functions SDK — from imports through handler definition to export. It also covers the type system, IAM permissions, the qualified ARN requirement for invocation, and infrastructure-as-code deployment options.

## Complete File Structure

A durable Lambda function follows a straightforward pattern: import the SDK, define a handler function, wrap it with `withDurableExecution`, and export the result.

```typescript
// 1. Import the SDK
import {
  withDurableExecution,
  DurableContext,
} from "@aws/durable-execution-sdk-js";

// 2. Define the handler function
const durableHandler = async (event: { orderId: string }, context: DurableContext) => {
  const order = await context.step("fetch-order", async () =>
    fetchOrder(event.orderId)
  );

  await context.wait("processing-delay", { seconds: 30 });

  const result = await context.step("process-order", async () =>
    processOrder(order)
  );

  return { status: "completed", result };
};

// 3. Wrap and export
export const handler = withDurableExecution(durableHandler);
```

The exported `handler` is what Lambda invokes. The SDK handles all durable execution mechanics — checkpoint management, replay, state loading — transparently.

## Handler Types

The SDK exposes three key types for structuring a durable Lambda function.

### `DurableExecutionHandler<TEvent, TResult, TLogger>`

This is the type for the function you write — the business logic handler that receives the event and a `DurableContext`.

```typescript
export type DurableExecutionHandler<
  TEvent = any,
  TResult = any,
  TLogger extends DurableLogger = DurableLogger,
> = (event: TEvent, context: DurableContext<TLogger>) => Promise<TResult>;
```

**Type parameters:**

| Parameter | Default | Description |
|-----------|---------|-------------|
| `TEvent` | `any` | The type of the input event payload. This is the parsed JSON data from the invocation, not the raw `DurableExecutionInvocationInput` envelope — the SDK extracts and deserializes the event for you. |
| `TResult` | `any` | The return type of the handler. The SDK serializes this as the execution result on success. |
| `TLogger` | `DurableLogger` | The logger type available via `context.logger`. Must extend `DurableLogger`. Use this to add custom logging methods beyond the standard `log`, `info`, `warn`, `error`, `debug`. |

**Source:** [`packages/aws-durable-execution-sdk-js/src/types/durable-execution.ts`](../packages/aws-durable-execution-sdk-js/src/types/durable-execution.ts)

### `DurableLambdaHandler`

This is the type returned by `withDurableExecution` — the actual Lambda handler that gets deployed and invoked by the Durable Execution service.

```typescript
export type DurableLambdaHandler = (
  event: DurableExecutionInvocationInput,
  context: Context,
) => Promise<DurableExecutionInvocationOutput>;
```

Unlike `DurableExecutionHandler`, this handler receives the raw `DurableExecutionInvocationInput` (containing execution metadata, checkpoint tokens, and operation history) and the standard Lambda `Context`. You never implement this type directly — `withDurableExecution` creates it for you.

**Source:** [`packages/aws-durable-execution-sdk-js/src/types/durable-execution.ts`](../packages/aws-durable-execution-sdk-js/src/types/durable-execution.ts)

### Typing Patterns

There are three ways to apply types to your handler:

**Explicit type parameters:**

```typescript
import {
  DurableExecutionHandler,
  DurableLambdaHandler,
  withDurableExecution,
} from "@aws/durable-execution-sdk-js";

interface OrderEvent {
  orderId: string;
  action: "create" | "update";
}

interface OrderResult {
  success: boolean;
  processedAt: string;
}

const durableHandler: DurableExecutionHandler<OrderEvent, OrderResult> =
  async (event, context) => {
    // event is typed as OrderEvent
    // return type must match OrderResult
    const data = await context.step("process", async () => ({
      action: event.action,
      timestamp: new Date().toISOString(),
    }));

    return { success: true, processedAt: data.timestamp };
  };

export const handler: DurableLambdaHandler = withDurableExecution(durableHandler);
```

**Inline with `satisfies`:**

```typescript
import {
  DurableLambdaHandler,
  withDurableExecution,
} from "@aws/durable-execution-sdk-js";

export const handler = withDurableExecution(async (event, context) => {
  const result = await context.step(async () => ({
    processed: true,
    completedAt: new Date().toISOString(),
  }));
  return result;
}) satisfies DurableLambdaHandler;
```

**Default types (no explicit generics):**

```typescript
const durableHandler: DurableExecutionHandler = async (event, context) => {
  // event is `any`, return type is `any`
  return await context.step(async () => ({ message: "done", data: event }));
};

export const handler = withDurableExecution(durableHandler);
```

## `withDurableExecution` Transformation

`withDurableExecution` is the bridge between your handler and the Lambda runtime. It takes a `DurableExecutionHandler` and returns a `DurableLambdaHandler`:

```typescript
export const withDurableExecution = <
  TEvent = any,
  TResult = any,
  TLogger extends DurableLogger = DurableLogger,
>(
  handler: DurableExecutionHandler<TEvent, TResult, TLogger>,
  config?: DurableExecutionConfig,
): DurableLambdaHandler => { ... };
```

When the returned handler is invoked by Lambda, it:

1. Validates the `DurableExecutionInvocationInput` event
2. Initializes the execution context (loads operation history, sets up checkpoint management)
3. Runs your handler function, providing the extracted event payload and a `DurableContext`
4. Returns a `DurableExecutionInvocationOutput` with status `SUCCEEDED`, `FAILED`, or `PENDING`

The generic type parameters flow through from your handler — if you define `DurableExecutionHandler<OrderEvent, OrderResult>`, TypeScript infers the types automatically when you call `withDurableExecution(durableHandler)`.

**Source:** [`packages/aws-durable-execution-sdk-js/src/with-durable-execution.ts`](../packages/aws-durable-execution-sdk-js/src/with-durable-execution.ts)

## `DurableExecutionConfig`

The optional second argument to `withDurableExecution` allows you to customize the SDK's runtime behavior:

```typescript
export interface DurableExecutionConfig {
  /**
   * Optional custom AWS Lambda client for durable execution operations.
   */
  client?: LambdaClient;
}
```

The `client` property accepts a custom `LambdaClient` instance from `@aws-sdk/client-lambda`. This is useful for:

- Custom AWS configurations (region, credentials, endpoints)
- Testing with mocked Lambda clients
- Advanced networking (VPC endpoints, proxies)
- Custom retry and timeout settings

```typescript
import { LambdaClient } from "@aws-sdk/client-lambda";

const customClient = new LambdaClient({
  region: "us-west-2",
  maxAttempts: 5,
  retryMode: "adaptive",
});

export const handler = withDurableExecution(durableHandler, {
  client: customClient,
});
```

If omitted, the SDK creates a default `LambdaClient` using the standard AWS SDK configuration chain (environment variables, IAM roles, instance metadata).

**Source:** [`packages/aws-durable-execution-sdk-js/src/types/durable-execution.ts`](../packages/aws-durable-execution-sdk-js/src/types/durable-execution.ts)

## IAM Permissions

Durable functions require specific IAM permissions to checkpoint state and retrieve execution history.

### Base Policy

The [`AWSLambdaBasicDurableExecutionRolePolicy`](https://docs.aws.amazon.com/aws-managed-policy/latest/reference/AWSLambdaBasicDurableExecutionRolePolicy.html) managed policy provides the minimum permissions:

| Permission | Purpose |
|-----------|---------|
| `logs:CreateLogGroup` | Create CloudWatch log groups |
| `logs:CreateLogStream` | Create CloudWatch log streams |
| `logs:PutLogEvents` | Write log entries |
| `lambda:CheckpointDurableExecutions` | Persist execution state (step results, wait registrations, etc.) |
| `lambda:GetDurableExecutionState` | Retrieve execution state during replay |

This policy is sufficient for durable functions that only use steps, waits, and child contexts.

### Additional Permissions for Durable Invokes

If your function uses `context.invoke()` to call other durable functions, the execution role also needs:

```json
{
  "Effect": "Allow",
  "Action": "lambda:InvokeFunction",
  "Resource": "arn:aws:lambda:us-east-1:123456789012:function:target-function:*"
}
```

Scope the resource ARN to the specific functions being invoked. The `:*` suffix covers all versions and aliases.

### Additional Permissions for Callbacks

If external systems send callbacks to your durable function, those systems need:

| Permission | Purpose |
|-----------|---------|
| `lambda:SendDurableExecutionCallbackSuccess` | Send a successful callback result |
| `lambda:SendDurableExecutionCallbackFailure` | Send a callback failure |

These permissions are granted to the external system's role, not the durable function's role.

## Qualified ARN Requirement

Durable functions require qualified identifiers for invocation. You must use a version number, alias, or `$LATEST` — unqualified function names and ARNs are not supported.

### Valid Invocations

```
# Full ARN with version number
arn:aws:lambda:us-east-1:123456789012:function:my-function:1

# Full ARN with alias
arn:aws:lambda:us-east-1:123456789012:function:my-function:prod

# Full ARN with $LATEST
arn:aws:lambda:us-east-1:123456789012:function:my-function:$LATEST

# Function name with version or alias
my-function:1
my-function:prod
```

### Invalid Invocations

```
# Unqualified ARN — NOT ALLOWED
arn:aws:lambda:us-east-1:123456789012:function:my-function

# Unqualified function name — NOT ALLOWED
my-function
```

### Why Qualified ARNs?

The Lambda Durable Functions service uses the qualifier to associate execution state with a specific function version. This ensures that replay uses the same code version that started the execution, preventing behavior mismatches if the function code changes between invocations.

Use numbered versions or aliases for production deployments. Use `$LATEST` only for development and prototyping.

## Infrastructure as Code

Deploy durable functions using CloudFormation, CDK, or SAM. All approaches require:

1. Enabling durable execution on the function via `DurableConfig`
2. Granting checkpoint permissions to the execution role
3. Publishing a version or creating an alias (qualified ARNs are required for invocation)

### AWS CloudFormation

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Resources:
  DurableFunctionRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicDurableExecutionRolePolicy

  DurableFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: myDurableFunction
      Runtime: nodejs22.x
      Handler: index.handler
      Role: !GetAtt DurableFunctionRole.Arn
      Code:
        ZipFile: |
          // Your durable function code
      DurableConfig:
        ExecutionTimeout: 3600
        RetentionPeriodInDays: 7

  DurableFunctionVersion:
    Type: AWS::Lambda::Version
    Properties:
      FunctionName: !Ref DurableFunction

  DurableFunctionAlias:
    Type: AWS::Lambda::Alias
    Properties:
      FunctionName: !Ref DurableFunction
      FunctionVersion: !GetAtt DurableFunctionVersion.Version
      Name: prod
```

Key points:
- `DurableConfig.ExecutionTimeout` sets the maximum execution duration in seconds (here, 1 hour)
- `DurableConfig.RetentionPeriodInDays` controls how long execution state is retained
- A `Version` and `Alias` are created to satisfy the qualified ARN requirement

### AWS CDK (TypeScript)

```typescript
import * as cdk from "aws-cdk-lib";
import * as lambda from "aws-cdk-lib/aws-lambda";

const durableFunction = new lambda.Function(this, "DurableFunction", {
  runtime: lambda.Runtime.NODEJS_22_X,
  handler: "index.handler",
  code: lambda.Code.fromAsset("lambda"),
  durableConfig: {
    executionTimeout: cdk.Duration.hours(1),
    retentionPeriod: cdk.Duration.days(7),
  },
});

// CDK automatically adds checkpoint permissions when durableConfig is set

// Create version and alias for qualified invocation
const version = durableFunction.currentVersion;
const alias = new lambda.Alias(this, "ProdAlias", {
  aliasName: "prod",
  version: version,
});
```

When you set `durableConfig` on a CDK `Function`, the construct automatically attaches the required checkpoint permissions — no need to add the managed policy manually.

### AWS SAM

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Transform: AWS::Serverless-2016-10-31

Resources:
  DurableFunction:
    Type: AWS::Serverless::Function
    Properties:
      FunctionName: myDurableFunction
      Runtime: nodejs22.x
      Handler: index.handler
      CodeUri: ./src
      DurableConfig:
        ExecutionTimeout: 3600
        RetentionPeriodInDays: 7
      Policies:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicDurableExecutionRolePolicy
      AutoPublishAlias: prod
```

SAM's `AutoPublishAlias` automatically creates a new version on each deployment and points the `prod` alias to it — handling the qualified ARN requirement with no extra resources.

## Key Takeaways

- A durable Lambda function is structured as: import SDK → define `DurableExecutionHandler` → wrap with `withDurableExecution` → export the result.
- `DurableExecutionHandler<TEvent, TResult, TLogger>` is the type you implement. `DurableLambdaHandler` is the type Lambda invokes. `withDurableExecution` bridges the two.
- `DurableExecutionConfig` optionally accepts a custom `LambdaClient` for advanced scenarios.
- The `AWSLambdaBasicDurableExecutionRolePolicy` managed policy provides base permissions. Add `lambda:InvokeFunction` for durable invokes and callback permissions for external systems.
- Durable functions must be invoked with qualified ARNs (version, alias, or `$LATEST`).
- CloudFormation, CDK, and SAM all support `DurableConfig` for enabling durable execution. Always publish a version or alias for production deployments.

---

[← Previous: Overview and Conceptual Foundation](./01-overview.md) | [Next: Consumer Interfaces →](./03-consumer-interfaces.md)
