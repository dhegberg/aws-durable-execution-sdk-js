# Testing with the SDK

The testing SDK (`@aws/durable-execution-sdk-js-testing`) provides two test runners for validating durable functions: a local runner that executes handlers in-process with a fake checkpoint server, and a cloud runner that invokes deployed Lambda functions and polls for execution history. Both runners share a common interface for inspecting operations, asserting on results, and interacting with callbacks.

This chapter covers the full testing API surface — runner setup, execution, operation inspection, and the enums used to classify operations and their statuses.

## Local vs Cloud Testing

The SDK provides two test runners with different tradeoffs:

| Aspect | `LocalDurableTestRunner` | `CloudDurableTestRunner` |
|--------|--------------------------|--------------------------|
| Execution environment | In-process, no AWS infrastructure | Deployed Lambda function |
| Speed | Fast — no network calls | Slower — real Lambda invocations + polling |
| Time handling | Fake clock — skips waits instantly | Real time — waits execute with actual delays |
| Checkpoint server | Local in-process server | Real Lambda Durable Functions service |
| Function registration | Register handlers for `context.invoke` locally | Target function must be deployed |
| Use case | Unit tests, local development, CI pipelines | Integration tests, end-to-end validation |
| Setup complexity | Minimal — no AWS credentials needed | Requires deployed function + IAM permissions |

Use `LocalDurableTestRunner` for fast feedback during development. Use `CloudDurableTestRunner` when you need to validate behavior against the real Lambda Durable Functions service.

## `LocalDurableTestRunner`

The local test runner executes durable function handlers in-process using a local checkpoint server. It supports fake timers for skipping time-based operations, function registration for testing `context.invoke` chains, and full operation inspection.

Source: [`local-durable-test-runner.ts`](../packages/aws-durable-execution-sdk-js-testing/src/test-runner/local/local-durable-test-runner.ts)

### Environment Setup and Teardown

Before running any tests, call `setupTestEnvironment` to start the local checkpoint server. Call `teardownTestEnvironment` after all tests complete to clean up resources.

```typescript
import { LocalDurableTestRunner } from "@aws/durable-execution-sdk-js-testing";

beforeAll(async () => {
  await LocalDurableTestRunner.setupTestEnvironment({ skipTime: true });
});

afterAll(async () => {
  await LocalDurableTestRunner.teardownTestEnvironment();
});
```

#### `setupTestEnvironment(params?)`

Static method. Initializes the local checkpoint server and optionally installs fake timers.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `params.skipTime` | `boolean` | `false` | When `true`, installs fake timers that skip `context.wait` delays and step retry delays instantly. |
| `params.checkpointDelay` | `number` | `undefined` | Simulates checkpoint API latency in milliseconds. Useful for finding concurrency bugs and race conditions. |

If fake timers are already installed (e.g., via `jest.useFakeTimers()`), setup will throw an error.

#### `teardownTestEnvironment()`

Static method. Stops the checkpoint server and uninstalls fake timers. Must be called after `setupTestEnvironment` to prevent resource leaks.

### Constructor

```typescript
const runner = new LocalDurableTestRunner<ResultType>({
  handlerFunction: handler,
});
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `handlerFunction` | `DurableLambdaHandler` | The handler created with `withDurableExecution`. |

The generic type parameter `ResultType` specifies the expected return type of the durable function.

### `run(params?)`

Executes the durable function and returns a `TestResult`. The method resolves when the handler completes successfully or throws an error.

```typescript
const execution = await runner.run({ payload: { userId: "123" } });
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `params.payload` | `unknown` | Optional payload passed as the event to the handler. |

Returns: `Promise<TestResult<ResultType>>`

### Operation Retrieval Methods

These methods return `DurableOperation` instances that can be used to inspect operation details. Operations can be retrieved before or after calling `run` — the operation object resolves its data once the execution produces it.

#### `getOperation(name, index?)`

Gets the first operation (or the one at the specified index) with the given name.

```typescript
const stepOp = runner.getOperation("process-data");
const secondStep = runner.getOperation("process-item", 1);
```

#### `getOperationByIndex(index)`

Gets an operation by its execution order index (zero-based).

```typescript
const firstOp = runner.getOperationByIndex(0);
```

#### `getOperationByNameAndIndex(name, index)`

Gets a specific occurrence of a named operation. Useful when the same operation name appears multiple times (e.g., in a loop).

```typescript
const thirdRetry = runner.getOperationByNameAndIndex("fetch-data", 2);
```

#### `getOperationById(id)`

Gets an operation by its unique identifier.

```typescript
const op = runner.getOperationById("c4ca4238a0b92382");
```

### `reset()`

Clears all cached operations and history, allowing the runner to be reused for multiple test executions. Call this between test runs.

```typescript
beforeEach(() => {
  runner.reset();
});
```

### `registerDurableFunction(functionName, durableHandler)`

Registers a durable function handler that can be invoked via `context.invoke` during test execution. The `functionName` should match the ARN or name used in the handler under test. Returns `this` for method chaining.

```typescript
const paymentHandler = withDurableExecution(async (event, ctx) => {
  const result = await ctx.step("charge", async () => charge(event));
  return result;
});

runner.registerDurableFunction("payment-function-arn", paymentHandler);
```

### `registerFunction(functionName, handler)`

Registers a non-durable Lambda handler for `context.invoke` calls. Returns `this` for method chaining.

```typescript
runner.registerFunction("legacy-function", async (event) => {
  return { statusCode: 200, body: JSON.stringify(event) };
});
```

### Static Properties

#### `skipTime`

Internal static boolean. Set by `setupTestEnvironment` — controls whether fake timers are active for all runner instances.

#### `fakeClock`

Internal static property. The `@sinonjs/fake-timers` `InstalledClock` instance when `skipTime` is enabled. Available for advanced time manipulation in tests.

## `CloudDurableTestRunner`

The cloud test runner invokes a deployed Lambda function and polls the execution history API to build the operation tree. It provides the same operation inspection interface as the local runner.

Source: [`cloud-durable-test-runner.ts`](../packages/aws-durable-execution-sdk-js-testing/src/test-runner/cloud/cloud-durable-test-runner.ts)

### Constructor

```typescript
import { CloudDurableTestRunner } from "@aws/durable-execution-sdk-js-testing";
import { LambdaClient } from "@aws-sdk/client-lambda";

const runner = new CloudDurableTestRunner({
  functionName: "my-durable-function:prod",
  client: new LambdaClient({ region: "us-east-1" }),
  config: {
    pollInterval: 500,
    invocationType: "RequestResponse",
  },
});
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `functionName` | `string` | — | The name or ARN of the deployed Lambda function. Must be a qualified identifier (with version, alias, or `$LATEST`). |
| `client` | `LambdaClient` | `new LambdaClient()` | Optional AWS Lambda client instance. |
| `config.pollInterval` | `number` | `1000` | Interval in milliseconds between history polling requests. |
| `config.invocationType` | `InvocationType` | `"RequestResponse"` | Lambda invocation type. Use `"Event"` for asynchronous invocations. |

### `run(params?)`

Invokes the Lambda function, polls for execution history, and returns a `TestResult` when the execution completes.

```typescript
const execution = await runner.run({ payload: { orderId: "order-123" } });
```

Throws if the invocation response does not include a `DurableExecutionArn` — this indicates the target function is not a durable function.

### `getOperation(name)`

Gets the first operation with the specified name. Equivalent to `getOperationByNameAndIndex(name, 0)`.

### `getOperationByIndex(index)`

Gets an operation by its execution order index.

### `getOperationByNameAndIndex(name, index)`

Gets a specific occurrence of a named operation.

### `getOperationById(id)`

Gets an operation by its unique identifier.

### `reset()`

Clears cached operations and history for reuse between test runs.

## `TestResult`

Returned by both runners' `run` methods. Provides access to the execution outcome, operation history, and invocation details.

Source: [`durable-test-runner.ts`](../packages/aws-durable-execution-sdk-js-testing/src/test-runner/types/durable-test-runner.ts)

| Method | Return Type | Description |
|--------|-------------|-------------|
| `getStatus()` | `ExecutionStatus \| undefined` | The final execution status (`SUCCEEDED`, `FAILED`, `RUNNING`, `STOPPED`, `TIMED_OUT`). |
| `getResult()` | `TResult \| undefined` | The return value of the handler. Throws if the execution failed. |
| `getError()` | `TestResultError` | Error details from a failed execution. Throws if the execution succeeded. |
| `getOperations(params?)` | `DurableOperation[]` | All operations from the execution. Optionally filter by `{ status: OperationStatus }`. |
| `getInvocations()` | `Invocation[]` | Individual Lambda handler invocations with timing and request ID. |
| `getHistoryEvents()` | `Event[]` | The complete event history — low-level events from the checkpoint server. |
| `print(config?)` | `void` | Prints a formatted table of operations to the console. |

### `print` Configuration

The `print` method accepts an optional config object to control which columns appear:

```typescript
execution.print({
  parentId: true,
  name: true,
  type: true,
  subType: true,
  status: true,
  startTime: true,
  endTime: true,
  duration: true,
});
```

All columns default to being shown. Set a column to `false` to hide it.

### `TestResultError`

The error structure returned by `getError()` and used in operation details:

| Property | Type | Description |
|----------|------|-------------|
| `errorMessage` | `string \| undefined` | The error message. |
| `errorType` | `string \| undefined` | The error class name (e.g., `"StepError"`). |
| `errorData` | `string \| undefined` | Arbitrary string data attached to the error. |
| `stackTrace` | `string[] \| undefined` | Stack trace lines. |

### `Invocation`

Tracks a single handler invocation within the durable execution:

| Property | Type | Description |
|----------|------|-------------|
| `startTimestamp` | `Date \| undefined` | When the invocation started. |
| `endTimestamp` | `Date \| undefined` | When the invocation ended. |
| `requestId` | `string \| undefined` | The AWS request ID. |
| `error` | `TestResultError \| undefined` | Error information if the invocation failed. |

## `DurableOperation`

Represents a single operation within a durable execution. Provides methods to inspect the operation's type, status, result, and child operations, and to interact with callback operations.

Source: [`durable-operation.ts`](../packages/aws-durable-execution-sdk-js-testing/src/test-runner/types/durable-operation.ts)

### Identity and Metadata

| Method | Return Type | Description |
|--------|-------------|-------------|
| `getId()` | `string \| undefined` | The unique operation identifier. |
| `getParentId()` | `string \| undefined` | The parent operation's identifier. |
| `getName()` | `string \| undefined` | The operation name (as passed to `context.step("name", ...)`, etc.). |
| `getType()` | `OperationType \| undefined` | The operation type (`STEP`, `WAIT`, `CONTEXT`, `CALLBACK`, `CHAINED_INVOKE`, `EXECUTION`). |
| `getSubType()` | `OperationSubType \| undefined` | The operation subtype (e.g., `WAIT_FOR_CALLBACK`, `MAP`, `PARALLEL`). |
| `getStatus()` | `OperationStatus \| undefined` | The current status (`STARTED`, `SUCCEEDED`, `FAILED`, `CANCELLED`, `STOPPED`, `TIMED_OUT`). |
| `getStartTimestamp()` | `Date \| undefined` | When the operation started. |
| `getEndTimestamp()` | `Date \| undefined` | When the operation ended. |

### Type Checks

| Method | Return Type | Description |
|--------|-------------|-------------|
| `isWaitForCallback()` | `boolean` | `true` if this is a `CONTEXT` operation with subtype `WAIT_FOR_CALLBACK`. |
| `isCallback()` | `boolean` | `true` if this is a `CALLBACK` operation. |

### Detail Accessors

Each accessor throws if called on an operation of the wrong type.

#### `getStepDetails()`

Returns `StepDetails` for `STEP` operations:

| Property | Type | Description |
|----------|------|-------------|
| `attempt` | `number \| undefined` | The current attempt number. |
| `nextAttemptTimestamp` | `Date \| undefined` | When the next retry is scheduled. |
| `result` | `TResult \| undefined` | The step's return value. |
| `error` | `TestResultError \| undefined` | Error if the step failed. |

#### `getWaitDetails()`

Returns `WaitResultDetails` for `WAIT` operations:

| Property | Type | Description |
|----------|------|-------------|
| `waitSeconds` | `number \| undefined` | The wait duration in seconds. |
| `scheduledEndTimestamp` | `Date \| undefined` | When the wait is scheduled to complete. |

#### `getCallbackDetails()`

Returns `CallbackDetails` for callback-related operations:

| Property | Type | Description |
|----------|------|-------------|
| `callbackId` | `string` | The unique callback identifier. |
| `result` | `TResult \| undefined` | The callback result. |
| `error` | `TestResultError \| undefined` | Error if the callback failed. |

#### `getChainedInvokeDetails()`

Returns `ChainedInvokeDetails` for `CHAINED_INVOKE` operations:

| Property | Type | Description |
|----------|------|-------------|
| `result` | `TResult \| undefined` | The invoked function's return value. |
| `error` | `TestResultError \| undefined` | Error if the invocation failed. |

#### `getContextDetails()`

Returns `ContextDetails` for `CONTEXT` operations (child contexts, map iterations, parallel branches):

| Property | Type | Description |
|----------|------|-------------|
| `result` | `TResult \| undefined` | The context's return value. |
| `error` | `TestResultError \| undefined` | Error if the context failed. |

### Child Operations

#### `getChildOperations()`

Returns an array of `DurableOperation` instances representing operations nested within this operation (e.g., steps inside a child context, iterations inside a map).

### Raw Data

| Method | Return Type | Description |
|--------|-------------|-------------|
| `getOperationData()` | `Operation \| undefined` | The raw operation object from the checkpoint server. |
| `getEvents()` | `Event[] \| undefined` | Events associated with this operation. |

### Waiting for Data

#### `waitForData(status?)`

Returns a promise that resolves when the operation reaches the specified status. This is essential for callback testing — you need to wait for the callback to be submitted before sending a response.

```typescript
const callbackOp = runner.getOperation("my-callback");
const executionPromise = runner.run();

// Wait for the callback submitter to complete
await callbackOp.waitForData(WaitingOperationStatus.SUBMITTED);

// Now send the callback response
await callbackOp.sendCallbackSuccess(JSON.stringify({ approved: true }));

const result = await executionPromise;
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `status` | `WaitingOperationStatus` | `STARTED` | The status to wait for. |

Throws if the operation is not found by the time the execution completes.

### Callback Interaction Methods

These methods send responses to callback operations during test execution.

#### `sendCallbackSuccess(result?)`

Sends a successful callback response. The `result` parameter must be a JSON string.

```typescript
await callbackOp.sendCallbackSuccess(JSON.stringify({ approved: true }));
```

#### `sendCallbackFailure(error?)`

Sends a callback failure with an optional `ErrorObject`.

```typescript
await callbackOp.sendCallbackFailure({
  ErrorType: "ValidationError",
  ErrorMessage: "Invalid input",
});
```

#### `sendCallbackHeartbeat()`

Sends a heartbeat to keep the callback active and prevent timeout.

```typescript
await callbackOp.sendCallbackHeartbeat();
```

## Enums

The testing SDK re-exports several enums from `@aws-sdk/client-lambda` and defines one of its own.

### `OperationType`

Classifies the kind of operation.

| Value | Description |
|-------|-------------|
| `STEP` | A `context.step` operation. |
| `WAIT` | A `context.wait` operation. |
| `CONTEXT` | A child context — `runInChildContext`, `map` iteration, `parallel` branch, or `waitForCallback` wrapper. |
| `CALLBACK` | A `createCallback` operation. |
| `CHAINED_INVOKE` | A `context.invoke` operation. |
| `EXECUTION` | The root execution operation. |

### `OperationStatus`

The current status of an operation.

| Value | Description |
|-------|-------------|
| `STARTED` | The operation has started but not yet completed. |
| `SUCCEEDED` | The operation completed successfully. |
| `FAILED` | The operation failed. |
| `CANCELLED` | The operation was cancelled. |
| `STOPPED` | The operation was stopped. |
| `TIMED_OUT` | The operation exceeded its timeout. |

### `ExecutionStatus`

The overall status of the durable execution.

| Value | Description |
|-------|-------------|
| `SUCCEEDED` | The execution completed successfully. |
| `FAILED` | The execution failed. |
| `RUNNING` | The execution is still in progress. |
| `STOPPED` | The execution was stopped. |
| `TIMED_OUT` | The execution exceeded its timeout. |

### `InvocationType`

The Lambda invocation type used by `CloudDurableTestRunner`.

| Value | Description |
|-------|-------------|
| `RequestResponse` | Synchronous invocation — waits for the function to complete (up to 15 minutes). |
| `Event` | Asynchronous invocation — returns immediately, execution continues in the background. |
| `DryRun` | Validates the request without executing the function. |

### `WaitingOperationStatus`

Controls what `waitForData` waits for. Defined in the testing SDK.

| Value | Description |
|-------|-------------|
| `STARTED` | Fires when the operation starts. |
| `SUBMITTED` | Same as `COMPLETED`, except for `waitForCallback` — fires when the submitter function completes (before the callback response is received). |
| `COMPLETED` | Fires when the operation reaches a terminal status (`CANCELLED`, `FAILED`, `STOPPED`, `SUCCEEDED`, or `TIMED_OUT`). |

Source: [`durable-operation.ts`](../packages/aws-durable-execution-sdk-js-testing/src/test-runner/types/durable-operation.ts)

## Test Examples

### Basic Step Testing

A minimal test that executes a handler with a single step and verifies the result and operation details.

```typescript
import {
  LocalDurableTestRunner,
  OperationType,
  OperationStatus,
} from "@aws/durable-execution-sdk-js-testing";
import {
  withDurableExecution,
  DurableContext,
} from "@aws/durable-execution-sdk-js";

beforeAll(() => LocalDurableTestRunner.setupTestEnvironment());
afterAll(() => LocalDurableTestRunner.teardownTestEnvironment());

describe("basic step", () => {
  it("should execute a step and return the result", async () => {
    const handler = withDurableExecution(
      async (_event: unknown, context: DurableContext) => {
        const result = await context.step("process", async () => {
          return "step completed";
        });
        return result;
      },
    );

    const runner = new LocalDurableTestRunner({ handlerFunction: handler });
    const execution = await runner.run();

    // Verify execution result
    expect(execution.getResult()).toBe("step completed");

    // Verify operation details
    const stepOp = runner.getOperationByIndex(0);
    expect(stepOp.getType()).toBe(OperationType.STEP);
    expect(stepOp.getStatus()).toBe(OperationStatus.SUCCEEDED);
    expect(stepOp.getStepDetails()?.result).toBe("step completed");

    // Verify operation count
    expect(execution.getOperations()).toHaveLength(1);
  });
});
```

### Callback Testing

Testing `waitForCallback` requires coordinating between the test and the running execution. Use `waitForData` to wait for the callback submitter to complete, then send the response.

```typescript
import {
  LocalDurableTestRunner,
  WaitingOperationStatus,
  OperationStatus,
} from "@aws/durable-execution-sdk-js-testing";
import {
  withDurableExecution,
  DurableContext,
} from "@aws/durable-execution-sdk-js";

beforeAll(() =>
  LocalDurableTestRunner.setupTestEnvironment({ skipTime: true }),
);
afterAll(() => LocalDurableTestRunner.teardownTestEnvironment());

describe("callback", () => {
  it("should wait for callback and receive the response", async () => {
    let capturedCallbackId: string | undefined;

    const handler = withDurableExecution(
      async (_event: unknown, context: DurableContext) => {
        const approval = await context.waitForCallback<{ approved: boolean }>(
          "wait-for-approval",
          async (callbackId) => {
            capturedCallbackId = callbackId;
          },
          { timeout: { hours: 24 } },
        );
        return { approved: approval.approved };
      },
    );

    const runner = new LocalDurableTestRunner({ handlerFunction: handler });

    // Get the callback operation handle BEFORE running
    const callbackOp = runner.getOperation("wait-for-approval");

    // Start execution (don't await yet)
    const executionPromise = runner.run();

    // Wait for the submitter to complete
    await callbackOp.waitForData(WaitingOperationStatus.SUBMITTED);

    // Send the callback response
    await callbackOp.sendCallbackSuccess(
      JSON.stringify({ approved: true }),
    );

    // Now await the execution result
    const execution = await executionPromise;

    expect(execution.getResult()).toEqual({ approved: true });
    expect(capturedCallbackId).toBeDefined();
  });
});
```

### Wait Testing

Testing `context.wait` operations. Enable `skipTime` to avoid real delays.

```typescript
import {
  LocalDurableTestRunner,
  OperationType,
  OperationStatus,
} from "@aws/durable-execution-sdk-js-testing";
import {
  withDurableExecution,
  DurableContext,
} from "@aws/durable-execution-sdk-js";

beforeAll(() =>
  LocalDurableTestRunner.setupTestEnvironment({ skipTime: true }),
);
afterAll(() => LocalDurableTestRunner.teardownTestEnvironment());

describe("wait", () => {
  it("should execute wait and verify duration", async () => {
    const handler = withDurableExecution(
      async (_event: unknown, context: DurableContext) => {
        await context.wait("cooldown", { seconds: 30 });
        return "completed after wait";
      },
    );

    const runner = new LocalDurableTestRunner({ handlerFunction: handler });
    const execution = await runner.run();

    expect(execution.getResult()).toBe("completed after wait");

    const waitOp = runner.getOperation("cooldown");
    expect(waitOp.getType()).toBe(OperationType.WAIT);
    expect(waitOp.getStatus()).toBe(OperationStatus.SUCCEEDED);
    expect(waitOp.getWaitDetails()?.waitSeconds).toBe(30);
    expect(waitOp.getWaitDetails()?.scheduledEndTimestamp).toBeInstanceOf(Date);
  });
});
```

### Invoke Testing

Testing `context.invoke` with registered durable functions.

```typescript
import {
  LocalDurableTestRunner,
} from "@aws/durable-execution-sdk-js-testing";
import {
  withDurableExecution,
  DurableContext,
} from "@aws/durable-execution-sdk-js";

beforeAll(() => LocalDurableTestRunner.setupTestEnvironment());
afterAll(() => LocalDurableTestRunner.teardownTestEnvironment());

describe("invoke", () => {
  it("should invoke a registered durable function", async () => {
    const mainHandler = withDurableExecution(
      async (_event: unknown, context: DurableContext) => {
        const result = await context.invoke(
          "call-processor",
          "processor-function-arn",
          { input: "data" },
        );
        return { processorResult: result };
      },
    );

    const processorHandler = withDurableExecution(
      async (event: { input: string }, context: DurableContext) => {
        const processed = await context.step("process", async () => {
          return `processed: ${event.input}`;
        });
        return processed;
      },
    );

    const runner = new LocalDurableTestRunner({ handlerFunction: mainHandler });
    runner.registerDurableFunction("processor-function-arn", processorHandler);

    const execution = await runner.run();

    expect(execution.getResult()).toEqual({
      processorResult: "processed: data",
    });

    const invokeOp = runner.getOperation("call-processor");
    expect(invokeOp.getChainedInvokeDetails()?.result).toBe("processed: data");
  });
});
```

### Multi-Step Workflow with Invocations Tracking

A more complete example showing how to verify invocation count and operation history across multiple replay cycles.

```typescript
import {
  LocalDurableTestRunner,
  OperationStatus,
} from "@aws/durable-execution-sdk-js-testing";
import {
  withDurableExecution,
  DurableContext,
} from "@aws/durable-execution-sdk-js";

beforeAll(() =>
  LocalDurableTestRunner.setupTestEnvironment({ skipTime: true }),
);
afterAll(() => LocalDurableTestRunner.teardownTestEnvironment());

describe("multi-step workflow", () => {
  it("should track operations and invocations across replays", async () => {
    const handler = withDurableExecution(
      async (_event: unknown, context: DurableContext) => {
        await context.wait("delay-1", { seconds: 1 });
        const data = await context.step("process-data", async () => {
          return { processed: true, timestamp: Date.now() };
        });
        await context.wait("delay-2", { seconds: 1 });
        return { result: data, completed: true };
      },
    );

    const runner = new LocalDurableTestRunner({ handlerFunction: handler });
    const execution = await runner.run();

    // Verify final result
    const result = execution.getResult() as {
      result: { processed: boolean };
      completed: boolean;
    };
    expect(result.completed).toBe(true);
    expect(result.result.processed).toBe(true);

    // Verify operations
    expect(execution.getOperations()).toHaveLength(3);

    // Verify invocations (waits cause separate invocations)
    const invocations = execution.getInvocations();
    expect(invocations.length).toBeGreaterThanOrEqual(2);
    expect(invocations[0].requestId).toBeDefined();

    // Filter operations by status
    const succeeded = execution.getOperations({
      status: OperationStatus.SUCCEEDED,
    });
    expect(succeeded).toHaveLength(3);

    // Print operation table for debugging
    execution.print();
  });
});
```

---

[← Previous: Error Handling](./07-error-handling.md) | [Next: Common Patterns →](./09-common-patterns.md)
