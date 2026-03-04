# Configuration Reference

Every durable operation on `DurableContext` accepts an optional configuration object that controls retry behavior, serialization, concurrency, timeouts, and more. This chapter is a comprehensive reference for all consumer-facing configuration interfaces in the SDK.

For conceptual explanations of how these operations work, see [Consumer Interfaces](./03-consumer-interfaces.md). This chapter focuses on the configuration knobs and their precise types, defaults, and behavior.

## Duration

The `Duration` type is used throughout the SDK wherever a time interval is needed — retry delays, wait durations, timeouts, and backoff configurations. It is a union type that requires at least one time unit to be specified.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `days` | `number` | Conditional | Number of days. Must be a positive integer. |
| `hours` | `number` | Conditional | Number of hours. Must be a positive integer. |
| `minutes` | `number` | Conditional | Number of minutes. Must be a positive integer. |
| `seconds` | `number` | Conditional | Number of seconds. Must be a positive integer. |

At least one unit must be specified. Units can be combined — the SDK converts the total to seconds internally using `durationToSeconds()`.

```typescript
// Single unit
const thirtySeconds: Duration = { seconds: 30 };
const oneHour: Duration = { hours: 1 };
const sevenDays: Duration = { days: 7 };

// Combined units
const ninetyMinutes: Duration = { hours: 1, minutes: 30 };
const complexDuration: Duration = { days: 1, hours: 6, minutes: 30, seconds: 15 };
```

The type is defined as a discriminated union to enforce that at least one unit is present:

```typescript
type Duration =
  | { days: number; hours?: number; minutes?: number; seconds?: number }
  | { hours: number; minutes?: number; seconds?: number }
  | { minutes: number; seconds?: number }
  | { seconds: number };
```

> **Source**: [`types/core.ts`](../packages/aws-durable-execution-sdk-js/src/types/core.ts), [`utils/duration/duration.ts`](../packages/aws-durable-execution-sdk-js/src/utils/duration/duration.ts)

## StepConfig

Configuration for `context.step()` operations. Controls retry behavior, execution semantics, and serialization.

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `retryStrategy` | `(error: Error, attemptCount: number) => RetryDecision` | `undefined` | Strategy function that decides whether to retry a failed step and how long to wait. See [RetryStrategyConfig](#retrystrategyconfig) for the factory approach. |
| `semantics` | `StepSemantics` | `AtLeastOncePerRetry` | Controls checkpoint timing relative to step execution. |
| `serdes` | `Serdes<T>` | `undefined` | Custom serialization/deserialization for step results. Falls back to `defaultSerdes` (JSON). |

### StepSemantics

| Value | Description |
|-------|-------------|
| `AtLeastOncePerRetry` | Default. Checkpoint is created after step execution. If the checkpoint fails (e.g., sandbox crash), the step re-executes on replay. Safe for idempotent operations. |
| `AtMostOncePerRetry` | Checkpoint is created before step execution. If a failure occurs after checkpoint but before completion, the step is skipped on replay. Use for non-idempotent operations like payment processing. |

> **Important**: These guarantees apply *per retry attempt*. With retries enabled, a step can still execute multiple times across different retry attempts even with `AtMostOncePerRetry`. To guarantee at-most-once execution overall, combine `AtMostOncePerRetry` with a no-retry strategy.

```typescript
// Default: at-least-once, no custom retry
await context.step("fetch-data", async () => fetchData());

// At-most-once with no retries for payment processing
await context.step("charge-payment", async () => processPayment(), {
  semantics: StepSemantics.AtMostOncePerRetry,
  retryStrategy: () => ({ shouldRetry: false }),
});

// Custom retry with class-based serdes
await context.step("create-user", async () => {
  const user = new User();
  user.name = "Alice";
  return user;
}, {
  retryStrategy: createRetryStrategy({ maxAttempts: 5 }),
  serdes: createClassSerdes(User),
});
```

### RetryDecision

The return type of retry strategy functions:

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `shouldRetry` | `boolean` | — | Whether the operation should be retried. |
| `delay` | `Duration` | `{ seconds: 1 }` | Delay before the next retry attempt. Only used when `shouldRetry` is `true`. |

> **Source**: [`types/step.ts`](../packages/aws-durable-execution-sdk-js/src/types/step.ts)

## RetryStrategyConfig

Configuration object for the `createRetryStrategy()` factory function. This factory produces a retry strategy function suitable for `StepConfig.retryStrategy` and `WaitForCallbackConfig.retryStrategy`.

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `maxAttempts` | `number` | `3` | Maximum total attempts including the initial attempt. Set to `1` to disable retries. |
| `initialDelay` | `Duration` | `{ seconds: 5 }` | Delay before the first retry. |
| `maxDelay` | `Duration` | `{ minutes: 5 }` | Maximum delay between retries (caps exponential growth). |
| `backoffRate` | `number` | `2` | Multiplier applied to the delay on each subsequent retry. |
| `jitter` | `JitterStrategy` | `FULL` | Jitter strategy to randomize delays and prevent thundering herd. |
| `retryableErrors` | `(string \| RegExp)[]` | All errors (when both filters undefined) | Error message patterns that are retryable. Strings use `includes()`, RegExp uses `test()`. |
| `retryableErrorTypes` | `(new () => Error)[]` | `[]` | Error class constructors that are retryable. Uses `instanceof` checks. |

### JitterStrategy

| Value | Description |
|-------|-------------|
| `NONE` | No jitter — use the exact calculated delay. |
| `FULL` | Random delay between 0 and the calculated delay. |
| `HALF` | Random delay between 50% and 100% of the calculated delay. |

### Delay Calculation

The retry delay follows exponential backoff with jitter:

1. **Base delay** = `initialDelay × backoffRate^(attemptsMade - 1)`
2. **Capped** at `maxDelay`
3. **Jitter** applied based on the selected strategy
4. **Rounded** to the nearest second, minimum 1 second

### Error Filtering

- If **neither** `retryableErrors` nor `retryableErrorTypes` is specified: all errors are retried
- If **either** is specified: only matching errors are retried
- If **both** are specified: errors matching **either** criteria are retried (OR logic)

### createRetryStrategy Examples

```typescript
import { createRetryStrategy, JitterStrategy } from "@aws/durable-execution-sdk-js";

// Default: retry all errors, 3 attempts, exponential backoff
const defaultRetry = createRetryStrategy();

// Custom configuration
const customRetry = createRetryStrategy({
  maxAttempts: 5,
  initialDelay: { seconds: 10 },
  maxDelay: { seconds: 60 },
  backoffRate: 2,
  jitter: JitterStrategy.HALF,
});

// Retry only specific error types
class TimeoutError extends Error {}
class NetworkError extends Error {}

const typeBasedRetry = createRetryStrategy({
  retryableErrorTypes: [TimeoutError, NetworkError],
});

// Retry only errors matching message patterns
const patternBasedRetry = createRetryStrategy({
  retryableErrors: [/timeout/i, /connection refused/i, "rate limit"],
});

// Combine error types and patterns (OR logic)
const combinedRetry = createRetryStrategy({
  retryableErrorTypes: [TimeoutError],
  retryableErrors: [/network/i],
});

// Use in a step
await context.step("api-call", async () => callExternalAPI(), {
  retryStrategy: customRetry,
});
```

> **Source**: [`utils/retry/retry-config/index.ts`](../packages/aws-durable-execution-sdk-js/src/utils/retry/retry-config/index.ts)

## retryPresets

Pre-configured retry strategies for common use cases. These are ready-to-use functions that can be passed directly to `StepConfig.retryStrategy`.

### `retryPresets.default`

| Parameter | Value |
|-----------|-------|
| `maxAttempts` | `6` (1 initial + 5 retries) |
| `initialDelay` | `{ seconds: 5 }` |
| `maxDelay` | `{ seconds: 60 }` |
| `backoffRate` | `2` |
| `jitter` | `FULL` |
| Total max wait | ~150 seconds (2:30) |

### `retryPresets.noRetry`

| Parameter | Value |
|-----------|-------|
| `maxAttempts` | `1` (no retries) |

```typescript
import { retryPresets } from "@aws/durable-execution-sdk-js";

// Use default retry preset
await context.step("resilient-call", async () => callAPI(), {
  retryStrategy: retryPresets.default,
});

// Fail immediately on error
await context.step("critical-step", async () => criticalOperation(), {
  retryStrategy: retryPresets.noRetry,
});
```

> **Source**: [`utils/retry/retry-presets/retry-presets.ts`](../packages/aws-durable-execution-sdk-js/src/utils/retry/retry-presets/retry-presets.ts)

## WaitForCallbackConfig

Configuration for `context.waitForCallback()` operations. Controls timeouts, heartbeats, retry behavior for the submitter function, and deserialization of callback data.

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `timeout` | `Duration` | `undefined` | Maximum time to wait for the callback to be completed by an external system. If exceeded, a `CallbackTimeoutError` is thrown. |
| `heartbeatTimeout` | `Duration` | `undefined` | Maximum time between heartbeat signals. If no heartbeat is received within this window, the callback is considered stalled. |
| `retryStrategy` | `(error: Error, attemptCount: number) => RetryDecision` | `undefined` | Retry strategy for the submitter function. If the submitter throws, this determines whether to retry submitting the callback ID. |
| `serdes` | `Omit<Serdes<TOutput>, "serialize">` | `undefined` | Deserialization configuration for callback result data. Only `deserialize` is needed since the external system provides the serialized value. |

```typescript
import { createRetryStrategy } from "@aws/durable-execution-sdk-js";

const approval = await context.waitForCallback(
  "wait-for-approval",
  async (callbackId, ctx) => {
    ctx.logger.info("Sending approval request", { callbackId });
    await sendApprovalEmail(callbackId);
  },
  {
    timeout: { hours: 24 },
    heartbeatTimeout: { minutes: 30 },
    retryStrategy: createRetryStrategy({ maxAttempts: 3 }),
  },
);
```

> **Source**: [`types/callback.ts`](../packages/aws-durable-execution-sdk-js/src/types/callback.ts)

## CreateCallbackConfig

Configuration for `context.createCallback()` operations. Similar to `WaitForCallbackConfig` but without the submitter retry strategy, since `createCallback` returns the callback ID directly without a submitter function.

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `timeout` | `Duration` | `undefined` | Maximum time to wait for the callback to be completed. |
| `heartbeatTimeout` | `Duration` | `undefined` | Maximum time between heartbeat signals before the callback is considered stalled. |
| `serdes` | `Omit<Serdes<TOutput>, "serialize">` | `undefined` | Deserialization configuration for callback result data. Only `deserialize` is needed. |

```typescript
const [resultPromise, callbackId] = await context.createCallback(
  "manual-callback",
  {
    timeout: { hours: 48 },
    heartbeatTimeout: { hours: 1 },
  },
);

// Use callbackId externally, then await the result
await context.step("submit-callback", async () => {
  await submitToExternalSystem(callbackId);
});

const result = await resultPromise;
```

> **Source**: [`types/callback.ts`](../packages/aws-durable-execution-sdk-js/src/types/callback.ts)

## WaitForConditionConfig

Configuration for `context.waitForCondition()` operations. Defines the polling strategy, initial state, and serialization for condition-based waiting.

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `waitStrategy` | `WaitForConditionWaitStrategyFunc<T>` | — (required) | Function that determines whether to continue polling and the delay between checks. See [WaitStrategyConfig](#waitstrategyconfig) for the factory approach. |
| `initialState` | `T` | — (required) | Initial state value passed to the first check function invocation. |
| `serdes` | `Serdes<T>` | `undefined` | Custom serialization/deserialization for the condition state. |

The `waitStrategy` function receives the current state and attempt number, and returns a `WaitForConditionDecision`:

```typescript
type WaitForConditionDecision =
  | { shouldContinue: true; delay: Duration }
  | { shouldContinue: false };
```

```typescript
const finalState = await context.waitForCondition(
  "wait-for-job",
  async (state, ctx) => {
    const status = await checkJobStatus(state.jobId);
    return { ...state, status };
  },
  {
    initialState: { jobId: "job-123", status: "pending" },
    waitStrategy: (state, attempt) => {
      if (state.status === "completed" || state.status === "failed") {
        return { shouldContinue: false };
      }
      return {
        shouldContinue: true,
        delay: { seconds: Math.min(attempt * 5, 60) },
      };
    },
  },
);
```

> **Source**: [`types/wait-condition.ts`](../packages/aws-durable-execution-sdk-js/src/types/wait-condition.ts)

## WaitStrategyConfig

Configuration object for the `createWaitStrategy()` factory function. This factory produces a wait strategy function suitable for `WaitForConditionConfig.waitStrategy`.

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `maxAttempts` | `number` | `60` | Maximum number of polling attempts before throwing an error. |
| `initialDelay` | `Duration` | `{ seconds: 5 }` | Delay before the first poll. |
| `maxDelay` | `Duration` | `{ seconds: 300 }` (5 min) | Maximum delay between polls (caps exponential growth). |
| `backoffRate` | `number` | `1.5` | Multiplier applied to the delay on each subsequent poll. |
| `jitter` | `JitterStrategy` | `FULL` | Jitter strategy to randomize polling intervals. |
| `shouldContinuePolling` | `(result: T) => boolean` | — (required) | Function that returns `true` if polling should continue based on the current state. |

If `maxAttempts` is exceeded, the strategy throws an error: `"waitForCondition exceeded maximum attempts (N)"`.

### createWaitStrategy Example

```typescript
import { createWaitStrategy, JitterStrategy } from "@aws/durable-execution-sdk-js";

interface JobState {
  jobId: string;
  status: string;
}

const jobWaitStrategy = createWaitStrategy<JobState>({
  maxAttempts: 30,
  initialDelay: { seconds: 10 },
  maxDelay: { minutes: 2 },
  backoffRate: 1.5,
  jitter: JitterStrategy.HALF,
  shouldContinuePolling: (state) => state.status === "pending",
});

const finalState = await context.waitForCondition(
  "poll-job",
  async (state) => {
    const status = await checkJobStatus(state.jobId);
    return { ...state, status };
  },
  {
    initialState: { jobId: "job-456", status: "pending" },
    waitStrategy: jobWaitStrategy,
  },
);
```

> **Source**: [`utils/wait-strategy/wait-strategy-config.ts`](../packages/aws-durable-execution-sdk-js/src/utils/wait-strategy/wait-strategy-config.ts)

## MapConfig

Configuration for `context.map()` operations. Controls concurrency, completion behavior, naming, and serialization.

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `maxConcurrency` | `number` | `undefined` (unlimited) | Maximum number of items executing concurrently. When a slot frees up, the next item starts. |
| `itemNamer` | `(item: TItem, index: number) => string` | `undefined` | Function to generate custom names for each map iteration (used in checkpoint metadata). |
| `serdes` | `Serdes<BatchResult<TResult>>` | `undefined` | Serialization/deserialization for the overall `BatchResult`. |
| `itemSerdes` | `Serdes<TResult>` | `undefined` | Serialization/deserialization for individual item results. |
| `completionConfig` | `CompletionConfig` | `undefined` | Early completion criteria. See [CompletionConfig](#completionconfig). |

```typescript
const results = await context.map(
  "process-orders",
  orders,
  async (ctx, order, index) => {
    return await ctx.step(`process-${index}`, async () => processOrder(order));
  },
  {
    maxConcurrency: 5,
    itemNamer: (order, index) => `order-${order.id}`,
    completionConfig: {
      minSuccessful: 8,
      toleratedFailureCount: 2,
    },
  },
);
```

> **Source**: [`types/batch.ts`](../packages/aws-durable-execution-sdk-js/src/types/batch.ts)

## ParallelConfig

Configuration for `context.parallel()` operations. Controls concurrency, completion behavior, and serialization for parallel branch execution.

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `maxConcurrency` | `number` | `undefined` (unlimited) | Maximum number of branches executing concurrently. |
| `serdes` | `Serdes<BatchResult<TResult>>` | `undefined` | Serialization/deserialization for the overall `BatchResult`. |
| `itemSerdes` | `Serdes<TResult>` | `undefined` | Serialization/deserialization for individual branch results. |
| `completionConfig` | `CompletionConfig` | `undefined` | Early completion criteria. See [CompletionConfig](#completionconfig). |

```typescript
const results = await context.parallel(
  "parallel-tasks",
  [
    { name: "fetch-users", func: async (ctx) => ctx.step(async () => fetchUsers()) },
    { name: "fetch-orders", func: async (ctx) => ctx.step(async () => fetchOrders()) },
    { name: "fetch-inventory", func: async (ctx) => ctx.step(async () => fetchInventory()) },
  ],
  {
    maxConcurrency: 2,
    completionConfig: {
      toleratedFailureCount: 1,
    },
  },
);
```

> **Source**: [`types/batch.ts`](../packages/aws-durable-execution-sdk-js/src/types/batch.ts)

## CompletionConfig

Configuration for early completion of `map` and `parallel` operations. When any threshold is met, the operation completes and remaining items receive `STARTED` status in the `BatchResult`.

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `minSuccessful` | `number` | `undefined` | Minimum number of successful executions required. Once reached, the operation completes early with `MIN_SUCCESSFUL_REACHED` reason. |
| `toleratedFailureCount` | `number` | `undefined` | Maximum number of failures tolerated. If exceeded, the operation completes early with `FAILURE_TOLERANCE_EXCEEDED` reason. |
| `toleratedFailurePercentage` | `number` | `undefined` | Maximum percentage of failures tolerated (0–100). If exceeded, the operation completes early. |

When multiple thresholds are configured, the first one reached triggers completion. If no `CompletionConfig` is provided, the operation waits for all items to complete (`ALL_COMPLETED` reason).

> **Note**: Due to the asynchronous nature of concurrent execution, the actual number of completed items may slightly exceed the threshold by the time the completion check runs.

```typescript
// Stop after 10 successes, even if more items remain
{ minSuccessful: 10 }

// Stop if more than 3 items fail
{ toleratedFailureCount: 3 }

// Stop if more than 20% of items fail
{ toleratedFailurePercentage: 20 }

// Combine: stop after 10 successes OR if 5 fail
{ minSuccessful: 10, toleratedFailureCount: 5 }
```

> **Source**: [`types/batch.ts`](../packages/aws-durable-execution-sdk-js/src/types/batch.ts)

## ChildConfig

Configuration for `context.runInChildContext()` operations. Controls serialization, categorization, and error mapping for child context executions.

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `serdes` | `Serdes<T>` | `undefined` | Serialization/deserialization for the child context result. |
| `subType` | `string` | `undefined` | Sub-type identifier for categorizing child contexts in checkpoint metadata. Used internally by `map` and `parallel` to distinguish iteration types. |
| `summaryGenerator` | `(result: T) => string` | `undefined` | Function to generate a summary string from the result. Used internally by `map`/`parallel` for checkpoint metadata. |
| `errorMapper` | `(originalError: DurableOperationError) => DurableOperationError` | `undefined` | Function to transform child context errors into custom error types. Receives the original `DurableOperationError` and returns a (potentially different) `DurableOperationError`. |

```typescript
import { DurableOperationError } from "@aws/durable-execution-sdk-js";

const result = await context.runInChildContext(
  "process-batch",
  async (childCtx) => {
    const data = await childCtx.step("fetch", async () => fetchData());
    await childCtx.wait({ seconds: 5 });
    return await childCtx.step("transform", async () => transform(data));
  },
  {
    serdes: createClassSerdes(BatchResult),
    errorMapper: (originalError) => {
      // Wrap child errors in a domain-specific error
      const mapped = new DurableOperationError(
        `Batch processing failed: ${originalError.message}`,
      );
      return mapped;
    },
  },
);
```

> **Source**: [`types/child-context.ts`](../packages/aws-durable-execution-sdk-js/src/types/child-context.ts)

## InvokeConfig

Configuration for `context.invoke()` operations. Controls serialization for both the input payload sent to the target function and the result received back.

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `payloadSerdes` | `Serdes<I>` | `undefined` | Serialization/deserialization for the input payload sent to the invoked function. |
| `resultSerdes` | `Serdes<O>` | `undefined` | Serialization/deserialization for the result returned by the invoked function. |

```typescript
const result = await context.invoke(
  "call-processor",
  process.env.PROCESSOR_ARN!,
  { orderId: "12345", amount: 99.99 },
  {
    payloadSerdes: customPayloadSerdes,
    resultSerdes: createClassSerdes(ProcessorResult),
  },
);
```

> **Source**: [`types/invoke.ts`](../packages/aws-durable-execution-sdk-js/src/types/invoke.ts)

## DurableExecutionConfig

Top-level configuration for the `withDurableExecution()` wrapper. Passed as the second argument to customize the durable execution runtime.

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `client` | `LambdaClient` | `undefined` | Custom AWS Lambda client instance for checkpoint and state management API calls. If not provided, a default client is created using the standard AWS SDK configuration chain. |

Use cases for a custom client include:
- Custom AWS region or credentials configuration
- Testing with mocked Lambda clients
- Advanced networking (VPC endpoints, proxies)
- Custom retry and timeout settings for the underlying AWS SDK calls

```typescript
import { LambdaClient } from "@aws-sdk/client-lambda";
import { withDurableExecution } from "@aws/durable-execution-sdk-js";

const customClient = new LambdaClient({
  region: "us-west-2",
  maxAttempts: 5,
  retryMode: "adaptive",
});

export const handler = withDurableExecution(
  async (event, context) => {
    // Your durable workflow
    return { success: true };
  },
  { client: customClient },
);
```

> **Source**: [`types/durable-execution.ts`](../packages/aws-durable-execution-sdk-js/src/types/durable-execution.ts)

## LoggerConfig

Configuration for `context.configureLogger()`. Controls the logger implementation and replay-aware behavior.

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `customLogger` | `Logger extends DurableLogger` | Default console logger | Custom logger implementation. Must implement the `DurableLogger` interface (`error`, `warn`, `info`, `debug`, and optionally `log` and `configureDurableLoggingContext`). |
| `modeAware` | `boolean` | `true` | When `true`, log messages are suppressed during replay to avoid duplicate output. Set to `false` to see all log messages including replayed ones. |

### DurableLogger Interface

Any custom logger must implement this interface:

| Method | Required | Description |
|--------|----------|-------------|
| `error(...params: any)` | Yes | Log error messages. |
| `warn(...params: any)` | Yes | Log warning messages. |
| `info(...params: any)` | Yes | Log informational messages. |
| `debug(...params: any)` | Yes | Log debug messages. |
| `log?(level: string, ...params: any)` | No | Generic log method with configurable level. |
| `configureDurableLoggingContext?(ctx: DurableLoggingContext)` | No | Called by the SDK to provide durable execution metadata (execution ARN, operation ID, attempt number) for structured logging. |

```typescript
import { DurableLogger, DurableLoggingContext } from "@aws/durable-execution-sdk-js";

// Custom structured logger
class MyLogger implements DurableLogger {
  private durableContext?: DurableLoggingContext;

  configureDurableLoggingContext(ctx: DurableLoggingContext) {
    this.durableContext = ctx;
  }

  info(...params: any[]) {
    const logData = this.durableContext?.getDurableLogData();
    console.log(JSON.stringify({
      level: "INFO",
      executionArn: logData?.executionArn,
      operationId: logData?.operationId,
      message: params,
    }));
  }

  warn(...params: any[]) { /* ... */ }
  error(...params: any[]) { /* ... */ }
  debug(...params: any[]) { /* ... */ }
}

// Configure in handler
export const handler = withDurableExecution(async (event, context) => {
  context.configureLogger({
    customLogger: new MyLogger() as any,
    modeAware: true,
  });

  context.logger.info("Starting workflow"); // Suppressed during replay
  // ...
});
```

> **Source**: [`types/logger.ts`](../packages/aws-durable-execution-sdk-js/src/types/logger.ts), [`types/durable-logger.ts`](../packages/aws-durable-execution-sdk-js/src/types/durable-logger.ts)

## Serdes

The `Serdes<T>` interface controls how data is serialized for checkpointing and deserialized during replay. Every configuration that accepts a `serdes` property uses this interface.

| Method | Signature | Description |
|--------|-----------|-------------|
| `serialize` | `(value: T \| undefined, context: SerdesContext) => Promise<string \| undefined>` | Converts a value to a string for checkpoint storage. |
| `deserialize` | `(data: string \| undefined, context: SerdesContext) => Promise<T \| undefined>` | Converts a stored string back to the original value. |

### SerdesContext

Both methods receive a `SerdesContext` with metadata about the current operation:

| Property | Type | Description |
|----------|------|-------------|
| `entityId` | `string` | Unique identifier for the step or operation being serialized. |
| `durableExecutionArn` | `string` | ARN of the durable execution, useful for avoiding collisions in external storage. |

Both methods are async to support implementations that interact with external services (S3, DynamoDB, etc.) for large payloads.

### defaultSerdes

The built-in default serializer uses `JSON.stringify` and `JSON.parse`. It handles any JSON-serializable value and is used automatically when no custom `serdes` is provided.

```typescript
import { defaultSerdes } from "@aws/durable-execution-sdk-js";

// Equivalent to the built-in behavior:
const result = await context.step("my-step", async () => ({ key: "value" }), {
  serdes: defaultSerdes,
});
```

### createClassSerdes

Creates a `Serdes` for a specific class that preserves the class prototype during deserialization. The class constructor must have no required parameters.

```typescript
import { createClassSerdes } from "@aws/durable-execution-sdk-js";

class User {
  name: string = "";
  age: number = 0;

  greet() {
    return `Hello, ${this.name}`;
  }
}

const userSerdes = createClassSerdes(User);

const user = await context.step("create-user", async () => {
  const u = new User();
  u.name = "Alice";
  u.age = 30;
  return u;
}, { serdes: userSerdes });

console.log(user.greet()); // "Hello, Alice" — methods are preserved
```

**Limitations:**
- Constructor must have no required parameters
- Constructor side-effects re-run during deserialization
- Private fields (`#field`) cannot be serialized
- Getters/setters are not preserved
- Nested class instances lose their prototype

### createClassSerdesWithDates

Extends `createClassSerdes` with automatic `Date` property restoration. Date values are serialized as ISO strings by `JSON.stringify` and need explicit conversion back to `Date` objects.

```typescript
import { createClassSerdesWithDates } from "@aws/durable-execution-sdk-js";

class Article {
  title: string = "";
  createdAt: Date = new Date();
  metadata: {
    publishedAt: Date;
    updatedAt: Date;
  } = {
    publishedAt: new Date(),
    updatedAt: new Date(),
  };

  getAge() {
    return Date.now() - this.createdAt.getTime();
  }
}

// Specify which properties are Dates (supports nested paths)
const articleSerdes = createClassSerdesWithDates(Article, [
  "createdAt",
  "metadata.publishedAt",
  "metadata.updatedAt",
]);

const article = await context.step("create-article", async () => {
  const a = new Article();
  a.title = "My Article";
  return a;
}, { serdes: articleSerdes });

console.log(article.getAge()); // Works — Dates are properly restored
```

The `dateProps` parameter accepts dot-notation paths for nested properties (e.g., `"metadata.publishedAt"`).

### Custom Serdes Example

For full control over serialization, implement the `Serdes` interface directly. This example stores large payloads in S3:

```typescript
import { Serdes, SerdesContext } from "@aws/durable-execution-sdk-js";
import { S3Client, PutObjectCommand, GetObjectCommand } from "@aws-sdk/client-s3";

const s3Client = new S3Client({});
const BUCKET = "my-durable-state-bucket";

const s3Serdes: Serdes<LargePayload> = {
  serialize: async (value, context: SerdesContext) => {
    if (value === undefined) return undefined;

    const key = `${context.durableExecutionArn}/${context.entityId}`;
    await s3Client.send(new PutObjectCommand({
      Bucket: BUCKET,
      Key: key,
      Body: JSON.stringify(value),
    }));

    // Return a pointer instead of the full payload
    return JSON.stringify({ s3Key: key });
  },

  deserialize: async (data, context: SerdesContext) => {
    if (data === undefined) return undefined;

    const { s3Key } = JSON.parse(data);
    const response = await s3Client.send(new GetObjectCommand({
      Bucket: BUCKET,
      Key: s3Key,
    }));

    const body = await response.Body?.transformToString();
    return body ? JSON.parse(body) : undefined;
  },
};

// Use with any operation
await context.step("process-large-data", async () => generateLargePayload(), {
  serdes: s3Serdes,
});
```

> **Source**: [`utils/serdes/serdes.ts`](../packages/aws-durable-execution-sdk-js/src/utils/serdes/serdes.ts)

## Configuration Quick Reference

A summary of which configuration applies to each `DurableContext` method:

| Method | Config Type | Key Options |
|--------|-------------|-------------|
| `step` | `StepConfig` | `retryStrategy`, `semantics`, `serdes` |
| `wait` | — | No configuration (only accepts `Duration`) |
| `invoke` | `InvokeConfig` | `payloadSerdes`, `resultSerdes` |
| `runInChildContext` | `ChildConfig` | `serdes`, `subType`, `errorMapper` |
| `waitForCallback` | `WaitForCallbackConfig` | `timeout`, `heartbeatTimeout`, `retryStrategy`, `serdes` |
| `createCallback` | `CreateCallbackConfig` | `timeout`, `heartbeatTimeout`, `serdes` |
| `waitForCondition` | `WaitForConditionConfig` | `waitStrategy`, `initialState`, `serdes` |
| `map` | `MapConfig` | `maxConcurrency`, `completionConfig`, `serdes`, `itemSerdes` |
| `parallel` | `ParallelConfig` | `maxConcurrency`, `completionConfig`, `serdes`, `itemSerdes` |
| `configureLogger` | `LoggerConfig` | `customLogger`, `modeAware` |
| `withDurableExecution` | `DurableExecutionConfig` | `client` |

---

[← Previous: Threading, Concurrency, and Execution Model](./05-threading-and-concurrency.md) | [Next: Error Handling →](./07-error-handling.md)
