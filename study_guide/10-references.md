# References

This chapter consolidates all external documentation, source code, and API reference links used throughout the study guide. Use it as a quick-access directory when you need to jump to a specific resource.

## AWS Documentation

- [AWS Lambda Durable Functions Guide](https://docs.aws.amazon.com/lambda/latest/dg/durable-functions.html) — Primary documentation for durable functions on AWS Lambda
- [Invoking Durable Functions](https://docs.aws.amazon.com/lambda/latest/dg/durable-invoking.html) — Invocation methods, qualified ARN requirements, synchronous and asynchronous patterns
- [Deploy with Infrastructure as Code](https://docs.aws.amazon.com/lambda/latest/dg/durable-getting-started-iac.html) — CloudFormation, CDK, and SAM deployment guides
- [Lambda API Reference](https://docs.aws.amazon.com/lambda/latest/api/) — Full API reference for AWS Lambda, including durable execution endpoints
- [AWSLambdaBasicDurableExecutionRolePolicy](https://docs.aws.amazon.com/aws-managed-policy/latest/reference/AWSLambdaBasicDurableExecutionRolePolicy.html) — IAM managed policy for durable function permissions

## SDK Source Code

### Main SDK (`@aws/durable-execution-sdk-js`)

- [SDK README](../packages/aws-durable-execution-sdk-js/README.md) — Getting started, installation, and usage overview
- [Concepts Document](../packages/aws-durable-execution-sdk-js/src/documents/CONCEPTS.md) — In-depth explanation of the durable execution model and SDK design

#### Core Entry Points

- [`with-durable-execution.ts`](../packages/aws-durable-execution-sdk-js/src/with-durable-execution.ts) — Handler wrapper implementation
- [`index.ts`](../packages/aws-durable-execution-sdk-js/src/index.ts) — Public API exports

#### Type Definitions

- [`durable-context.ts`](../packages/aws-durable-execution-sdk-js/src/types/durable-context.ts) — `DurableContext` interface
- [`durable-execution.ts`](../packages/aws-durable-execution-sdk-js/src/types/durable-execution.ts) — `DurableExecutionHandler`, `DurableLambdaHandler`, `DurableExecutionConfig`
- [`core.ts`](../packages/aws-durable-execution-sdk-js/src/types/core.ts) — Core types including `Duration`, `InvocationStatus`, `OperationSubType`
- [`step.ts`](../packages/aws-durable-execution-sdk-js/src/types/step.ts) — `StepConfig`, `StepFunc`, `StepSemantics`
- [`callback.ts`](../packages/aws-durable-execution-sdk-js/src/types/callback.ts) — `WaitForCallbackConfig`, `CreateCallbackConfig`
- [`wait-condition.ts`](../packages/aws-durable-execution-sdk-js/src/types/wait-condition.ts) — `WaitForConditionConfig`, `WaitStrategyConfig`
- [`batch.ts`](../packages/aws-durable-execution-sdk-js/src/types/batch.ts) — `BatchResult`, `BatchItem`, `CompletionConfig`
- [`child-context.ts`](../packages/aws-durable-execution-sdk-js/src/types/child-context.ts) — `ChildConfig`, `MapConfig`, `ParallelConfig`
- [`invoke.ts`](../packages/aws-durable-execution-sdk-js/src/types/invoke.ts) — `InvokeConfig`
- [`durable-promise.ts`](../packages/aws-durable-execution-sdk-js/src/types/durable-promise.ts) — `DurablePromise` class
- [`durable-logger.ts`](../packages/aws-durable-execution-sdk-js/src/types/durable-logger.ts) — `DurableLogger`, `DurableLogData`
- [`logger.ts`](../packages/aws-durable-execution-sdk-js/src/types/logger.ts) — `LoggerConfig`

#### Error Types

- [`errors/durable-error/`](../packages/aws-durable-execution-sdk-js/src/errors/durable-error/) — `DurableOperationError` base class
- [`errors/step-errors/`](../packages/aws-durable-execution-sdk-js/src/errors/step-errors/) — `StepError`, `StepInterruptedError`
- [`errors/callback-error/`](../packages/aws-durable-execution-sdk-js/src/errors/callback-error/) — `CallbackError`, `CallbackTimeoutError`, `CallbackSubmitterError`

#### Utilities

- [`utils/serdes/`](../packages/aws-durable-execution-sdk-js/src/utils/serdes/) — `Serdes` interface, `defaultSerdes`, `createClassSerdes`, `createClassSerdesWithDates`
- [`utils/retry/`](../packages/aws-durable-execution-sdk-js/src/utils/retry/) — `createRetryStrategy`, `RetryStrategyConfig`, `retryPresets`
- [`utils/wait-strategy/`](../packages/aws-durable-execution-sdk-js/src/utils/wait-strategy/) — `createWaitStrategy`, `WaitStrategyConfig`
- [`utils/error-object/`](../packages/aws-durable-execution-sdk-js/src/utils/error-object/) — `ErrorObject` serialization
- [`utils/checkpoint/`](../packages/aws-durable-execution-sdk-js/src/utils/checkpoint/) — Checkpoint manager internals
- [`utils/duration/`](../packages/aws-durable-execution-sdk-js/src/utils/duration/) — `Duration` utilities

#### API Client

- [`durable-execution-api-client.ts`](../packages/aws-durable-execution-sdk-js/src/durable-execution-api-client/durable-execution-api-client.ts) — `DurableExecutionApiClient` implementation

#### Operation Handlers

- [`handlers/step-handler/`](../packages/aws-durable-execution-sdk-js/src/handlers/step-handler/) — Step execution and retry logic
- [`handlers/wait-handler/`](../packages/aws-durable-execution-sdk-js/src/handlers/wait-handler/) — Wait operation handling
- [`handlers/callback-handler/`](../packages/aws-durable-execution-sdk-js/src/handlers/callback-handler/) — Callback and waitForCallback handling
- [`handlers/invoke-handler/`](../packages/aws-durable-execution-sdk-js/src/handlers/invoke-handler/) — Durable invoke handling
- [`handlers/map-handler/`](../packages/aws-durable-execution-sdk-js/src/handlers/map-handler/) — Map operation handling
- [`handlers/parallel-handler/`](../packages/aws-durable-execution-sdk-js/src/handlers/parallel-handler/) — Parallel operation handling
- [`handlers/wait-for-condition-handler/`](../packages/aws-durable-execution-sdk-js/src/handlers/wait-for-condition-handler/) — Polling/condition handling
- [`handlers/wait-for-callback-handler/`](../packages/aws-durable-execution-sdk-js/src/handlers/wait-for-callback-handler/) — WaitForCallback handling
- [`handlers/run-in-child-context-handler/`](../packages/aws-durable-execution-sdk-js/src/handlers/run-in-child-context-handler/) — Child context handling
- [`handlers/concurrent-execution-handler/`](../packages/aws-durable-execution-sdk-js/src/handlers/concurrent-execution-handler/) — Shared concurrency logic for map/parallel
- [`handlers/promise-handler/`](../packages/aws-durable-execution-sdk-js/src/handlers/promise-handler/) — Promise combinator handling

#### Termination Manager

- [`termination-manager/termination-manager.ts`](../packages/aws-durable-execution-sdk-js/src/termination-manager/termination-manager.ts) — Termination manager and `TerminationReason`

### Testing SDK (`@aws/durable-execution-sdk-js-testing`)

- [Testing SDK README](../packages/aws-durable-execution-sdk-js-testing/README.md) — Setup, usage, and API overview for the testing package

## API Reference

Generated API documentation for both packages:

- [API Reference Index](../docs/api-reference/index.md) — Entry point for all generated API docs
- [`DurableContext`](../docs/api-reference/durable-execution-sdk-js.durablecontext.md) — Full interface reference
- [`withDurableExecution`](../docs/api-reference/durable-execution-sdk-js.withdurableexecution.md) — Handler wrapper reference
- [`DurableExecutionHandler`](../docs/api-reference/durable-execution-sdk-js.durableexecutionhandler.md) — Handler type reference
- [`DurablePromise`](../docs/api-reference/durable-execution-sdk-js.durablepromise.md) — Promise class reference
- [`BatchResult`](../docs/api-reference/durable-execution-sdk-js.batchresult.md) — Batch result reference
- [`DurableOperationError`](../docs/api-reference/durable-execution-sdk-js.durableoperationerror.md) — Base error reference
- [`StepConfig`](../docs/api-reference/durable-execution-sdk-js.stepconfig.md) — Step configuration reference
- [`RetryStrategyConfig`](../docs/api-reference/durable-execution-sdk-js.retrystrategyconfig.md) — Retry configuration reference
- [`Serdes`](../docs/api-reference/durable-execution-sdk-js.serdes.md) — Serialization interface reference
- [`Duration`](../docs/api-reference/durable-execution-sdk-js.duration.md) — Duration type reference
- [`LocalDurableTestRunner`](../docs/api-reference/durable-execution-sdk-js-testing.localdurabletestrunner.md) — Local test runner reference
- [`CloudDurableTestRunner`](../docs/api-reference/durable-execution-sdk-js-testing.clouddurabletestrunner.md) — Cloud test runner reference
- [`TestResult`](../docs/api-reference/durable-execution-sdk-js-testing.testresult.md) — Test result reference
- [`DurableOperation`](../docs/api-reference/durable-execution-sdk-js-testing.durableoperation.md) — Operation inspection reference

## Related Guides

### Study Guide Chapters

1. [Overview and Conceptual Foundation](./01-overview.md)
2. [Lambda Function Structure](./02-lambda-function-structure.md)
3. [Consumer Interfaces](./03-consumer-interfaces.md)
4. [API Interaction and Request Paths](./04-api-interaction.md)
5. [Threading, Concurrency, and Execution Model](./05-threading-and-concurrency.md)
6. [Configuration Reference](./06-configuration-reference.md)
7. [Error Handling](./07-error-handling.md)
8. [Testing with the SDK](./08-testing.md)
9. [Common Patterns and Use Cases](./09-common-patterns.md)

---

[← Previous: Common Patterns](./09-common-patterns.md)
