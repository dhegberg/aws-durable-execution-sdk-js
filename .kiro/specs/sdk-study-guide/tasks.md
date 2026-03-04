# Implementation Plan: SDK Study Guide

## Overview

Create a comprehensive developer study guide as Markdown files in `/study_guide`. Each task creates one or more chapter files, building incrementally from the entry point through foundational concepts to advanced topics. All chapters use relative links for navigation and source code references.

## Tasks

- [x] 1. Create study guide directory and entry point
  - [x] 1.1 Create `study_guide/README.md` with title, overview, prerequisites, numbered table of contents linking to all 10 chapters, and references section with links to AWS docs, SDK README, API reference, and CONCEPTS.md
    - Use relative links for all internal references (e.g., `../packages/aws-durable-execution-sdk-js/README.md`)
    - Include links to: AWS Lambda Durable Functions Guide, SDK README, API Reference index, CONCEPTS.md
    - _Requirements: 1.1, 1.2, 1.4, 11.3, 11.4_

- [x] 2. Write foundational chapters
  - [x] 2.1 Create `study_guide/01-overview.md` — Overview and Conceptual Foundation
    - Explain what durable functions are and the problems they solve
    - Explain the Replay Model with step-by-step walkthrough
    - Include Mermaid sequence diagram showing first execution → checkpoint → suspension → replay lifecycle
    - Explain determinism requirement with correct/incorrect TypeScript code examples
    - Explain SDK-to-Lambda API relationship at high level
    - Add Previous (README.md) / Next (02) navigation footer
    - _Requirements: 2.1, 2.2, 2.3, 2.4, 2.5_

  - [x] 2.2 Create `study_guide/02-lambda-function-structure.md` — Lambda Function Structure
    - Show complete file structure of a durable Lambda function (imports → handler → export)
    - Document `DurableExecutionHandler<TEvent, TResult, TLogger>` type parameters
    - Document `DurableLambdaHandler` and `withDurableExecution` transformation
    - Document `DurableExecutionConfig` with optional custom Lambda client
    - Document IAM permissions: `AWSLambdaBasicDurableExecutionRolePolicy` plus invoke/callback permissions
    - Explain qualified ARN requirement with valid/invalid examples
    - Include IaC examples for CloudFormation, CDK, and SAM with `DurableConfig`
    - Add Previous/Next navigation footer
    - _Requirements: 7.1, 7.2, 7.3, 7.4, 7.5, 7.6_

- [x] 3. Write core interface chapter
  - [x] 3.1 Create `study_guide/03-consumer-interfaces.md` — Consumer Interfaces
    - Document `withDurableExecution(handler, config?)` wrapper
    - Document every DurableContext method with signature, parameters, return type, config options, and TypeScript code example: `step`, `wait`, `invoke`, `runInChildContext`, `waitForCallback`, `createCallback`, `waitForCondition`, `map`, `parallel`, `configureLogger`
    - Document `context.promise` property: `all`, `allSettled`, `any`, `race` with guidance on when to use vs map/parallel
    - Document `DurablePromise` class: lazy execution, `isExecuted`, difference from native Promise
    - Document `context.logger` — replay-aware, mode-aware logging
    - Document `context.lambdaContext` — access to underlying Lambda context
    - Include relative links to source files for each interface (e.g., `../packages/aws-durable-execution-sdk-js/src/types/durable-context.ts`)
    - Add Previous/Next navigation footer
    - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5, 3.6, 3.7, 11.1_

- [x] 4. Write API interaction and concurrency chapters
  - [x] 4.1 Create `study_guide/04-api-interaction.md` — API Interaction and Request Paths
    - Document `DurableExecutionClient` interface: `getExecutionState` and `checkpoint`
    - Document `DurableExecutionApiClient` implementation
    - Document `DurableExecutionInvocationInput`: `DurableExecutionArn`, `CheckpointToken`, `InitialExecutionState`
    - Document `DurableExecutionInvocationOutput`: SUCCEEDED, FAILED, PENDING variants
    - Document `InvocationStatus` and `OperationSubType` enums with all values
    - Include Mermaid sequence diagrams for first execution and replay request paths
    - Include links to Lambda API documentation (docs.aws.amazon.com)
    - Add Previous/Next navigation footer
    - _Requirements: 4.1, 4.2, 4.3, 4.4, 4.5, 4.6, 4.7_

  - [x] 4.2 Create `study_guide/05-threading-and-concurrency.md` — Threading, Concurrency, and Execution Model
    - Explain child contexts and why they're necessary for deterministic replay (isolated step counters)
    - Explain checkpoint manager: queue-based processing, async checkpoint submission
    - Explain `Promise.race` between handler promise and termination promise in `withDurableExecution`
    - Explain termination manager and `TerminationReason` variants
    - Explain `EventEmitter`-based step data communication
    - Explain execution context initialization and operation history pagination
    - Include Mermaid diagram showing map/parallel concurrent child context coordination
    - Add Previous/Next navigation footer
    - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5, 5.6_

- [x] 5. Checkpoint — Ensure all chapters 01-05 are complete and consistent
  - Verify all inter-chapter links resolve correctly
  - Verify all source code links point to existing files
  - Verify navigation footers are present and correct
  - Ensure all tests pass, ask the user if questions arise.

- [x] 6. Write configuration and error handling chapters
  - [x] 6.1 Create `study_guide/06-configuration-reference.md` — Configuration Reference
    - Document all configuration interfaces in reference format with tables (property | type | default | description)
    - Cover: `StepConfig`, `RetryStrategyConfig`, `retryPresets`, `WaitForCallbackConfig`, `CreateCallbackConfig`, `WaitForConditionConfig`, `WaitStrategyConfig`, `MapConfig`, `ParallelConfig`, `CompletionConfig`, `ChildConfig`, `InvokeConfig`, `DurableExecutionConfig`, `LoggerConfig`
    - Document `Serdes` interface with `defaultSerdes`, `createClassSerdes`, `createClassSerdesWithDates`
    - Document `Duration` type with supported units
    - Include code examples for `createRetryStrategy`, `createWaitStrategy`, custom Serdes
    - Link to source files for each interface
    - Add Previous/Next navigation footer
    - _Requirements: 6.1, 6.2, 6.3, 6.4, 6.5, 6.6, 6.7, 6.8, 6.9, 6.10, 6.11, 6.12, 6.13_

  - [x] 6.2 Create `study_guide/07-error-handling.md` — Error Handling
    - Document error hierarchy with Mermaid class diagram
    - Document each error type: `DurableOperationError`, `StepError`, `CallbackError`, `CallbackTimeoutError`, `CallbackSubmitterError`, `InvokeError`, `ChildContextError`, `WaitForConditionError`, `StepInterruptedError`
    - Explain error serialization across checkpoints: `ErrorObject` structure
    - Explain recoverable vs unrecoverable errors
    - Document `BatchResult` error handling: `throwIfError()`, `getErrors()`, `hasFailure`, `failed()`
    - Include saga pattern code example
    - Add Previous/Next navigation footer
    - _Requirements: 8.1, 8.2, 8.3, 8.4, 8.5_

- [x] 7. Write testing and patterns chapters
  - [x] 7.1 Create `study_guide/08-testing.md` — Testing with the SDK
    - Document `LocalDurableTestRunner`: `setupTestEnvironment`, `teardownTestEnvironment`, `run`, `getOperation`, `getOperationByIndex`, `getOperationByNameAndIndex`, `reset`, `skipTime`, `fakeClock`, `registerFunction`, `registerDurableFunction`
    - Document `CloudDurableTestRunner`: constructor, `run`, `getOperation`, `reset`
    - Document `TestResult`: `getStatus`, `getResult`, `getError`, `getOperations`, `getInvocations`, `getHistoryEvents`, `print`
    - Document `DurableOperation`: type, status, name, details, child operations, callback methods
    - Document enums: `OperationType`, `OperationStatus`, `ExecutionStatus`, `InvocationType`, `WaitingOperationStatus`
    - Include complete test examples for: basic step, callback, wait, parallel/map
    - Explain local vs cloud testing tradeoffs
    - Add Previous/Next navigation footer
    - _Requirements: 9.1, 9.2, 9.3, 9.4, 9.5, 9.6, 9.7_

  - [x] 7.2 Create `study_guide/09-common-patterns.md` — Common Patterns and Use Cases
    - GenAI agentic loop pattern (step in while loop with model invocation and tool execution)
    - Human-in-the-loop approval pattern (waitForCallback)
    - Saga pattern for compensating transactions (try/catch with compensations array)
    - Fan-out/fan-in pattern (invoke + promise.all, or map)
    - Polling pattern (waitForCondition + createWaitStrategy)
    - Concurrent processing pattern (map with maxConcurrency + completionConfig)
    - Each pattern: description, when to use, complete code example, key considerations
    - Add Previous/Next navigation footer
    - _Requirements: 10.1, 10.2, 10.3, 10.4, 10.5, 10.6, 10.7_

  - [x] 7.3 Create `study_guide/10-references.md` — References
    - Consolidated list of all external links organized by category: AWS Documentation, SDK Source, API Reference, Related Guides
    - Add Previous (09) / Next (none — final chapter) navigation footer
    - _Requirements: 11.1, 11.2, 11.3, 11.4_

- [x] 8. Final checkpoint — Validate all files and links
  - Verify all 11 files exist in `study_guide/`
  - Verify README.md TOC links resolve to all chapter files
  - Verify all Previous/Next navigation links are correct and resolve
  - Verify all source code relative links point to existing files
  - Verify all Mermaid diagrams use valid syntax
  - Verify all code blocks specify TypeScript language tag
  - Ensure all tests pass, ask the user if questions arise.
  - _Requirements: 1.3, 1.5, 1.6, 12.1, 12.2, 12.3_

## Notes

- Each task creates complete, self-contained chapter files — no partial content
- All source code links use relative paths from `study_guide/` to the repository root
- The guide targets the `aws-durable-execution-sdk-js` repository but keeps prose language-agnostic for reusability
- Checkpoints at tasks 5 and 8 ensure incremental validation
