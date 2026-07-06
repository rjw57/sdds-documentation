---
title: Implementation of portability jobs
status: proposed
date: 2026-06-06
---

# Using Step Functions or Durable Lambdas to implement portability jobs

## Context and Problem Statement

We require an AWS-friendly solution to support initiating portability jobs using providers'
portability APIs, waiting for those jobs to complete and performing the required ingest steps. This
process involves at least one point where the job must be paused and subsequently resumed once the
provider indicates that data is ready to be fetched.

AWS provides at least two ways of implementing such processes: [step functions][step-function] and
[durable lambdas][durable-lambda]

[step-function]: https://docs.aws.amazon.com/step-functions/latest/dg/welcome.html
[durable-lambda]: https://docs.aws.amazon.com/lambda/latest/dg/durable-functions.html

We want to select an implementation technology for portability jobs.

## Decision Drivers

- We want to use AWS-provided concurrency and rate limit controls when possible rather than writing
  our own semaphore implementation.
- We want to be able to deploy our stack via terraform.
- We want to be able to test our portability jobs locally.
- We want to be able to package our portability jobs in a manner which is amenable to static SBOM
  and vulnerability scanning.
- We want to be able to package our portability jobs in a manner which is amenable to recording
  provenance and attestation in package artefacts so that we have confidence in the source of deployed
  jobs in production.
- We want to be able to implement a "poll for data readiness" flow.
- We want to be able to implement an external "provider has told us the data is ready" flow.
- We want to use a well supported technology.
- We want to have a low-friction way on introducing per-provider concurrency limits.
- We want to use a programming language which the team currently has capability in.
- We want to adopt a "DevOps" mindset where the deployment-side of the implementation is developed
  alongside the pure programming-side.
- We want to ensure that two portability jobs for the same user are not started at once.
- We want to be able to retry a failed portability jobs.
- We want to be able to monitor processing job backlog size, failure rate and total processing job
  duration.
- We want to be able to add monitoring-based alerts in case of errors.

## Considered Options

- AWS Step Functions
- AWS Durable Lambdas

## Decision Outcome

- TBD

### Consequences

- TBD

## Pros and Cons of the Options

The AWS documentation includes [guidance on how to choose technology][aws-guidance]:

[aws-guidance]: https://docs.aws.amazon.com/lambda/latest/dg/durable-step-functions.html

> Use durable functions when:
>
> - Your team prefers standard programming languages and familiar development tools
> - Your application logic is primarily within Lambda functions
> - You want fine-grained control over execution state in code
> - You're building Lambda-centric applications with tight coupling between workflow and business logic
> - You want to iterate quickly without switching between code and visual/JSON designers
>
> Use Step Functions when:
>
> - You need visual workflow representation for cross-team visibility
> - You're orchestrating multiple AWS services and want native integrations without custom SDK code
> - You require zero-maintenance infrastructure (no patching, runtime updates)
> - Non-technical stakeholders need to understand and validate workflow logic

We'll expand on these below.

### Durable functions

- Good, because we have full choice of language.
- Good, because there is full support for [offline testing][durable-testing] outside of AWS.
- Good, because they are packaged and invoked exactly as normal lambdas and so there is strong
  support within the AWS ecosystem for event-based triggering, etc.
- Good, because lambda event source mapping allows processing portability jobs from a SQS queue with
  AWS-enforced concurrency limits.
- Good, because they can be deployed using terraform.
- Good, because they can be packaged as container images which can have automated SBOM,
  vulnerability scanning and provenance attestation. The containers can then be tested as finished
  artifacts using a lambda emulator.
- Good, because they have inbuilt idempotency and restart support so that we can have AWS-level
  assurances that we are not running two copies of a portability job while allowing for restart and
  retry.
- Good, because polling is a [first class concept][wait-for-condition] with support for backoff.
- Good, because waiting for a push event from a provider is a [first class
  concept][wait-for-callback] via callbacks.
- Bad, because they are a relatively new AWS service.
- Bad, because, unlike normal lambdas, durable functions triggered from a SQS queue are not
  automatically retried if they fail.
- Bad, because there is additional complexity ensuring that we work within the bounds of the
  [durable execution replay model][durable-execution].
- Bad, because there is no native visualisation of the flow.

[durable-testing]: https://docs.aws.amazon.com/durable-execution/testing/
[wait-for-condition]: https://docs.aws.amazon.com/durable-execution/sdk-reference/operations/wait-for-condition/
[wait-for-callback]: https://docs.aws.amazon.com/durable-execution/sdk-reference/operations/callback/
[durable-execution]: https://docs.aws.amazon.com/lambda/latest/dg/durable-basic-concepts.html#durable-execution-concept

### Step functions

- Good, because there is a native visual representation of the step function state machine.
- Good, because we can maintain a YAML description in source control of the step function with
  appropriate pre-commit hooks to enforce schema.
- Good, because we can start it from EventBridge.
- Good, because lambda package artifacts can have SBOM, vulnerability scanning and provenance
  attestation.
- Good, because we can express a poll loop natively.
- Good, because it is a mature, well-supported product.
- Bad, because the visual representation lives only in the AWS console.
- Bad, because the YAML description may be a little opaque.
- Bad, because visual depiction is intended to be primary editing experience whereas we are a
  code-first team.
- Bad, because each task needs to be its own lambda and so implementation is split over multiple
  packaging artifacts.
- Bad, because we cannot natively support concurrency limits via SQS queues.
- Bad, because they are tricky to test locally.
- Bad, because native `Wait` state does not natively support exponential backoff and we're limited
  in JSONata as to how flexible we can be with computing wait times.
- Bad, because there is no "wait for external event" state built in so we'd have to build one or
  construct one from `Wait` states.
- Bad, because costing is unclear. E.g. "Express" has 5 minute limit and "Standard" charges by state
  transitions which can make structures more complicated to optimise for cost.

## More Information

- [Slack thread](https://uksdds.slack.com/archives/C08UYP843M1/p1782483194674179)
- [AWS Durable Functions documentation](https://docs.aws.amazon.com/lambda/latest/dg/durable-functions.html)
- [AWS Step Functions
  documentation](https://docs.aws.amazon.com/step-functions/latest/dg/welcome.html)
