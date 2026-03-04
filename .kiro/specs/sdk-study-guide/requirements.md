# Requirements Document

## Introduction

This document defines the requirements for a comprehensive developer study guide for the AWS Lambda Durable Functions SDK. The study guide targets developers working on the SDK itself, providing deep understanding of all consumer-facing interfaces, the interaction with Lambda Durable Functions APIs, internal concepts like threading and replay, configuration options, and how consumers structure Lambda functions to use the SDK. The guide is written in Markdown, organized into chapters with a main entry point, and uses relative links for navigation. It is designed to be reusable across SDK language implementations by keeping repository references to a single configurable target.

## Glossary

- **Study_Guide**: The collection of Markdown files forming the developer study guide, located under a `/study_guide` directory in the target repository
- **Entry_Point**: The main Markdown file (`README.md`) that serves as the table of contents and starting page for the Study_Guide
- **Chapter**: An individual Markdown file within the Study_Guide covering a specific topic area
- **SDK**: The AWS Lambda Durable Functions SDK, the software library whose interfaces and behavior the Study_Guide documents
- **Consumer**: A developer who uses the SDK to build durable Lambda functions
- **DurableContext**: The primary interface exposed to Consumers, providing methods for steps, waits, callbacks, invokes, map, parallel, child contexts, and promise combinators
- **Replay_Model**: The execution model where durable functions re-execute from the beginning on resume, skipping previously checkpointed operations
- **Checkpoint**: A persisted record of a completed operation's result, used during replay to avoid re-execution
- **Target_Repository**: The single SDK repository that the Study_Guide references for code links and examples

## Requirements

### Requirement 1: Study Guide Structure and Navigation

**User Story:** As a developer, I want the study guide to be well-organized with a clear chapter structure and navigation, so that I can find and move between topics efficiently.

#### Acceptance Criteria

1. THE Study_Guide SHALL be located in a `/study_guide` directory at the root of the Target_Repository
2. THE Entry_Point SHALL be a `README.md` file at the root of the `/study_guide` directory containing a title, overview paragraph, and a numbered table of contents linking to all Chapters
3. WHEN a reader navigates between Chapters, THE Study_Guide SHALL use relative Markdown links so that navigation works in any standard Markdown viewer
4. THE Entry_Point SHALL list Chapters in a logical learning progression from foundational concepts to advanced topics
5. WHEN a Chapter references another Chapter, THE Chapter SHALL include a relative link to the referenced Chapter
6. THE Study_Guide SHALL include "Previous" and "Next" navigation links at the bottom of each Chapter to enable sequential reading

### Requirement 2: Overview and Conceptual Foundation

**User Story:** As a developer, I want an overview chapter that explains what durable functions are and how the SDK fits into the Lambda ecosystem, so that I have the foundational context before diving into specifics.

#### Acceptance Criteria

1. THE Study_Guide SHALL include an overview Chapter that explains the purpose of durable functions and the problems the SDK solves
2. THE overview Chapter SHALL explain the Replay_Model, including how functions re-execute from the beginning and how completed operations are skipped via Checkpoints
3. THE overview Chapter SHALL include a Mermaid sequence diagram showing the lifecycle of a durable function invocation from first execution through suspension and replay
4. THE overview Chapter SHALL explain the determinism requirement for code outside of steps, with examples of correct and incorrect patterns
5. THE overview Chapter SHALL explain the relationship between the SDK and the Lambda Durable Functions backend APIs

### Requirement 3: Consumer-Facing Interfaces

**User Story:** As a developer, I want a detailed walkthrough of every interface exposed to SDK consumers, so that I understand the full API surface and how each piece fits together.

#### Acceptance Criteria

1. THE Study_Guide SHALL include a Chapter documenting the handler wrapper function (`withDurableExecution`) including its signature, parameters, return type, and how it transforms a consumer handler into a Lambda-compatible handler
2. THE Study_Guide SHALL include a Chapter documenting the DurableContext interface, covering every public method: `step`, `wait`, `invoke`, `runInChildContext`, `waitForCallback`, `createCallback`, `waitForCondition`, `map`, `parallel`, and `configureLogger`
3. WHEN documenting each DurableContext method, THE Chapter SHALL include the method signature, parameter descriptions, return type, configuration options, and at least one code example
4. THE Study_Guide SHALL document the `promise` property on DurableContext, covering `all`, `allSettled`, `any`, and `race` combinators with guidance on when to use them versus `map`/`parallel`
5. THE Study_Guide SHALL document the `DurablePromise` class, explaining its lazy execution model and how it differs from native Promises
6. THE Study_Guide SHALL document the `logger` property on DurableContext, explaining replay-aware logging and mode-aware behavior
7. THE Study_Guide SHALL document the `lambdaContext` property on DurableContext, explaining access to the underlying Lambda context

### Requirement 4: Lambda API Interaction and Request Paths

**User Story:** As a developer, I want to understand how the SDK interacts with the Lambda Durable Functions backend APIs, so that I can reason about the system's behavior end-to-end.

#### Acceptance Criteria

1. THE Study_Guide SHALL include a Chapter explaining the interaction between the SDK and the Lambda Durable Functions APIs (`CheckpointDurableExecution` and `GetDurableExecutionState`)
2. THE Chapter SHALL include a Mermaid sequence diagram showing the request path for a first-time execution: invocation → handler execution → step execution → checkpoint API call → response
3. THE Chapter SHALL include a Mermaid sequence diagram showing the request path for a replay execution: invocation → get execution state → replay completed operations → execute new operations → checkpoint
4. THE Chapter SHALL explain the `DurableExecutionInvocationInput` structure including `DurableExecutionArn`, `CheckpointToken`, and `InitialExecutionState` with its `Operations` array and `NextMarker` pagination
5. THE Chapter SHALL explain the `DurableExecutionInvocationOutput` structure including the three status variants: `SUCCEEDED`, `FAILED`, and `PENDING`
6. THE Chapter SHALL explain the `DurableExecutionClient` interface and how the `DurableExecutionApiClient` implements it
7. THE Chapter SHALL include links to the relevant Lambda API documentation

### Requirement 5: Threading, Concurrency, and Execution Model

**User Story:** As a developer, I want to understand how the SDK handles threading, concurrency, and the execution lifecycle, so that I can reason about concurrent operations and state management.

#### Acceptance Criteria

1. THE Study_Guide SHALL include a Chapter explaining how the SDK manages concurrent operations using child contexts, including why child contexts are necessary for deterministic replay
2. THE Chapter SHALL explain the checkpoint manager's role in queuing and processing checkpoint requests, including the interaction between the handler promise and the termination promise via `Promise.race`
3. THE Chapter SHALL include a Mermaid diagram showing how `map` and `parallel` operations coordinate concurrent child context executions
4. THE Chapter SHALL explain the termination manager and the different termination reasons: checkpoint failure, serdes failure, and context validation error
5. THE Chapter SHALL explain how the `EventEmitter`-based step data communication works between the checkpoint manager and operation handlers
6. THE Chapter SHALL explain the execution context initialization process, including how operation history is loaded and paginated

### Requirement 6: Configuration Reference

**User Story:** As a developer, I want a detailed reference of all configuration options exposed to SDK consumers, so that I can understand how to customize SDK behavior.

#### Acceptance Criteria

1. THE Study_Guide SHALL include a Chapter documenting all consumer-facing configuration interfaces
2. THE Chapter SHALL document `StepConfig` including `retryStrategy`, `semantics` (AtLeastOncePerRetry vs AtMostOncePerRetry), and `serdes`
3. THE Chapter SHALL document `RetryStrategyConfig` including `maxAttempts`, `initialDelay`, `maxDelay`, `backoffRate`, `jitter` (NONE, FULL, HALF), `retryableErrors`, and `retryableErrorTypes`, with an explanation of how the `createRetryStrategy` factory function works
4. THE Chapter SHALL document the `retryPresets` object including the `default` and `noRetry` presets with their specific parameter values
5. THE Chapter SHALL document `WaitForCallbackConfig` and `CreateCallbackConfig` including `timeout`, `heartbeatTimeout`, `retryStrategy`, and `serdes`
6. THE Chapter SHALL document `WaitForConditionConfig` including `waitStrategy`, `initialState`, and `serdes`, and explain how `createWaitStrategy` and `WaitStrategyConfig` work
7. THE Chapter SHALL document `MapConfig` and `ParallelConfig` including `maxConcurrency`, `completionConfig` (with `minSuccessful`, `toleratedFailureCount`, `toleratedFailurePercentage`), `serdes`, and `itemSerdes`
8. THE Chapter SHALL document `ChildConfig` including `serdes`, `subType`, `summaryGenerator`, and `errorMapper`
9. THE Chapter SHALL document `InvokeConfig` including `payloadSerdes` and `resultSerdes`
10. THE Chapter SHALL document `DurableExecutionConfig` including the optional `client` parameter for custom Lambda client injection
11. THE Chapter SHALL document `LoggerConfig` including `customLogger` and `modeAware` options
12. THE Chapter SHALL document the `Serdes` interface including `serialize` and `deserialize` methods, the `SerdesContext` parameter, and the built-in helpers: `defaultSerdes`, `createClassSerdes`, and `createClassSerdesWithDates`
13. THE Chapter SHALL document the `Duration` type and its supported units: `days`, `hours`, `minutes`, `seconds`

### Requirement 7: Lambda Function Structure

**User Story:** As a developer, I want to understand how a consumer structures a Lambda function to use the SDK, so that I can see the complete picture from project setup to deployment.

#### Acceptance Criteria

1. THE Study_Guide SHALL include a Chapter showing how a Consumer structures a Lambda function to use the SDK, from import statements through handler definition to export
2. THE Chapter SHALL explain the `DurableExecutionHandler` type signature including its generic type parameters: `TEvent`, `TResult`, and `TLogger`
3. THE Chapter SHALL explain the `DurableLambdaHandler` type and how `withDurableExecution` transforms a `DurableExecutionHandler` into a `DurableLambdaHandler`
4. THE Chapter SHALL document the required IAM permissions (`AWSLambdaBasicDurableExecutionRolePolicy`) and additional permissions needed for durable invokes and callbacks
5. THE Chapter SHALL explain the requirement for qualified ARNs (version number, alias, or `$LATEST`) when invoking durable functions
6. THE Chapter SHALL include infrastructure-as-code examples for deploying durable functions using CloudFormation, CDK, and SAM, showing the `DurableConfig` settings

### Requirement 8: Error Handling and Error Types

**User Story:** As a developer, I want to understand the SDK's error hierarchy and error handling patterns, so that I can properly handle failures in durable workflows.

#### Acceptance Criteria

1. THE Study_Guide SHALL include a Chapter documenting the error type hierarchy: `DurableOperationError` as the base, with `StepError`, `CallbackError`, `CallbackTimeoutError`, `CallbackSubmitterError`, `InvokeError`, `ChildContextError`, `WaitForConditionError`, and `StepInterruptedError`
2. THE Chapter SHALL explain how errors are serialized and deserialized across checkpoints, including the `ErrorObject` structure with `errorType`, `errorMessage`, `errorData`, and `stackTrace`
3. THE Chapter SHALL explain the difference between recoverable errors (retried automatically) and unrecoverable errors (terminate execution)
4. THE Chapter SHALL document the saga pattern for compensating transactions with a code example
5. THE Chapter SHALL explain how `BatchResult` exposes errors from `map` and `parallel` operations, including `throwIfError()`, `getErrors()`, `hasFailure`, and the `failed()` method

### Requirement 9: Testing with the SDK

**User Story:** As a developer, I want to understand how to test durable functions using the testing SDK, so that I can write reliable tests for durable workflows.

#### Acceptance Criteria

1. THE Study_Guide SHALL include a Chapter documenting the testing SDK (`@aws/durable-execution-sdk-js-testing` or equivalent package for the Target_Repository)
2. THE Chapter SHALL document the `LocalDurableTestRunner` including `setupTestEnvironment`, `teardownTestEnvironment`, `run`, `getOperation`, `getOperationByIndex`, `getOperationByNameAndIndex`, `reset`, `skipTime`, `fakeClock`, `registerFunction`, and `registerDurableFunction`
3. THE Chapter SHALL document the `CloudDurableTestRunner` including its constructor parameters, `run`, `getOperation`, and `reset` methods
4. THE Chapter SHALL document the `TestResult` interface including `getStatus`, `getResult`, `getError`, `getOperations`, `getInvocations`, `getHistoryEvents`, and `print`
5. THE Chapter SHALL document the `DurableOperation` interface including methods for inspecting operation type, status, name, details (step, wait, callback, chained invoke, context), child operations, and callback interaction methods (`sendCallbackSuccess`, `sendCallbackFailure`, `sendCallbackHeartbeat`)
6. THE Chapter SHALL include complete test examples for common patterns: basic step testing, callback testing, wait testing, and parallel/map testing
7. THE Chapter SHALL explain the difference between local testing (in-process, fast, uses fake clock) and cloud testing (deployed Lambda, real service, uses polling)

### Requirement 10: Common Patterns and Use Cases

**User Story:** As a developer, I want a chapter covering common durable function patterns, so that I can learn from proven approaches to typical problems.

#### Acceptance Criteria

1. THE Study_Guide SHALL include a Chapter documenting common durable function patterns with complete code examples
2. THE Chapter SHALL include a GenAI agentic loop pattern showing how to use steps in a while loop with model invocation and tool execution
3. THE Chapter SHALL include a human-in-the-loop approval pattern using `waitForCallback`
4. THE Chapter SHALL include a saga pattern for compensating transactions using try/catch with a compensations array
5. THE Chapter SHALL include a fan-out/fan-in pattern using `invoke` and `promise.all` or `map`
6. THE Chapter SHALL include a polling pattern using `waitForCondition` with `createWaitStrategy`
7. THE Chapter SHALL include a concurrent processing pattern using `map` with `maxConcurrency` and `completionConfig`

### Requirement 11: Code and Documentation Links

**User Story:** As a developer, I want the study guide to link to relevant source code and official documentation, so that I can dive deeper into specific areas.

#### Acceptance Criteria

1. WHEN the Study_Guide references a specific SDK interface or class, THE Study_Guide SHALL include a relative link to the source file in the Target_Repository where that interface or class is defined
2. WHEN the Study_Guide references a Lambda backend API, THE Study_Guide SHALL include a link to the corresponding AWS Lambda API documentation
3. THE Entry_Point SHALL include a "References" section with links to the official AWS Lambda Durable Functions documentation, the SDK README, and the API reference documentation
4. THE Study_Guide SHALL keep all Target_Repository references relative to the repository root so that the guide works when the repository is cloned locally

### Requirement 12: Reusability Across SDK Implementations

**User Story:** As a developer, I want the study guide structure to be reusable across different language SDK implementations, so that the same organizational approach can be applied to other repositories.

#### Acceptance Criteria

1. THE Study_Guide SHALL use language-agnostic concept explanations in prose sections, with language-specific details confined to code examples and interface signatures
2. THE Study_Guide SHALL reference the Target_Repository as a single configurable location, avoiding hardcoded references to multiple repositories
3. WHEN the Study_Guide includes code examples, THE Study_Guide SHALL use the Target_Repository's language and idioms while keeping the surrounding explanation applicable to other SDK implementations
