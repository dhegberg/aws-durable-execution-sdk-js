# AWS Lambda Durable Functions SDK — Developer Study Guide

## Overview

This study guide provides a comprehensive walkthrough of the AWS Lambda Durable Functions SDK for developers working on or with the SDK. It covers every consumer-facing interface, the interaction between the SDK and the Lambda Durable Functions backend APIs, internal concepts like threading and replay, configuration options, error handling, testing strategies, and common workflow patterns.

The guide is organized as a progressive learning path. Early chapters establish foundational concepts — what durable functions are, how the replay model works, and how a Lambda function is structured to use the SDK. Later chapters dive into the full API surface, concurrency internals, configuration knobs, error hierarchies, and testing tools. The final chapters cover practical patterns you can apply directly to real workloads.

Whether you are onboarding to the SDK codebase, building durable workflows, or looking for a reference on a specific interface, this guide is designed to get you there efficiently.

## Prerequisites

- Familiarity with AWS Lambda (invocation model, execution environment, IAM roles)
- Understanding of async/await patterns and Promises
- Basic knowledge of TypeScript/JavaScript (the target language for this repository)
- A cloned copy of the SDK repository for following source code links

## Table of Contents

1. [Overview and Conceptual Foundation](./01-overview.md)
2. [Lambda Function Structure](./02-lambda-function-structure.md)
3. [Consumer Interfaces](./03-consumer-interfaces.md)
4. [API Interaction and Request Paths](./04-api-interaction.md)
5. [Threading, Concurrency, and Execution Model](./05-threading-and-concurrency.md)
6. [Configuration Reference](./06-configuration-reference.md)
7. [Error Handling](./07-error-handling.md)
8. [Testing with the SDK](./08-testing.md)
9. [Common Patterns and Use Cases](./09-common-patterns.md)
10. [References](./10-references.md)

## References

- [AWS Lambda Durable Functions Guide](https://docs.aws.amazon.com/lambda/latest/dg/durable-functions.html)
- [SDK README](../packages/aws-durable-execution-sdk-js/README.md)
- [API Reference](../docs/api-reference/index.md)
- [Concepts Document](../packages/aws-durable-execution-sdk-js/src/documents/CONCEPTS.md)
