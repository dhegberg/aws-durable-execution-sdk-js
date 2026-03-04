# Consumer Interfaces

This chapter documents every consumer-facing interface in the Durable Functions SDK — the `withDurableExecution` wrapper, every method on `DurableContext`, the `DurablePromise` class, promise combinators, the replay-aware logger, and access to the underlying Lambda context.

This is the core API surface. Each method includes its signature, parameter descriptions, return type, configuration options, and at least one TypeScript code example.

## `withDurableExecution`

The entry point for the SDK. This function wraps a `DurableExecutionHandler` and returns a `DurableLambdaHandler` that Lambda can invoke directly.

```typescript
function withDurableExecution<TEvent, TResult, TLogger extends DurableLogger>(
  handler: DurableExecutionHandler<TEvent, TResult, TLogger>,
  config?: DurableExecutionConfig,
): DurableLambdaHandler;
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `handler` | `DurableExecutionHandler<TEvent, TResult, TLogger>` | Your business logic handler that receives the event and a `DurableContext` |
| `config` | `DurableExecutionConfig` (optional) | Optional configuration — currently supports a custom `LambdaClient` via the `client` property |

**Returns:** `DurableLambdaHandler` — a function that accepts `DurableExecutionInvocationInput` and a Lambda `Context`, returning `Promise<DurableExecutionInvocationOutput>`.

When invoked, the returned handler:
1. Validates the incoming `DurableExecutionInvocationInput`
2. Initializes the execution context (loads operation history, sets up checkpoint management)
3. Runs your handler, providing the extracted event payload and a `DurableContext`
4. Returns a `DurableExecutionInvocationOutput` with status `SUCCEEDED`, `FAILED`, or `PENDING`

```typescript
import { withDurableExecution, DurableContext } from "@aws/durable-execution-sdk-js";

const durableHandler = async (event: { userId: string }, context: DurableContext) => {
  const user = await context.step("fetch-user", async () => fetchUser(event.userId));
  return { success: true, user };
};

export const handler = withDurableExecution(durableHandler);
```

**Source:** [`packages/aws-durable-execution-sdk-js/src/types/durable-execution.ts`](../packages/aws-durable-execution-sdk-js/src/types/durable-execution.ts)

## `DurableContext`

The primary interface exposed to consumers. Every durable operation — steps, waits, invokes, callbacks, map, parallel — is a method on this interface.

```typescript
interface DurableContext<TLogger extends DurableLogger = DurableLogger>
```

`DurableContext` is generic over `TLogger`, which defaults to `DurableLogger`. This allows custom logger implementations to flow through to all operations.

**Source:** [`packages/aws-durable-execution-sdk-js/src/types/durable-context.ts`](../packages/aws-durable-execution-sdk-js/src/types/durable-context.ts)

---

### `step`

Executes a function as a durable step with automatic checkpointing and retry. Steps are the fundamental building block — each step's result is persisted, and on replay the step returns its checkpointed result without re-executing.

```typescript
// Named overload (recommended)
step<TOutput>(
  name: string | undefined,
  fn: StepFunc<TOutput, TLogger>,
  config?: StepConfig<TOutput>,
): DurablePromise<TOutput>;

// Unnamed overload
step<TOutput>(
  fn: StepFunc<TOutput, TLogger>,
  config?: StepConfig<TOutput>,
): DurablePromise<TOutput>;
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `name` | `string \| undefined` | Step name for tracking and debugging. Recommended for all steps. |
| `fn` | `StepFunc<TOutput, TLogger>` | Function to execute. Receives a `StepContext` (for logging only), not a full `DurableContext`. |
| `config` | `StepConfig<TOutput>` (optional) | Retry strategy, execution semantics, and serialization options. |

**Returns:** `DurablePromise<TOutput>` — resolves with the step result.

**Throws:** `StepError` when the step function fails after all retry attempts are exhausted.

**Important:** `step()` is for single atomic operations. You cannot call other durable operations (steps, waits, invokes) inside a step function. To group multiple durable operations, use [`runInChildContext`](#runinchildcontext).

```typescript
// Basic step with retry
const user = await context.step("fetch-user", async (ctx) => {
  ctx.logger.info("Fetching user data");
  return await fetchUserFromAPI(userId);
});

// Step with custom retry strategy
const result = await context.step(
  "call-external-api",
  async () => callExternalAPI(payload),
  {
    retryStrategy: (error, attemptCount) => ({
      shouldRetry: attemptCount < 3,
      delay: { seconds: Math.pow(2, attemptCount) },
    }),
  }
);
```

**Source:** [`packages/aws-durable-execution-sdk-js/src/types/durable-context.ts`](../packages/aws-durable-execution-sdk-js/src/types/durable-context.ts) | [`packages/aws-durable-execution-sdk-js/src/types/step.ts`](../packages/aws-durable-execution-sdk-js/src/types/step.ts)


---

### `wait`

Pauses execution for a specified duration. The function suspends without consuming compute — the Lambda sandbox is released and the execution resumes after the duration elapses.

```typescript
// Named overload
wait(name: string, duration: Duration): DurablePromise<void>;

// Unnamed overload
wait(duration: Duration): DurablePromise<void>;
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `name` | `string` | Step name for tracking and debugging |
| `duration` | `Duration` | How long to wait. Supports `days`, `hours`, `minutes`, `seconds` — combinable in a single object. |

**Returns:** `DurablePromise<void>`

The `Duration` type is an object with optional numeric fields:

```typescript
interface Duration {
  days?: number;
  hours?: number;
  minutes?: number;
  seconds?: number;
}
```

```typescript
// Wait 30 seconds
await context.wait("rate-limit-delay", { seconds: 30 });

// Wait 1 hour and 30 minutes
await context.wait("long-delay", { hours: 1, minutes: 30 });

// Wait 7 days
await context.wait("weekly-check", { days: 7 });
```

**Source:** [`packages/aws-durable-execution-sdk-js/src/types/durable-context.ts`](../packages/aws-durable-execution-sdk-js/src/types/durable-context.ts)

---

### `invoke`

Invokes another durable or non-durable Lambda function. The invocation is checkpointed — on replay, the result is returned from the checkpoint without re-invoking the target function.

```typescript
// Named overload
invoke<TInput, TOutput>(
  name: string,
  funcId: string,
  input?: TInput,
  config?: InvokeConfig<TInput, TOutput>,
): DurablePromise<TOutput>;

// Unnamed overload
invoke<TInput, TOutput>(
  funcId: string,
  input?: TInput,
  config?: InvokeConfig<TInput, TOutput>,
): DurablePromise<TOutput>;
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `name` | `string` | Step name for tracking and debugging |
| `funcId` | `string` | Function ID or ARN. Alias/version qualifier required for durable functions. |
| `input` | `TInput` (optional) | Input data to pass to the invoked function |
| `config` | `InvokeConfig<TInput, TOutput>` (optional) | Serialization options for payload and result (`payloadSerdes`, `resultSerdes`) |

**Returns:** `DurablePromise<TOutput>` — resolves with the invoked function's result.

**Throws:** `InvokeError` when the invoked function fails or times out.

```typescript
// Invoke by ARN with version qualifier
const paymentResult = await context.invoke(
  "process-payment",
  "arn:aws:lambda:us-east-1:123456789012:function:payment-processor:1",
  { amount: 100, currency: "USD" }
);

// Invoke by function name with alias
const result = await context.invoke(
  "validate-order",
  "order-validator:prod",
  { orderId: "order-123" }
);
```

**Source:** [`packages/aws-durable-execution-sdk-js/src/types/durable-context.ts`](../packages/aws-durable-execution-sdk-js/src/types/durable-context.ts) | [`packages/aws-durable-execution-sdk-js/src/types/invoke.ts`](../packages/aws-durable-execution-sdk-js/src/types/invoke.ts)

---

### `runInChildContext`

Runs a function in a child context with isolated state and execution tracking. Child contexts have their own step counters, which is essential for deterministic replay of grouped operations.

```typescript
// Named overload
runInChildContext<TOutput>(
  name: string | undefined,
  fn: ChildFunc<TOutput, TLogger>,
  config?: ChildConfig<TOutput>,
): DurablePromise<TOutput>;

// Unnamed overload
runInChildContext<TOutput>(
  fn: ChildFunc<TOutput, TLogger>,
  config?: ChildConfig<TOutput>,
): DurablePromise<TOutput>;
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `name` | `string \| undefined` | Step name for tracking and debugging |
| `fn` | `ChildFunc<TOutput, TLogger>` | Function that receives a full `DurableContext` — unlike `step`, you can call any durable operation inside. |
| `config` | `ChildConfig<TOutput>` (optional) | Serialization, sub-type identifier, summary generator, and error mapper options. |

**Returns:** `DurablePromise<TOutput>` — resolves with the child function's result.

**Throws:** `ChildContextError` when the child context function fails.

Use `runInChildContext` when you need to group multiple durable operations (steps, waits, invokes) into a single logical unit:

```typescript
const orderResult = await context.runInChildContext(
  "process-order",
  async (childCtx) => {
    const validated = await childCtx.step("validate", async () =>
      validateOrder(order)
    );
    await childCtx.wait("processing-delay", { seconds: 5 });
    const processed = await childCtx.step("charge", async () =>
      chargePayment(validated)
    );
    return processed;
  }
);
```

**Source:** [`packages/aws-durable-execution-sdk-js/src/types/durable-context.ts`](../packages/aws-durable-execution-sdk-js/src/types/durable-context.ts) | [`packages/aws-durable-execution-sdk-js/src/types/child-context.ts`](../packages/aws-durable-execution-sdk-js/src/types/child-context.ts)


---

### `waitForCallback`

Pauses execution until an external system completes a callback using the `SendDurableExecutionCallbackSuccess` or `SendDurableExecutionCallbackFailure` Lambda APIs. The SDK generates a unique callback ID, passes it to your submitter function, then suspends.

```typescript
// Named overload
waitForCallback<TOutput = string>(
  name: string | undefined,
  submitter: WaitForCallbackSubmitterFunc<TLogger>,
  config?: WaitForCallbackConfig<TOutput>,
): DurablePromise<TOutput>;

// Unnamed overload
waitForCallback<TOutput = string>(
  submitter: WaitForCallbackSubmitterFunc<TLogger>,
  config?: WaitForCallbackConfig<TOutput>,
): DurablePromise<TOutput>;
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `name` | `string \| undefined` | Step name for tracking and debugging |
| `submitter` | `WaitForCallbackSubmitterFunc<TLogger>` | Function that receives the `callbackId` and a context. Responsible for sending the callback ID to the external system. |
| `config` | `WaitForCallbackConfig<TOutput>` (optional) | Timeout, heartbeat timeout, retry strategy for the submitter, and deserialization options. |

**Returns:** `DurablePromise<TOutput>` — resolves with the value sent by the external system via `SendDurableExecutionCallbackSuccess`. Defaults to `string`.

**Throws:** `CallbackError`, `CallbackTimeoutError`, or `CallbackSubmitterError` depending on the failure mode.

```typescript
const approval = await context.waitForCallback(
  "wait-for-approval",
  async (callbackId, ctx) => {
    ctx.logger.info("Sending approval request", { callbackId });
    await sendApprovalEmail(approverEmail, callbackId);
  },
  { timeout: { hours: 24 } }
);

if (approval === "APPROVED") {
  await context.step("execute-plan", async () => executePlan(plan));
}
```

**Source:** [`packages/aws-durable-execution-sdk-js/src/types/durable-context.ts`](../packages/aws-durable-execution-sdk-js/src/types/durable-context.ts) | [`packages/aws-durable-execution-sdk-js/src/types/callback.ts`](../packages/aws-durable-execution-sdk-js/src/types/callback.ts)

---

### `createCallback`

Creates a callback that external systems can complete, returning both a promise and the callback ID. Unlike `waitForCallback`, this gives you direct control over when to await the callback result — useful when you need to do additional work between creating the callback and waiting for it.

```typescript
// Named overload
createCallback<TOutput = string>(
  name: string | undefined,
  config?: CreateCallbackConfig<TOutput>,
): DurablePromise<CreateCallbackResult<TOutput>>;

// Unnamed overload
createCallback<TOutput = string>(
  config?: CreateCallbackConfig<TOutput>,
): DurablePromise<CreateCallbackResult<TOutput>>;
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `name` | `string \| undefined` | Step name for tracking and debugging |
| `config` | `CreateCallbackConfig<TOutput>` (optional) | Timeout, heartbeat timeout, and deserialization options. |

**Returns:** `DurablePromise<CreateCallbackResult<TOutput>>` — resolves to a tuple `[DurablePromise<TOutput>, string]` where the first element is a promise that resolves when the callback is completed, and the second is the callback ID.

**Throws:** `CallbackError` when the callback fails, times out, or the external system reports failure (thrown by the returned promise).

```typescript
const [callbackPromise, callbackId] = await context.createCallback(
  "external-approval",
  { timeout: { hours: 1 } }
);

// Send the callback ID to an external system
await context.step("send-request", async () =>
  sendApprovalRequest(callbackId, requestData)
);

// Do other work while waiting...
await context.step("prepare-data", async () => prepareData());

// Now wait for the external system to respond
const approvalResult = await callbackPromise;
```

**Source:** [`packages/aws-durable-execution-sdk-js/src/types/durable-context.ts`](../packages/aws-durable-execution-sdk-js/src/types/durable-context.ts) | [`packages/aws-durable-execution-sdk-js/src/types/callback.ts`](../packages/aws-durable-execution-sdk-js/src/types/callback.ts)

---

### `waitForCondition`

Polls a condition by periodically executing a check function until the wait strategy signals completion. Useful for monitoring external job status, waiting for resource availability, or any scenario where you need to poll until a condition is met.

```typescript
// Named overload
waitForCondition<TOutput>(
  name: string | undefined,
  checkFunc: WaitForConditionCheckFunc<TOutput, TLogger>,
  config: WaitForConditionConfig<TOutput>,
): DurablePromise<TOutput>;

// Unnamed overload
waitForCondition<TOutput>(
  checkFunc: WaitForConditionCheckFunc<TOutput, TLogger>,
  config: WaitForConditionConfig<TOutput>,
): DurablePromise<TOutput>;
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `name` | `string \| undefined` | Step name for tracking and debugging |
| `checkFunc` | `WaitForConditionCheckFunc<TOutput, TLogger>` | Function that receives the current state and a context, returns updated state. Called on each polling iteration. |
| `config` | `WaitForConditionConfig<TOutput>` (required) | Must include `waitStrategy` and `initialState`. Optionally includes `serdes`. |

**Returns:** `DurablePromise<TOutput>` — resolves with the final state when the wait strategy returns `{ shouldContinue: false }`.

**Throws:** `WaitForConditionError` on failure.

The `config` object has three properties:

| Property | Type | Description |
|----------|------|-------------|
| `waitStrategy` | `(state: TOutput, attempt: number) => WaitForConditionDecision` | Returns `{ shouldContinue: true, delay: Duration }` to keep polling, or `{ shouldContinue: false }` to stop. |
| `initialState` | `TOutput` | The starting state value passed to the first check. |
| `serdes` | `Serdes<TOutput>` (optional) | Custom serialization for the state. |

```typescript
const finalState = await context.waitForCondition(
  "wait-for-job-completion",
  async (currentState, ctx) => {
    const jobStatus = await checkJobStatus(currentState.jobId);
    return { ...currentState, status: jobStatus };
  },
  {
    initialState: { jobId: "job-123", status: "pending" },
    waitStrategy: (state, attempt) => {
      if (state.status === "completed" || state.status === "failed") {
        return { shouldContinue: false };
      }
      return {
        shouldContinue: true,
        delay: { seconds: Math.min(attempt * 2, 60) },
      };
    },
  }
);
```

**Source:** [`packages/aws-durable-execution-sdk-js/src/types/durable-context.ts`](../packages/aws-durable-execution-sdk-js/src/types/durable-context.ts) | [`packages/aws-durable-execution-sdk-js/src/types/wait-condition.ts`](../packages/aws-durable-execution-sdk-js/src/types/wait-condition.ts)


---

### `map`

Processes an array of items in parallel, applying a function to each item in its own child context. Supports concurrency control and early completion policies.

```typescript
// Named overload
map<TInput, TOutput>(
  name: string | undefined,
  items: TInput[],
  mapFunc: MapFunc<TInput, TOutput, TLogger>,
  config?: MapConfig<TInput, TOutput>,
): DurablePromise<BatchResult<TOutput>>;

// Unnamed overload
map<TInput, TOutput>(
  items: TInput[],
  mapFunc: MapFunc<TInput, TOutput, TLogger>,
  config?: MapConfig<TInput, TOutput>,
): DurablePromise<BatchResult<TOutput>>;
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `name` | `string \| undefined` | Step name for tracking and debugging |
| `items` | `TInput[]` | Array of items to process |
| `mapFunc` | `MapFunc<TInput, TOutput, TLogger>` | Function receiving `(context, item, index, array)` — the context is a full `DurableContext` for each item's child context. |
| `config` | `MapConfig<TInput, TOutput>` (optional) | Concurrency, completion behavior, item naming, and serialization options. |

**Returns:** `DurablePromise<BatchResult<TOutput>>` — a `BatchResult` containing all item results, errors, and completion metadata.

The `MapConfig` options:

| Property | Type | Description |
|----------|------|-------------|
| `maxConcurrency` | `number` (optional) | Maximum simultaneous executions. Default: unlimited. |
| `itemNamer` | `(item, index) => string` (optional) | Custom name generator for each item. |
| `completionConfig` | `CompletionConfig` (optional) | Early completion rules: `minSuccessful`, `toleratedFailureCount`, `toleratedFailurePercentage`. |
| `serdes` | `Serdes<BatchResult<TOutput>>` (optional) | Serialization for the parent result. |
| `itemSerdes` | `Serdes<TOutput>` (optional) | Serialization for each item result. |

```typescript
const results = await context.map(
  "process-orders",
  orders,
  async (ctx, order, index) => {
    const validated = await ctx.step("validate", async () =>
      validateOrder(order)
    );
    const processed = await ctx.step("process", async () =>
      processOrder(validated)
    );
    return processed;
  },
  {
    maxConcurrency: 5,
    itemNamer: (order, index) => `order-${order.id}`,
    completionConfig: {
      toleratedFailureCount: 2,
    },
  }
);

// Inspect results
const successfulResults = results.getResults();
if (results.hasFailure) {
  const errors = results.getErrors();
  // Handle failures...
}
```

**Source:** [`packages/aws-durable-execution-sdk-js/src/types/durable-context.ts`](../packages/aws-durable-execution-sdk-js/src/types/durable-context.ts) | [`packages/aws-durable-execution-sdk-js/src/types/batch.ts`](../packages/aws-durable-execution-sdk-js/src/types/batch.ts)

---

### `parallel`

Executes multiple branches in parallel, each in its own child context. Similar to `map` but for heterogeneous operations rather than processing an array of identical items.

```typescript
// Named overload (homogeneous return type)
parallel<TOutput>(
  name: string | undefined,
  branches: (ParallelFunc<TOutput, TLogger> | NamedParallelBranch<TOutput, TLogger>)[],
  config?: ParallelConfig<TOutput>,
): DurablePromise<BatchResult<TOutput>>;

// Unnamed overload (homogeneous return type)
parallel<TOutput>(
  branches: (ParallelFunc<TOutput, TLogger> | NamedParallelBranch<TOutput, TLogger>)[],
  config?: ParallelConfig<TOutput>,
): DurablePromise<BatchResult<TOutput>>;

// Inferred overload (heterogeneous return types)
parallel<Branches extends readonly unknown[]>(
  name: string | undefined,
  branches: Branches,
  config?: ParallelConfig<...>,
): Promise<BatchResult<...>>;
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `name` | `string \| undefined` | Step name for tracking and debugging |
| `branches` | Array of `ParallelFunc` or `NamedParallelBranch` | Functions to execute in parallel. Each receives a full `DurableContext`. |
| `config` | `ParallelConfig<TOutput>` (optional) | Concurrency, completion behavior, and serialization options. |

**Returns:** `DurablePromise<BatchResult<TOutput>>` — a `BatchResult` containing all branch results.

Branches can be plain functions or named branches:

```typescript
// Plain functions — all branches return the same type
const results = await context.parallel<string>("parallel-ops", [
  async (ctx) => ctx.step("task-1", async () => "result-1"),
  async (ctx) => ctx.step("task-2", async () => "result-2"),
]);

// Named branches — useful for debugging and tracking
const results = await context.parallel("parallel-ops", [
  { name: "fetch-user", func: async (ctx) => ctx.step(async () => fetchUser(id)) },
  { name: "fetch-posts", func: async (ctx) => ctx.step(async () => fetchPosts(id)) },
]);

// Heterogeneous return types — TypeScript infers the union
const results = await context.parallel("mixed-ops", [
  async (ctx) => ctx.step(async () => ({ user: "data" })),
  async (ctx) => ctx.step(async () => "string-result"),
]);
// results: BatchResult<{ user: string } | string>
```

The `ParallelConfig` options:

| Property | Type | Description |
|----------|------|-------------|
| `maxConcurrency` | `number` (optional) | Maximum simultaneous executions. Default: unlimited. |
| `completionConfig` | `CompletionConfig` (optional) | Early completion rules. |
| `serdes` | `Serdes<BatchResult<TOutput>>` (optional) | Serialization for the parent result. |
| `itemSerdes` | `Serdes<TOutput>` (optional) | Serialization for each branch result. |

**Source:** [`packages/aws-durable-execution-sdk-js/src/types/durable-context.ts`](../packages/aws-durable-execution-sdk-js/src/types/durable-context.ts) | [`packages/aws-durable-execution-sdk-js/src/types/batch.ts`](../packages/aws-durable-execution-sdk-js/src/types/batch.ts)

---

### `configureLogger`

Configures the logger for this context. Allows replacing the default logger with a custom implementation and controlling replay-aware behavior.

```typescript
configureLogger(config: LoggerConfig<TLogger>): void;
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `config` | `LoggerConfig<TLogger>` | Logger configuration with `customLogger` and `modeAware` options. |

**Returns:** `void`

The `LoggerConfig` options:

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `customLogger` | `TLogger` (optional) | Default console logger | Custom logger implementation. Must implement the `DurableLogger` interface (`log`, `error`, `warn`, `info`, `debug`). |
| `modeAware` | `boolean` (optional) | `true` | When `true`, logs are suppressed during replay to avoid duplicate output. Set to `false` to see logs on every execution including replays. |

```typescript
// Use a custom logger (e.g., AWS Lambda Powertools)
import { Logger } from "@aws-lambda-powertools/logger";

const powertoolsLogger = new Logger({ serviceName: "my-service" });
context.configureLogger({ customLogger: powertoolsLogger });

// Disable mode-aware logging to see logs during replay
context.configureLogger({ modeAware: false });

// Both together
context.configureLogger({
  customLogger: powertoolsLogger,
  modeAware: false,
});
```

**Source:** [`packages/aws-durable-execution-sdk-js/src/types/durable-context.ts`](../packages/aws-durable-execution-sdk-js/src/types/durable-context.ts) | [`packages/aws-durable-execution-sdk-js/src/types/logger.ts`](../packages/aws-durable-execution-sdk-js/src/types/logger.ts)


## `context.promise` — Promise Combinators

The `promise` property on `DurableContext` provides four combinators that mirror the native `Promise` static methods: `all`, `allSettled`, `any`, and `race`. Each accepts an array of `DurablePromise` instances and returns a `DurablePromise`.

All combinators have two overloads — one with an optional `name` parameter for tracking, and one without.

### When to Use Promise Combinators vs `map`/`parallel`

Promise combinators accept already-created promises that start executing immediately. They provide no concurrency control, no completion policies, and no durability across Lambda timeouts.

| Feature | Promise Combinators | `map` / `parallel` |
|---------|--------------------|--------------------|
| Concurrency control | No — all promises run immediately | Yes — `maxConcurrency` limits simultaneous executions |
| Completion policies | No | Yes — `minSuccessful`, `toleratedFailureCount` |
| Durability | No — if Lambda times out, progress is lost | Yes — survives timeouts, resumes from checkpoints |
| Per-item retry | No | Yes — each item runs in its own child context |
| Use case | Fast, in-memory coordination of already-running operations | Controlled, durable execution of independent work items |

**Use promise combinators** for lightweight coordination of operations that are already running and will complete quickly. **Use `map`/`parallel`** for anything that needs durability, concurrency control, or failure tolerance.

### `promise.all`

Waits for all promises to resolve. Rejects if any promise rejects.

```typescript
all<TOutput>(
  name: string | undefined,
  promises: DurablePromise<TOutput>[],
): DurablePromise<TOutput[]>;

all<TOutput>(
  promises: DurablePromise<TOutput>[],
): DurablePromise<TOutput[]>;
```

```typescript
const step1 = context.step("fetch-user", async () => fetchUser(id));
const step2 = context.step("fetch-posts", async () => fetchPosts(id));

const [user, posts] = await context.promise.all("fetch-all", [step1, step2]);
```

### `promise.allSettled`

Waits for all promises to settle (resolve or reject). Never rejects — returns an array of `PromiseSettledResult<TOutput>` objects with `status: "fulfilled"` or `status: "rejected"`.

```typescript
allSettled<TOutput>(
  name: string | undefined,
  promises: DurablePromise<TOutput>[],
): DurablePromise<PromiseSettledResult<TOutput>[]>;

allSettled<TOutput>(
  promises: DurablePromise<TOutput>[],
): DurablePromise<PromiseSettledResult<TOutput>[]>;
```

```typescript
const results = await context.promise.allSettled("fetch-all", [
  context.step("task-1", async () => riskyOperation1()),
  context.step("task-2", async () => riskyOperation2()),
]);

for (const result of results) {
  if (result.status === "fulfilled") {
    console.log("Success:", result.value);
  } else {
    console.log("Failed:", result.reason);
  }
}
```

### `promise.any`

Waits for the first promise to resolve successfully. Ignores rejections unless all promises reject (in which case it throws an `AggregateError`).

```typescript
any<TOutput>(
  name: string | undefined,
  promises: DurablePromise<TOutput>[],
): DurablePromise<TOutput>;

any<TOutput>(
  promises: DurablePromise<TOutput>[],
): DurablePromise<TOutput>;
```

```typescript
const userData = await context.promise.any("fetch-from-any-source", [
  context.step("primary-db", async () => fetchFromPrimaryDB(userId)),
  context.step("cache", async () => fetchFromCache(userId)),
  context.step("secondary-db", async () => fetchFromSecondaryDB(userId)),
]);
```

### `promise.race`

Returns the result of the first promise to settle (resolve or reject). Unlike `any`, a rejection wins the race.

```typescript
race<TOutput>(
  name: string | undefined,
  promises: DurablePromise<TOutput>[],
): DurablePromise<TOutput>;

race<TOutput>(
  promises: DurablePromise<TOutput>[],
): DurablePromise<TOutput>;
```

```typescript
// Race against a timeout
const result = await context.promise.race("fetch-with-timeout", [
  context.step("fetch-data", async () => fetchFromSlowAPI(id)),
  context.step("timeout", async () => {
    await new Promise((_, reject) =>
      setTimeout(() => reject(new Error("Timeout")), 5000)
    );
  }),
]);
```

**Source:** [`packages/aws-durable-execution-sdk-js/src/types/durable-context.ts`](../packages/aws-durable-execution-sdk-js/src/types/durable-context.ts)

## `DurablePromise`

Every durable operation returns a `DurablePromise<T>` rather than a native `Promise<T>`. The key difference: `DurablePromise` uses lazy execution — the underlying operation does not start until the promise is awaited or chained (`.then`, `.catch`, `.finally`).

```typescript
class DurablePromise<T> implements Promise<T> {
  constructor(executor: () => Promise<T>);

  then<TResult1, TResult2>(
    onfulfilled?: (value: T) => TResult1 | PromiseLike<TResult1>,
    onrejected?: (reason: unknown) => TResult2 | PromiseLike<TResult2>,
  ): Promise<TResult1 | TResult2>;

  catch<TResult>(
    onrejected?: (reason: unknown) => TResult | PromiseLike<TResult>,
  ): Promise<T | TResult>;

  finally(onfinally?: () => void): Promise<T>;

  get isExecuted(): boolean;
  get [Symbol.toStringTag](): string; // "DurablePromise"
}
```

### Lazy Execution

A `DurablePromise` does not execute its underlying function when created. Execution begins only when the promise is first awaited or chained:

```typescript
// Creating the promise does NOT start execution
const promise = context.step("expensive-operation", async () => {
  console.log("This runs only when awaited");
  return await expensiveComputation();
});

console.log(promise.isExecuted); // false — nothing has run yet

// NOW the step executes
const result = await promise;
console.log(promise.isExecuted); // true
```

### `isExecuted`

The `isExecuted` property returns `true` once the promise has been awaited or chained, `false` otherwise. This is useful for checking whether a deferred operation has started.

### Difference from Native `Promise`

| Behavior | Native `Promise` | `DurablePromise` |
|----------|-----------------|-----------------|
| Execution timing | Executor runs immediately on construction | Executor runs only when awaited or chained |
| `isExecuted` property | Not available | Available — tracks whether execution has started |
| `Promise` compatibility | Is a `Promise` | Implements `Promise<T>` — works with `await`, `.then`, `.catch`, `.finally` |

This lazy behavior is what enables the promise combinators (`context.promise.all`, etc.) to work correctly — you can create multiple `DurablePromise` instances and then coordinate them without triggering execution until you're ready.

**Source:** [`packages/aws-durable-execution-sdk-js/src/types/durable-promise.ts`](../packages/aws-durable-execution-sdk-js/src/types/durable-promise.ts)


## `context.logger` — Replay-Aware Logging

The `logger` property on `DurableContext` provides structured, replay-aware logging. It exposes the standard log methods: `log`, `info`, `warn`, `error`, and `debug`.

```typescript
logger: DurableContextLogger<TLogger>;
```

The `DurableContextLogger` type is a pick of the logging methods from the underlying logger:

```typescript
type DurableContextLogger<Logger extends DurableLogger> = Pick<
  Logger,
  "log" | "warn" | "info" | "error" | "debug"
>;
```

### Replay-Aware Behavior

By default, the logger is mode-aware — it suppresses log output during replay to prevent duplicate log entries. On a first execution, all logs emit normally. On replay, logs are silenced until the SDK reaches new (non-replayed) operations.

This is important because durable functions re-execute from the beginning on every replay. Without mode-aware logging, you'd see duplicate log entries for every operation that was already completed.

```typescript
// Safe to use anywhere — automatically suppressed during replay
context.logger.info("Processing order", { orderId: "order-123" });
context.logger.error("Payment failed", paymentError, { retryCount: 3 });
context.logger.debug("Step details", { stepName: "validate", duration: 150 });
```

To see logs during replay (useful for debugging), disable mode-aware logging:

```typescript
context.configureLogger({ modeAware: false });
```

### Default Log Format

The default logger emits structured JSON:

```json
{
  "timestamp": "2025-11-21T18:39:24.743Z",
  "executionArn": "arn:aws:lambda:...",
  "level": "INFO",
  "operationId": "abc123",
  "message": { "userId": "123", "action": "login" },
  "requestId": "72171fff-70c9-4066-b819-11d3eb549de0"
}
```

### Custom Loggers

You can replace the default logger with any implementation that satisfies the `DurableLogger` interface. The interface requires `error`, `warn`, `info`, and `debug` methods, with an optional `log` method for generic level-based logging:

```typescript
interface DurableLogger {
  log?(level: string, ...params: any): void;
  error(...params: any): void;
  warn(...params: any): void;
  info(...params: any): void;
  debug(...params: any): void;
  configureDurableLoggingContext?(context: DurableLoggingContext): void;
}
```

The optional `configureDurableLoggingContext` method is called by the SDK to provide durable execution metadata (execution ARN, operation ID, attempt number) that custom loggers can include in their output.

**Source:** [`packages/aws-durable-execution-sdk-js/src/types/durable-logger.ts`](../packages/aws-durable-execution-sdk-js/src/types/durable-logger.ts) | [`packages/aws-durable-execution-sdk-js/src/types/logger.ts`](../packages/aws-durable-execution-sdk-js/src/types/logger.ts)

## `context.lambdaContext` — Underlying Lambda Context

The `lambdaContext` property provides access to the standard AWS Lambda `Context` object. This is the same context that a non-durable Lambda handler receives.

```typescript
lambdaContext: Context;
```

Use this to access Lambda runtime information like the function name, memory limit, remaining time, and request ID:

```typescript
const handler = async (event: any, context: DurableContext) => {
  const functionName = context.lambdaContext.functionName;
  const remainingTime = context.lambdaContext.getRemainingTimeInMillis();
  const requestId = context.lambdaContext.awsRequestId;

  context.logger.info("Execution info", {
    functionName,
    remainingTime,
    requestId,
  });

  // Use remaining time to decide whether to start a long operation
  if (remainingTime < 30000) {
    context.logger.warn("Less than 30s remaining, skipping optional step");
  }
};
```

The `Context` type comes from the `aws-lambda` package and includes properties like `functionName`, `functionVersion`, `memoryLimitInMB`, `awsRequestId`, `logGroupName`, `logStreamName`, and the `getRemainingTimeInMillis()` method.

## `context.executionContext` — Execution Metadata

The `executionContext` property provides readonly metadata about the current durable execution:

```typescript
executionContext: {
  readonly durableExecutionArn: string;
};
```

```typescript
const arn = context.executionContext.durableExecutionArn;
context.logger.info("Running execution", { arn });
```

## Key Takeaways

- `withDurableExecution` wraps your handler and manages all durable execution mechanics. The optional `DurableExecutionConfig` allows injecting a custom Lambda client.
- `DurableContext` is the primary consumer interface. Every durable operation is a method on this interface, and every method returns a `DurablePromise`.
- `step` is for single atomic operations. `runInChildContext` is for grouping multiple durable operations. Never nest durable operations inside a step.
- `wait` suspends execution without compute charges. `waitForCallback` and `createCallback` integrate with external systems. `waitForCondition` polls until a condition is met.
- `invoke` calls other Lambda functions with checkpointed results. `map` and `parallel` provide concurrent execution with concurrency control and completion policies.
- Promise combinators (`all`, `allSettled`, `any`, `race`) coordinate already-running promises but lack durability and concurrency control — prefer `map`/`parallel` for most use cases.
- `DurablePromise` is lazy — it doesn't execute until awaited. This enables flexible composition of operations.
- `context.logger` is replay-aware by default, suppressing duplicate logs during replay. Use `configureLogger` to customize.
- `context.lambdaContext` provides access to the standard Lambda runtime context.

---

[← Previous: Lambda Function Structure](./02-lambda-function-structure.md) | [Next: API Interaction and Request Paths →](./04-api-interaction.md)