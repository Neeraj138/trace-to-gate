---
type: research-questions
repos:
  - identifier: "github.com/Neeraj138/trace-to-gate"
    sha: "5353d91ec2c0af938fbf937469e1b1a6d61acadd"
flow: design-discussion
---

# Research Questions: TraceToGate Foundations

The repository has no code yet. The "current system" for this research is the external landscape that TraceToGate will be built on and measured against: the τ²-Bench environment, observability standards and platforms, and documented industry practice for trace-to-eval and release gating. Answer each question descriptively (what exists, how it works, what is known to fail), with sources. Do not propose TraceToGate's design.

## Key Context Pointers

- Project brief and draft architecture: `trace-to-gate-brief-and-discussion.md`
- τ²-Bench: https://github.com/sierra-research/tau2-bench (Retail domain; also the τ²-Bench paper on arXiv and the original τ-bench repo https://github.com/sierra-research/tau-bench)
- OpenTelemetry GenAI semantic conventions: https://opentelemetry.io/docs/specs/semconv/gen-ai/ and https://github.com/open-telemetry/semantic-conventions
- Langfuse: https://langfuse.com/docs — Braintrust: https://www.braintrust.dev/docs

## Questions

### τ²-Bench as the execution environment

1. **How does a τ²-Bench Retail run execute end to end?** Describe the components (agent, user simulator, orchestrator, tools, domain database/environment, policy), how a custom agent is plugged in, and what a saved simulation result contains (messages, tool calls and results, timestamps, costs, initial and final DB state, termination reason). Which of these are available without modifying τ²-Bench code? Start in the repo's `src/tau2/` package, its CLI/run entry points, and the Retail domain data and tools.

2. **How does τ²-Bench define and evaluate a task?** Describe the task schema (user scenario/instructions, initial state, expected actions, expected outputs), the reward/evaluation components (DB state comparison, action matching, communicated info, NL assertions or others), how multiple trials and pass^k are computed, and which parts of a run are non-deterministic (LLM agent, LLM user simulator, anything else). What does it take to re-run one specific task from a fixed initial state?

3. **What failure behavior is publicly documented for agents on τ-bench / τ²-bench Retail?** Reported failure categories (policy violations, wrong arguments, premature termination, user-simulator errors, etc.), typical pass^1 vs pass^k gaps, and any published or downloadable trajectory sets or leaderboard submissions that could serve as real failure data.

### Observability standards and platforms

4. **What do the OpenTelemetry GenAI semantic conventions currently define for agents, model calls, and tool calls?** Span names and kinds, key attributes, events, how prompt/response content capture is handled, the stability status of each part, and which instrumentation libraries emit them today (e.g. OpenLLMetry, OpenInference, official OTel contrib instrumentations for OpenAI/LiteLLM/Anthropic). How do they represent parent-child relationships for parallel tool calls?

5. **How do Langfuse and Braintrust model and expose trace and eval data?** For each: the data model (traces, spans/observations, scores, sessions), OTel ingestion support and attribute mapping, datasets and dataset items linked to source traces, dataset versioning, experiments/comparisons, annotation/review queues, export and query APIs, redaction/masking, self-hosting requirements, and pricing or free-tier limits relevant to a single-developer project.

### Industry practice: from traces to evals

6. **How do production teams turn traces into eval datasets and regression tests today?** Documented workflows and their steps (sampling, error analysis/open coding, failure taxonomies, annotation queues, promoting a trace to a dataset item, writing evaluators), from sources such as Hamel Husain / Shreya Shankar's eval writing, Anthropic and OpenAI eval guidance, and LangSmith, Braintrust, Langfuse, Arize Phoenix documentation. What do they report as common pitfalls (e.g. unrepresentative datasets, overfitting to past failures, stale cases)?

7. **What makes an agent trace replayable, and how do existing tools handle it?** Approaches for capturing or reconstructing environment state, recording and replaying tool responses (record/replay, mocks, simulated tools, sandboxed environments), handling non-deterministic user turns, and the documented limits of replaying a multi-turn agent conversation versus re-simulating it.

8. **What methods exist for detecting and classifying agent failures from traces?** Published failure taxonomies (e.g. MAST, TRAIL, τ-bench error analyses), rule-based detectors (loops, repeated tool calls, tool errors, unconfirmed side effects), LLM-as-judge over trajectories, and clustering of failures. What accuracy, false-positive, and false-negative figures are reported?

9. **How is drift detected for LLM and agent systems?** Methods for input drift (embedding or topic distribution shift), behavioral drift (tool-usage distributions, trajectory length, refusal/escalation rates), and outcome/latency/cost drift; the statistical tests used; how tools such as Arize, Evidently, WhyLabs/LangKit or the trace platforms above implement this; and how they separate real change from run-to-run randomness.

### Release gating (Pipeline B context)

10. **How do teams compare a baseline agent with a candidate before release?** Offline replay vs. re-simulation vs. shadow/canary/A-B in production; how stochasticity is handled (number of trials, pass^k, confidence intervals, paired comparisons, minimum detectable effect); what gate criteria and thresholds are documented; and existing CI integrations (e.g. Braintrust or LangSmith CI, promptfoo, τ²-Bench's own reporting).

11. **How is evaluator reliability measured?** How teams validate LLM-as-judge and code-based evaluators against human labels (agreement metrics, TPR/TNR, calibration), how evaluator versions are tracked, and how evaluator errors are accounted for when an eval result feeds a release decision.
