# TraceToGate

TraceToGate turns the traces of an AI agent into regression tests. Then it uses those tests to decide if a new agent version is safe to release.

The system has two pipelines:

- **Pipeline A (Trace-to-Eval)** finds failures in agent traces. It labels them and turns them into a versioned eval dataset.
- **Pipeline B (Replay and Release Gate)** runs agent v1 and agent v2 on that dataset. It compares the results with statistics and gives a **PASS**, **FAIL**, or **REVIEW** recommendation.

The test environment is [τ²-Bench](https://github.com/sierra-research/tau2-bench) Retail, a customer-service benchmark with tools, a policy, a database, and a simulated user.

## Design

Read the system design document for the full architecture, the services, the design decisions, and the data flow:

**[TraceToGate System Design](.agents/artifacts/tracetogate-foundations/04b-final-architecture-tracetogate-foundations.md)**

## Status

The design is complete. The build plan is next. See [WORKFLOW_STATUS.md](WORKFLOW_STATUS.md) for the current step and [checkpoint.md](checkpoint.md) for the decision log.
