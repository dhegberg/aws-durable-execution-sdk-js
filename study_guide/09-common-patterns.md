# Common Patterns and Use Cases

Durable functions are well-suited to workflows that span multiple steps, involve external systems, or need to survive failures and restarts. This chapter presents six patterns that appear frequently in production durable function applications. Each pattern includes a description of the problem it solves, guidance on when to use it, a complete code example, and key considerations for implementation.

All patterns build on the primitives covered in earlier chapters — [steps](./03-consumer-interfaces.md#step), [waits](./03-consumer-interfaces.md#wait), [callbacks](./03-consumer-interfaces.md#waitforcallback), [map/parallel](./03-consumer-interfaces.md#map), and [child contexts](./03-consumer-interfaces.md#runinchildcontext). Refer to those sections for detailed API documentation.

## GenAI Agentic Loop

An agentic loop drives an AI model through multiple rounds of reasoning and tool use until the model produces a final answer. Each iteration invokes the model, checks whether it wants to call a tool, executes the tool if so, and feeds the result back. The loop terminates when the model returns a response without requesting a tool call.

### When to Use

- Building AI agents that use tools (code execution, web search, database queries)
- Multi-turn model interactions where the number of iterations is not known in advance
- Workflows where each model invocation and tool execution should be checkpointed independently

### Code Example

```typescript
import { withDurableExecution, DurableContext } from "@aws/durable-execution-sdk-js";

interface AgentEvent {
  prompt: string;
  model: string;
}

interface ModelResponse {
  response: string;
  tool?: { name: string; input: Record<string, unknown> };
}

export const handler = withDurableExecution(async (event: AgentEvent, context: DurableContext) => {
  const messages: Array<{ role: string; content: string }> = [
    { role: "user", content: event.prompt },
  ];

  let iteration = 0;

  while (true) {
    // Each model invocation is a checkpointed step
    const modelResult = await context.step(`invoke-model-${iteration}`, async () => {
      return await invokeAIModel(event.model, messages);
    });

    // If no tool call, the model is done
    if (modelResult.tool == null) {
      return modelResult.response;
    }

    // Execute the requested tool in its own step
    const toolResult = await context.step(
      `tool-${modelResult.tool.name}-${iteration}`,
      async () => {
        return await executeTool(modelResult.tool!.name, modelResult.tool!.input);
      },
    );

    // Feed the tool result back to the model
    messages.push({ role: "assistant", content: JSON.stringify(modelResult) });
    messages.push({ role: "tool", content: toolResult });

    iteration++;
  }
});
```

### Key Considerations

- Each step name must be unique across iterations. Append an iteration counter or use a unique identifier to avoid name collisions during replay.
- The `messages` array is rebuilt on each replay because it lives outside steps. This is safe as long as the model invocations and tool executions return the same checkpointed results.
- Model invocations and tool executions are separate steps, so a failure in tool execution does not re-invoke the model.
- For long-running agents, consider adding a maximum iteration guard to prevent unbounded loops.
- Non-deterministic values (like timestamps or random IDs) used in tool execution must be generated inside steps to ensure deterministic replay.

## Human-in-the-Loop Approval

This pattern pauses a workflow to wait for an external human decision — an approval, review, or manual input. The function sends a callback ID to an external system (email, Slack, web UI), then suspends without consuming compute. When the human responds, the external system calls the Lambda callback API to resume execution.

### When to Use

- Approval gates in deployment pipelines
- Content moderation workflows requiring human review
- Order processing that needs manual confirmation above a threshold
- Any workflow where a human decision is required before proceeding

### Code Example

```typescript
import { withDurableExecution, DurableContext } from "@aws/durable-execution-sdk-js";

interface OrderEvent {
  orderId: string;
  amount: number;
  approverEmail: string;
}

export const handler = withDurableExecution(async (event: OrderEvent, context: DurableContext) => {
  // Step 1: Validate the order
  const order = await context.step("validate-order", async () => {
    return await validateOrder(event.orderId, event.amount);
  });

  // Step 2: Wait for human approval (suspends execution — no compute charges)
  const approval = await context.waitForCallback<{ approved: boolean; reason?: string }>(
    "wait-for-approval",
    async (callbackId) => {
      // The submitter sends the callback ID to the approver
      await sendApprovalEmail(event.approverEmail, {
        orderId: event.orderId,
        amount: event.amount,
        callbackId,
      });
    },
    { timeout: { hours: 48 } },
  );

  // Step 3: Act on the decision
  if (approval.approved) {
    await context.step("process-order", async () => {
      return await processOrder(order);
    });
    return { status: "completed", orderId: event.orderId };
  }

  return { status: "rejected", reason: approval.reason };
});
```

The external system completes the callback using the Lambda API:

```typescript
// External system (e.g., approval web UI backend)
import { LambdaClient, SendDurableExecutionCallbackSuccessCommand } from "@aws-sdk/client-lambda";

const client = new LambdaClient({});

await client.send(new SendDurableExecutionCallbackSuccessCommand({
  DurableExecutionArn: executionArn,
  CallbackId: callbackId,
  Result: JSON.stringify({ approved: true }),
}));
```

### Key Considerations

- The `timeout` configuration prevents the workflow from waiting indefinitely. A `CallbackTimeoutError` is thrown if the timeout expires.
- The submitter function runs inside the durable execution. If it fails, a `CallbackSubmitterError` is thrown and the callback is not created.
- Use `heartbeatTimeout` if the external system should periodically signal that it is still processing. Missed heartbeats trigger a timeout.
- The callback result must be a JSON string when sent via the API. The SDK deserializes it using the configured `serdes` (defaults to JSON parse).
- Multiple callbacks can be active simultaneously — each gets a unique `callbackId`.

## Saga Pattern (Compensating Transactions)

The saga pattern coordinates a sequence of steps where each step has a corresponding compensation (undo) action. If any step fails, the compensations for all previously completed steps run in reverse order to restore consistency. This is the distributed systems alternative to a database transaction.

### When to Use

- Multi-service transactions (booking flights + hotels + car rentals)
- Order fulfillment with inventory reservation, payment, and shipping
- Any workflow where partial completion leaves the system in an inconsistent state
- Operations spanning multiple external services that lack a shared transaction coordinator

### Code Example

```typescript
import { withDurableExecution, DurableContext } from "@aws/durable-execution-sdk-js";

interface BookingEvent {
  flightId: string;
  hotelId: string;
  paymentMethod: string;
  total: number;
}

export const handler = withDurableExecution(async (event: BookingEvent, context: DurableContext) => {
  const compensations: Array<{ name: string; fn: () => Promise<void> }> = [];

  try {
    // Step 1: Reserve flight
    const flightReservation = await context.step("reserve-flight", async () => {
      return await flightService.reserve(event.flightId);
    });
    compensations.push({
      name: "cancel-flight",
      fn: () => flightService.cancel(flightReservation.confirmationId),
    });

    // Step 2: Reserve hotel
    const hotelReservation = await context.step("reserve-hotel", async () => {
      return await hotelService.reserve(event.hotelId);
    });
    compensations.push({
      name: "cancel-hotel",
      fn: () => hotelService.cancel(hotelReservation.confirmationId),
    });

    // Step 3: Charge payment
    const payment = await context.step("charge-payment", async () => {
      return await paymentService.charge(event.paymentMethod, event.total);
    });
    compensations.push({
      name: "refund-payment",
      fn: () => paymentService.refund(payment.transactionId),
    });

    return {
      status: "confirmed",
      flight: flightReservation.confirmationId,
      hotel: hotelReservation.confirmationId,
      payment: payment.transactionId,
    };
  } catch (error) {
    // Run compensations in reverse order
    for (const compensation of compensations.reverse()) {
      await context.step(compensation.name, async () => {
        await compensation.fn();
      });
    }
    throw error;
  }
});
```

### Key Considerations

- Each compensation runs inside its own `context.step`, so it is checkpointed independently. If the Lambda invocation is interrupted during compensation, replay skips already-completed compensations.
- The `compensations` array is rebuilt on replay because it lives outside steps. This works correctly because the same steps complete (or fail) in the same order during replay, producing the same compensation list.
- Compensation functions should be idempotent — a compensation might run more than once if the checkpoint after it fails.
- Consider adding retry strategies to compensation steps, since the external services being undone may also experience transient failures.
- For more complex scenarios, consider nesting sagas using `runInChildContext` to group related steps and their compensations.

## Fan-Out/Fan-In

Fan-out/fan-in distributes work across multiple parallel executions and then aggregates the results. The SDK provides two approaches: `context.invoke` with `context.promise.all` for distributing work to separate Lambda functions, and `context.map` for processing items within a single execution.

### When to Use

- Processing a batch of items where each item is independent
- Distributing work across multiple Lambda functions for isolation or different runtime requirements
- Aggregating results from multiple data sources or services
- Workloads that benefit from parallel execution to reduce total latency

### Code Example — Using `map`

The simpler approach for processing an array of items within a single durable execution:

```typescript
import { withDurableExecution, DurableContext } from "@aws/durable-execution-sdk-js";

interface BatchEvent {
  orderIds: string[];
}

export const handler = withDurableExecution(async (event: BatchEvent, context: DurableContext) => {
  // Fan out: process all orders in parallel
  const results = await context.map(
    "process-orders",
    event.orderIds,
    async (ctx, orderId, index) => {
      return await ctx.step(`process-${index}`, async () => {
        return await processOrder(orderId);
      });
    },
    { maxConcurrency: 10 },
  );

  // Fan in: aggregate results
  results.throwIfError();
  const processedOrders = results.getResults();

  // Summarize
  await context.step("send-summary", async () => {
    await sendBatchSummary(processedOrders);
  });

  return { processed: processedOrders.length };
});
```

### Code Example — Using `invoke` with `promise.all`

For distributing work to separate Lambda functions:

```typescript
import { withDurableExecution, DurableContext } from "@aws/durable-execution-sdk-js";

const PROCESSOR_ARN = process.env.PROCESSOR_FUNCTION_ARN!;

interface DistributedEvent {
  regions: string[];
}

export const handler = withDurableExecution(async (event: DistributedEvent, context: DurableContext) => {
  // Fan out: invoke a separate function for each region
  const invokePromises = event.regions.map((region) =>
    context.invoke(`process-${region}`, PROCESSOR_ARN, { region }),
  );

  // Fan in: wait for all invocations to complete
  const results = await context.promise.all("gather-results", invokePromises);

  // Aggregate
  const summary = await context.step("aggregate", async () => {
    return aggregateRegionResults(results);
  });

  return summary;
});
```

### Key Considerations

- Prefer `context.map` when items can be processed within a single execution. It provides concurrency control, completion policies, and checkpointing per item.
- Use `context.invoke` + `promise.all` when work needs to run in separate Lambda functions — for example, when each branch has different memory requirements or timeout needs.
- `promise.all` starts all invocations immediately with no concurrency control. For controlled concurrency with invocations, use `map` or `parallel` instead.
- `map` returns a `BatchResult` with rich error handling (`throwIfError`, `getErrors`, `failed`). `promise.all` rejects on the first failure, similar to native `Promise.all`.
- Each item in a `map` runs in its own child context with isolated step counters, ensuring deterministic replay regardless of execution order.

## Polling Pattern

The polling pattern repeatedly checks an external system until a condition is met, with configurable delays between checks. The SDK's `waitForCondition` combined with `createWaitStrategy` provides a declarative way to implement polling with exponential backoff, jitter, and maximum attempt limits.

### When to Use

- Waiting for an asynchronous job to complete (ML training, data pipeline, deployment)
- Monitoring an external resource until it reaches a desired state
- Polling an API that does not support webhooks or callbacks
- Any scenario where you need to periodically check and wait

### Code Example

```typescript
import {
  withDurableExecution,
  DurableContext,
  createWaitStrategy,
  JitterStrategy,
} from "@aws/durable-execution-sdk-js";

interface DeploymentEvent {
  deploymentId: string;
  environment: string;
}

interface DeploymentState {
  deploymentId: string;
  status: string;
  progress: number;
}

export const handler = withDurableExecution(async (event: DeploymentEvent, context: DurableContext) => {
  // Start the deployment
  await context.step("start-deployment", async () => {
    return await deploymentService.start(event.deploymentId, event.environment);
  });

  // Poll until the deployment completes
  const finalState = await context.waitForCondition<DeploymentState>(
    "wait-for-deployment",
    async (currentState) => {
      const status = await deploymentService.getStatus(currentState.deploymentId);
      return {
        deploymentId: currentState.deploymentId,
        status: status.state,
        progress: status.percentComplete,
      };
    },
    {
      initialState: {
        deploymentId: event.deploymentId,
        status: "IN_PROGRESS",
        progress: 0,
      },
      waitStrategy: createWaitStrategy<DeploymentState>({
        shouldContinuePolling: (state) =>
          state.status !== "COMPLETED" && state.status !== "FAILED",
        initialDelay: { seconds: 10 },
        maxDelay: { minutes: 2 },
        backoffRate: 1.5,
        maxAttempts: 60,
        jitter: JitterStrategy.FULL,
      }),
    },
  );

  if (finalState.status === "FAILED") {
    throw new Error(`Deployment ${event.deploymentId} failed`);
  }

  return { status: "deployed", deploymentId: event.deploymentId };
});
```

### `createWaitStrategy` Configuration

The `createWaitStrategy` factory produces a wait strategy function compatible with `waitForCondition`. It handles exponential backoff, jitter, and attempt limits:

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `shouldContinuePolling` | `(state: T) => boolean` | — (required) | Returns `true` to keep polling, `false` to stop |
| `maxAttempts` | `number` | `60` | Maximum number of polling attempts |
| `initialDelay` | `Duration` | `{ seconds: 5 }` | Delay before the first retry |
| `maxDelay` | `Duration` | `{ seconds: 300 }` | Maximum delay between retries |
| `backoffRate` | `number` | `1.5` | Multiplier for exponential backoff |
| `jitter` | `JitterStrategy` | `FULL` | Jitter strategy (`NONE`, `FULL`, `HALF`) |

Source: [`wait-strategy-config.ts`](../packages/aws-durable-execution-sdk-js/src/utils/wait-strategy/wait-strategy-config.ts)

### Key Considerations

- Each polling check runs as a step inside `waitForCondition`, so it is checkpointed. If the Lambda invocation is interrupted between checks, replay skips completed checks and resumes from the last one.
- The wait between checks uses `context.wait` internally, so the function suspends without consuming compute during delays.
- `createWaitStrategy` throws an error when `maxAttempts` is exceeded. Catch `WaitForConditionError` to handle this case.
- You can also write a custom wait strategy function directly instead of using `createWaitStrategy`:

```typescript
waitStrategy: (state: DeploymentState, attempt: number) => {
  if (state.status === "COMPLETED" || state.status === "FAILED") {
    return { shouldContinue: false };
  }
  if (attempt > 100) {
    throw new Error("Polling timeout exceeded");
  }
  return { shouldContinue: true, delay: { seconds: Math.min(attempt * 5, 120) } };
},
```

## Concurrent Processing with Completion Policies

This pattern processes a collection of items concurrently with fine-grained control over how many items run simultaneously and when the batch should be considered complete. The `completionConfig` option lets you define success thresholds and failure tolerances, enabling early termination when enough items succeed or too many fail.

### When to Use

- Processing large batches where some failures are acceptable
- Scenarios where you need results from a minimum number of items (e.g., quorum reads)
- Rate-limited APIs where you need to control concurrent request volume
- Workloads where early termination saves time and cost

### Code Example

```typescript
import { withDurableExecution, DurableContext } from "@aws/durable-execution-sdk-js";

interface NotificationEvent {
  userIds: string[];
  message: string;
}

export const handler = withDurableExecution(async (event: NotificationEvent, context: DurableContext) => {
  const results = await context.map(
    "send-notifications",
    event.userIds,
    async (ctx, userId, index) => {
      return await ctx.step(`notify-${index}`, async () => {
        const preferences = await getUserPreferences(userId);
        await sendNotification(userId, event.message, preferences.channel);
        return { userId, sent: true };
      });
    },
    {
      maxConcurrency: 5,
      completionConfig: {
        toleratedFailurePercentage: 10, // Allow up to 10% failures
      },
    },
  );

  // Inspect results
  const succeeded = results.getResults();
  const errors = results.getErrors();

  context.logger.info("Notification batch complete", {
    total: event.userIds.length,
    succeeded: succeeded.length,
    failed: errors.length,
    completionReason: results.completionReason,
  });

  return {
    sent: succeeded.length,
    failed: errors.length,
    completionReason: results.completionReason,
  };
});
```

### `completionConfig` Options

| Parameter | Type | Description |
|-----------|------|-------------|
| `minSuccessful` | `number` | Stop processing once this many items succeed. Remaining items are cancelled. |
| `toleratedFailureCount` | `number` | Maximum number of failures allowed. Exceeding this stops the batch. |
| `toleratedFailurePercentage` | `number` | Maximum failure percentage (0–100). Exceeding this stops the batch. |

The `completionReason` on `BatchResult` indicates why the batch finished:
- `"ALL_COMPLETED"` — every item ran to completion
- `"MIN_SUCCESSFUL_REACHED"` — the `minSuccessful` threshold was met
- `"FAILURE_TOLERANCE_EXCEEDED"` — failures exceeded the tolerated count or percentage

### Using `parallel` for Heterogeneous Branches

When branches perform different operations rather than the same operation on different items, use `parallel`:

```typescript
const results = await context.parallel(
  "fetch-all-data",
  [
    { name: "user-profile", func: async (ctx) => ctx.step(async () => fetchUserProfile(userId)) },
    { name: "order-history", func: async (ctx) => ctx.step(async () => fetchOrderHistory(userId)) },
    { name: "recommendations", func: async (ctx) => ctx.step(async () => fetchRecommendations(userId)) },
  ],
  {
    maxConcurrency: 3,
    completionConfig: { minSuccessful: 2 }, // Proceed if at least 2 of 3 succeed
  },
);

const successfulResults = results.getResults();
```

### Key Considerations

- `maxConcurrency` controls how many child contexts execute simultaneously. Without it, all items start at once, which can overwhelm downstream services.
- Each item runs in its own child context. A failure in one item does not affect others (unless `toleratedFailureCount` or `toleratedFailurePercentage` is exceeded).
- When `minSuccessful` is reached, remaining in-progress items are cancelled. Their status in the `BatchResult` will be `STARTED` (not `SUCCEEDED` or `FAILED`).
- Completion checks happen asynchronously. Due to concurrent execution, the actual number of completed items may slightly exceed the configured thresholds by the time the check runs.
- Use `results.failed()` to inspect individual failures. Each failed item's error is a `ChildContextError` wrapping the original error from the child context.

---

[← Previous: Testing](./08-testing.md) | [Next: References →](./10-references.md)
