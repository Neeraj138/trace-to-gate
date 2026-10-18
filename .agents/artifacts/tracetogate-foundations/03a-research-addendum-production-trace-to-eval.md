---
date: "2026-10-19"
topic: "Production trace-to-eval practice: human and automated review, routing rules, failure taxonomy reviewer, generic versus τ²-Bench-specific parts of Pipeline A"
type: research
tags: [research, addendum, trace-to-eval, llm-judge, annotation-queue, routing, failure-taxonomy, human-review, tau2-bench]
status: complete
flow: design-discussion
addendum_to: "03-research-tracetogate-foundations.md"
note: "Addendum to 03. The user asked for this research during the design step (04), after the first draft of the design discussion. It does not change 03. It adds evidence for DQ6 and for the Pipeline A review and routing steps."
---

# Research Addendum: Production Trace-to-Eval Practice

## Research Questions

The user asked four questions during the design step:

1. Can a large language model (LLM) review replace human review in Pipeline A (trace-to-eval)?
2. How do production systems implement this pipeline? Do they add a trace to the eval dataset automatically when defined rules are true, and send the remaining traces to a human?
3. For the failure taxonomy step, what does the reviewer look like: deterministic code, an LLM, or other?
4. Is Pipeline A generic, or is it specific to τ²-Bench?

## Research Methodology

Four parallel research agents read vendor documentation, company engineering posts, papers, and preprints on 2026-10-19. This document merges their four reports. It does not repeat 03. References such as "03 §8" point to sections of `03-research-tracetogate-foundations.md`.

**Evidence levels.** Unmarked claims come from a source page that an agent opened. **(secondary)** means a third party wrote the source. **(preprint)** means an arXiv or alphaXiv paper with no completed peer review. **Inference** marks a conclusion that no source states directly. The merge step spot-checked five claims with WebFetch: the Anthropic CORE-Bench and "20-50 tasks" claims, the LangSmith cost cap and decision-model judge, the "Catching One in Five" preprint numbers, the Clio correlation, and the Shopify kappa numbers. All five matched their sources.

**Abbreviations.** TPR is true positive rate. TNR is true negative rate. κ is Cohen's kappa. CI is continuous integration, or confidence interval where the text says so. PR is pull request. PII is personally identifiable information. SME is subject matter expert. SOP is standard operating procedure. SDK is software development kit. API is application programming interface. SQL is Structured Query Language. CoT is chain of thought. DB is database.

## Summary

**(1) An LLM cannot replace human review in all of Pipeline A.** No primary source does this for open-ended agent work. LLM review is acceptable for a known failure mode after a human defines the mode and validates the judge against human labels. Humans stay in four places: discovery of new failure modes, ground-truth labels for judge validation, confirmation before a case enters a blocking suite, and a random audit sample. One production preprint shows the risk: a judge found 2 of 9 human-confirmed defect patterns ([arXiv 2606.10315](https://arxiv.org/abs/2606.10315), preprint).

**(2) Production systems use rules to route traces, and humans decide promotion.** Vendors store all traces, score a sample with LLM judges, filter by rules and scores, and send a capped stream to human queues. Only LangSmith, Datadog, and Confident AI document a rule that writes to a dataset with no human step. Datadog makes such datasets read-only. No source describes automatic promotion of an LLM-judged production failure into a blocking regression suite. The "auto-add, else human" router in section 5 is an Inference that combines several sources.

**(3) The taxonomy reviewer is layered.** Humans create the taxonomy with open coding and axial coding. Deterministic rules assign the categories that code can decide. An LLM classifier with a fixed taxonomy, few-shot examples, and an "other" exit assigns the remaining categories. Humans review "other", low-confidence, and audit items, and only humans change the taxonomy. The best published hybrid design (AgentRx) feeds a rule violation log to an LLM judge.

**(4) Pipeline A is generic in its stages and τ²-Bench-specific at four adapter points.** The stages match the vendor pattern. The adapters are state evidence, the replay environment, ground truth and reward, and fault injection.

## 1. Vendor Platforms

### 1.1 Comparison table

Legend: Y = documented. P = partial or manual only. N = not documented on the pages read. "N (nc)" = the agent did not check this feature.

| Platform | Online LLM judge + sampling | Rule writes to dataset (no human) | Rule writes to human queue | Routing on judge score | Judge alignment feature | Clustering / discovery | Dataset versions |
|---|---|---|---|---|---|---|---|
| LangSmith | Y (rate 0–1, weekly cost cap) | Y | Y | Y | Y (Align Evaluator, few-shot) | Y (Insights, ≤1,000 traces) | Y (tags, splits) |
| Langfuse | Y (observation level) | N (API only) | N (API only) | N | P (calibration guide, kappa in Score Analytics) | P (manual coding guide) | Y (items only) |
| Braintrust | Y (trace, span, group) | P (UI) | P (flag by API) | P | P (Loop edits scorers) | Y (Topics, daily) | Y (snapshots) |
| Arize AX | Y (span, trace, session) | P | P | P | P | Y (Alyx, managed agents) | Y |
| Phoenix (open source) | P (code evaluators) | N | N | N | N | N | Y (splits) |
| W&B Weave | Y (Signals in preview) | N | N | N | N | P | Y |
| Galileo | Y | N | N | N | Y (Autotune) | Y (Signals) | N (nc) |
| HoneyHive | Y (default 10%) | N | Y | Y | N | N | N (nc) |
| Datadog | Y | Y (≤100%, read-only) | Y (≤5%) | Y | P (workflow guidance) | Y (Cluster Map) | N (nc) |
| Confident AI | Y (referenceless only) | Y (daily cap) | Y | N (nc) | P | N | N (nc) |
| Opik | Y | N | Y (Oct 2026) | Y (≤5 AND conditions) | N | N | N (nc) |
| Raindrop | Y (signals as classifiers) | N | N | N | Y (Match / Not Match) | Y (Issues) | N |
| Laminar | Y (5–95% per user) | P (push approved items) | P (SQL export) | N | N | Y (Clusters) | N (nc) |

Sources for the cells are in section 1.2 and in the vendor documentation list in the References.

### 1.2 Key facts

- **LangSmith.** A rule has a filter, a sampling rate, and an action. Actions include "add to annotation queue" and "add to dataset". The dataset action writes inputs and outputs as examples with no human step ([rules](https://docs.langchain.com/langsmith/rules)). Online evaluators can filter on another evaluator's score, so teams can chain a judge and a routing rule ([online evaluators](https://docs.langchain.com/langsmith/online-evaluations-llm-as-judge)). At the weekly cost cap, LangSmith pauses the evaluator until Monday 00:00 UTC. A decision model such as Jev can replace the LLM judge (same URL; spot-checked).
- **LangSmith judge alignment.** Experts label runs in a queue. The docs suggest at least 20 examples, balanced across labels. The alignment score is the percentage of examples where judge and human agree. The docs give no threshold ([Align Evaluator](https://docs.langchain.com/langsmith/improve-judge-evaluator-feedback)).
- **Langfuse.** Annotation queues get items manually, in bulk, or by API. The page describes no rule-based routing ([annotation queues](https://langfuse.com/docs/evaluation/evaluation-methods/annotation-queues)). The calibration guide uses an example gate of 0.90 TPR and 0.90 TNR on 35 items. It also recommends continued sampled human review in production ([calibration guide](https://langfuse.com/guides/llm-as-a-judge-calibration-skill)).
- **Braintrust.** The sampling guidance is 1–10% for high volume and 50–100% for low-volume or critical apps ([online scoring](https://www.braintrust.dev/docs/observe/score-online)). In the Topics cookbook, Topics finds a cluster, a human reads logs for 20–30 minutes, and the human saves the cluster to a dataset ([cookbook](https://www.braintrust.dev/docs/cookbook/recipes/FindingProductionIssues.md)).
- **Arize AX.** "Start at 10–20% and increase once you have validated your evaluator is working correctly." "Ground what you automate in trace review and a clear failure taxonomy first" ([online evals](https://arize.com/docs/ax/evaluate/online-evals/setting-up-online-evals)). Manual annotation "typically produces sharper, more reliable categories" than AI routes ([Agent Insights](https://arize.com/docs/ax/observe/agent-insights)).
- **Datadog.** Automation rules (announced 2026-03-23) route traces to a dataset or a queue on low eval scores, thumbs-down, errors, latency, or sampling ([blog](https://www.datadoghq.com/blog/automations-annotation-queues/)). Sampling caps are up to 5% for queues and up to 100% for datasets. Datasets that automations fill are read-only and hold up to 20,000 records ([automation rules](https://docs.datadoghq.com/llm_observability/configure/automation_rules/)). Queues hold up to 1,000 records, and the automation pauses when a queue is full ([queues](https://docs.datadoghq.com/llm_observability/investigate/annotation_queues/)).
- **Confident AI.** Auto-ingest rules send traces to a dataset, a queue, or both. The example uses stratified sampling with at most 200 traces per day. "Add sampling so you only ingest what you can actually review" ([blog](https://www.confident-ai.com/blog/launch-week-q1-2026-day-3-auto-ingest-traces)).
- **Opik.** Queue automation joins up to five feedback-score conditions with AND. It "never re-adds an item a reviewer removed" ([queues](https://www.comet.com/docs/opik/evaluation/advanced/annotation_queues.md)).
- **HoneyHive.** Queue automation adds matching events in real time, for example `metrics.accuracy < 0.7`. Humans suit "subjective qualities like brand voice, creative quality, or domain-specific accuracy" ([queues](https://docs.honeyhive.ai/v2/evaluation/annotation-queues)).
- **Status changes.** Humanloop shut down on 2025-09-08 ([migration page](https://humanloop.com/docs/v5/guides/migrating-from-humanloop)).

### 1.3 Common five-stage pattern (Inference)

All major vendors build the same funnel. This structure is an Inference from section 1.2.

1. **Capture all traces, score a sample.** Platforms store all traces and run online judges on a sampled subset to control cost.
2. **Filter by rules and scores.** Deterministic filters (metadata, errors, latency, thumbs-down) and judge-score thresholds select candidates.
3. **Discover failure modes with an LLM.** Clustering tools propose categories (LangSmith Insights, Braintrust Topics, Galileo Signals, Datadog Cluster Map).
4. **Route a capped stream to humans.** Examples are the Datadog 5% and 1,000-record caps and the Confident AI 200-per-day example.
5. **Promote to the dataset.** The dataset write is usually a human action from a queue or a filtered table.

The cap of 5% on queues and 100% on datasets in Datadog shows that the vendor treats human attention as the scarce resource (Inference). The docs show three gaps (Inference). Near-duplicate removal before dataset entry is almost absent. No vendor documents a closed loop from a discovered cluster to an automatic routing rule. Only Langfuse shows a numeric trust gate (0.90 TPR and TNR).

## 2. Company Practice

### 2.1 Who finds failures and who promotes them

| Company or author | How they find failures | Who promotes a failure to an eval case | Source type |
|---|---|---|---|
| Anthropic (agent evals guide) | Bug tracker, support queue, manual checks | Human | Primary |
| Anthropic (Claude Code review) | Incident PRs, reverts, fix-forwards | Engineers confirm each bug | Primary |
| Anthropic (Clio) | LLM clusters of conversations | Analysts act on clusters; no automated enforcement | Primary |
| Hamel Husain, Shreya Shankar | Human open coding of traces | One domain expert ("benevolent dictator") | Primary |
| Sierra | Customer team reports a live conversation | Human | Primary |
| Intercom Fin | Monitors flag live conversations | Human turns the conversation into a Simulation | Primary |
| Shopify Sidekick | Sampled merchant conversations | At least three product experts label | Primary |
| DoorDash support | Support cases, early simulations | Engineers | Secondary (InfoQ) |
| Notion | Worst customer experiences, trace search | Data specialist | Vendor case study |
| Zapier | Explicit and implicit feedback, churn | Human, with a one-click path | Primary talk |
| Replit | Trace clusters (Telescope) | Agent proposes a PR; human approves | Secondary (ZenML) |
| GitHub Copilot | Daily runs on production models | Human audit on degradation | Primary |
| Dropbox Dash | Human labels; conflicts between user behavior and LLM | Human | Primary |
| Cursor Tab | Accept and reject on 400M+ requests per day | Automatic (reward signal, no LLM reviewer) | Primary |

Sources: [Anthropic evals guide](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents), [Claude Code review](https://claude.dev/blog/solving-code-review-at-anthropic/) (15 October 2026), [Clio](https://www.anthropic.com/research/clio), [Hamel FAQ](https://hamel.dev/blog/posts/evals-faq/), [Sierra](https://sierra.ai/blog/simulations-the-secret-behind-every-great-agent), [Intercom](https://www.intercom.com/blog/announcing-evals-and-releases/), [Shopify](https://shopify.engineering/building-production-ready-agentic-systems), [DoorDash (InfoQ)](https://www.infoq.com/news/2026/03/doordash-llm-chatbot-simulator/), [Notion](https://www.braintrust.dev/blog/notion), [Zapier talk](https://www.ai.engineer/talks/blrovBxxN9o-turning-fails-into-features-zapiers-hard-won), [Replit (ZenML)](https://www.zenml.io/llmops-database/closing-the-loop-continuous-evaluation-and-improvement-of-ai-coding-agents-at-scale), [GitHub](https://github.blog/ai-and-ml/generative-ai/how-we-evaluate-models-for-github-copilot/), [Dropbox](https://dropbox.tech/machine-learning/llm-human-labeling-improving-search-relevance-dropbox-dash), [Cursor](https://cursor.com/blog/tab-rl).

### 2.2 The Anthropic code-review loop

This source shows a complete incident-to-eval loop with tiered human routing ([claude.dev](https://claude.dev/blog/solving-code-review-at-anthropic/)).

- The team collects past PRs that caused incidents. Engineers who know the code confirm each bug and its location. Each confirmed incident becomes one case.
- The reviewer runs against the full incident set daily. The rule for a change is "Score went up? keep it; if not, drop it."
- The team also runs the reviewer on PRs known to be clean, and counts harmful reports.
- A classifier sets a risk tier from the change. Low tier needs no human after checks pass. Medium tier needs one person. High tier needs two people.
- "The change sets the tier, not the author." "An agent's approval never counts as a person's."
- Results: one-round pass rate went from 65% to 78%. PRs that needed five or more rounds went from 8.9% to 1.5%. Revert share fell by more than 40%.

### 2.3 Judge validation in practice

- **Anthropic.** Use deterministic graders where possible and LLM graders where necessary. Human graders calibrate model graders. "You won't know if your graders are working well unless you read the transcripts and grades from many trials" ([evals guide](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)).
- **Grader bugs hide capability.** Opus 4.5 scored 42% on CORE-Bench. After bug fixes and a less constrained scaffold, it scored 95% (same URL; spot-checked).
- **Hamel Husain.** Label 100 to 200 examples per failure mode. Split them 10–20% train, 40–45% dev, and 40–45% test. Below 60 examples, confidence intervals are too wide. Report TPR and TNR separately ([FAQ](https://hamel.dev/blog/posts/evals-faq/)). In the Honeycomb example, the judge reached more than 90% agreement with the expert after three rounds ([judge guide](https://hamel.dev/blog/posts/llm-judge/)).
- **Shopify.** Judges improved from κ 0.02 to κ 0.61. The human-human baseline was κ 0.69. Shopify randomly replaced the judge with a human for each conversation, and trusted the judge when evaluators could not tell the difference ([Shopify](https://shopify.engineering/building-production-ready-agentic-systems); spot-checked).
- **Dropbox.** DSPy optimizers propose judge prompt changes and test them against human labels. Cases where user behavior conflicts with the LLM rating go to human review first ([Dropbox](https://dropbox.tech/machine-learning/llm-human-labeling-improving-search-relevance-dropbox-dash)).
- **Gaps.** Sierra and Intercom do not describe how they calibrate their grading agents. GitHub says alignment with human reviewers is "a challenge" and gives no number.

### 2.4 Replay methods

| Method | Used by | Source |
|---|---|---|
| LLM-simulated user with persona and goal | Sierra, Intercom, Shopify, DoorDash | [Sierra](https://sierra.ai/blog/simulations-the-secret-behind-every-great-agent), [Intercom](https://www.intercom.com/blog/announcing-evals-and-releases/), [Shopify](https://shopify.engineering/building-production-ready-agentic-systems), [InfoQ](https://www.infoq.com/news/2026/03/doordash-llm-chatbot-simulator/) |
| Replay of the first N−1 turns, test turn N | Practitioner guidance | [Hamel FAQ](https://hamel.dev/blog/posts/evals-faq/) |
| Mocked service APIs | DoorDash | [InfoQ](https://www.infoq.com/news/2026/03/doordash-llm-chatbot-simulator/) |
| Recorded tool responses; synthetic user environments | Zapier | [Zapier talk](https://www.ai.engineer/talks/blrovBxxN9o-turning-fails-into-features-zapiers-hard-won) |
| Clean environment per trial; forked sandboxes | Anthropic, Replit | [Anthropic](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents), [ZenML](https://www.zenml.io/llmops-database/closing-the-loop-continuous-evaluation-and-improvement-of-ai-coding-agents-at-scale) |

### 2.5 Key numbers

| Metric | Value | Source |
|---|---|---|
| Starting eval size | 20–50 tasks from real failures | [Anthropic](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) |
| Traces for error analysis | At least 100, or saturation | [Hamel FAQ](https://hamel.dev/blog/posts/evals-faq/) |
| Human-only first pass before LLM help | At least 30 traces | [Hamel FAQ](https://hamel.dev/blog/posts/evals-faq/) |
| Clio cluster versus Trust and Safety classifiers | r = 0.71 | [Clio](https://www.anthropic.com/research/clio) |
| Sierra simulated tests | More than 35,000 per day | [Sierra](https://sierra.ai/blog/simulations-the-secret-behind-every-great-agent) |
| Cursor Tab cycle | 1.5–2 hours; 21% fewer suggestions, 28% higher accept rate | [Cursor](https://cursor.com/blog/tab-rl) |
| DoorDash hallucination reduction before deployment | About 90% | [InfoQ](https://www.infoq.com/news/2026/03/doordash-llm-chatbot-simulator/) (secondary) |

Cursor Tab is the only fully automatic loop found. It uses a dense, direct user signal (accept or reject), not an LLM reviewer. Full automation works when the outcome signal is immediate, dense, and unambiguous, and most agent tasks do not have such a signal (Inference).

### 2.6 Lesson from the Claude Code postmortem

"Neither our internal usage nor evals initially reproduced the issues" ([postmortem](https://anthropic.com/engineering/april-23-postmortem)). User reports found the regressions first. Inference: a production loop must keep a channel for raw user reports.

## 3. Research Literature

### 3.1 What is automatable today

| Sub-task | Evidence | Status |
|---|---|---|
| Pass/fail on tasks with a checkable outcome, with an agentic judge | Agent-as-a-Judge 83.9–92.1% alignment versus human majority 94.0–95.1% ([arXiv 2410.10934](https://arxiv.org/html/2410.10934)) | Automatable with audit |
| Labels from a fixed failure taxonomy with few-shot examples | MAST o1: accuracy 0.94, F1 0.80, κ 0.77. Without few-shot: κ 0.58 ([MAST](https://arxiv.org/html/2503.13657)) | Automatable with audit |
| Step-by-step fault detection when the fault shows in the trace | All four injected fault types flagged, 0 of 100 clean runs flagged ([arXiv 2609.00038](https://arxiv.org/abs/2609.00038), preprint) | Automatable for visible faults; controlled testbed only (Inference) |
| Clustering into candidate themes | Clio: 94% recovery of ground-truth categories on 19,476 synthetic transcripts ([Clio paper](https://arxiv.org/html/2412.13678v1)) | Proposal step; humans confirm |
| Deterministic replay of a recorded incident | Chronicle: bit-stable over 20 runs, caught every relevant mutant ([arXiv 2609.20625](https://arxiv.org/abs/2609.20625), preprint) | Automatable when boundaries are recorded |
| Aggregate pass rates from judge labels | Prediction-powered inference gives valid confidence intervals with a small labelled set ([arXiv 2301.09633](https://arxiv.org/abs/2301.09633)); ARES uses "a few hundred" labels ([arXiv 2311.09476](https://arxiv.org/abs/2311.09476)) | Automatable with a small human sample |
| Abstention with guaranteed agreement | Trust or Escalate: more than 80% human agreement at almost 80% coverage ([arXiv 2407.18370](https://arxiv.org/abs/2407.18370)) | Automatable for the confident share |

### 3.2 What is not reliable without humans

| Sub-task | Evidence |
|---|---|
| Exact decisive-step localization | Who&When best step-level 14.2% ([arXiv 2505.00212](https://arxiv.org/abs/2505.00212)). AgenTracer-8B 20.7% handcrafted, 42.9% automated ([arXiv 2509.03312](https://arxiv.org/html/2509.03312)). LongRCA best 24.1% ([arXiv 2608.15242](https://arxiv.org/abs/2608.15242), preprint) |
| Joint category and location in long traces | TRAIL best joint 0.183 on GAIA, 0.050 on SWE-Bench ([TRAIL](https://arxiv.org/html/2505.08638)) |
| Success judgment in open web tasks | No judge reaches 70% precision ([AgentRewardBench](https://arxiv.org/html/2504.08942)) |
| Discovery of new production defect patterns | Judge found 2 of 9 patterns (22%). Its gate flagged 0 of 100 rounds where humans confirmed 23 defects ([arXiv 2606.10315](https://arxiv.org/abs/2606.10315), preprint; spot-checked) |
| Silent faults with an outcome-only judge | Outcome-only recall 34–76% with zero paired discrimination ([arXiv 2609.00038](https://arxiv.org/abs/2609.00038), preprint) |
| Definition of the criteria | Criteria drift ([EvalGen](https://arxiv.org/html/2404.12272)) |

TRAIL human annotation took about 110 to 120 minutes per trace ([TRAIL](https://arxiv.org/html/2505.08638)). This shows the cost of full human root-cause labels.

### 3.3 Risks of a fully automated loop

- **Self-preference.** LLM judges recognize their own outputs and prefer them ([arXiv 2404.13076](https://arxiv.org/abs/2404.13076)). Judges also favor similar models, and errors become more correlated as capability increases ([arXiv 2502.04313](https://arxiv.org/abs/2502.04313)). A panel of judges from different families beats a single GPT-4 judge: κ 0.763 versus 0.627, at seven to eight times lower cost ([PoLL](https://arxiv.org/html/2404.18796)).
- **Master-key attacks.** Trivial responses such as ":" fool generative judges. The false positive rate reached 53.6% for Claude-4 on MATH and 28.9% for GPT-4o ([arXiv 2507.08794](https://arxiv.org/html/2507.08794)).
- **Optimization against the monitor.** A CoT monitor caught reward hacks with 95% recall versus 60% for an action-only monitor. Strong optimization against the monitor causes obfuscated reward hacking ([arXiv 2503.11926](https://arxiv.org/html/2503.11926)).
- **Criteria drift.** Users need criteria to grade outputs, but grading outputs changes the criteria. A fixed automated rubric becomes stale ([EvalGen](https://arxiv.org/html/2404.12272)).
- **Simulator error.** In τ²-Bench, critical user-simulator errors occur in 13% of airline, 12% of retail, and 6% of telecom conversations ([τ²-Bench](https://arxiv.org/html/2506.07982v1)). Agent success varies up to 9 points across user LLMs ([Lost in Simulation](https://arxiv.org/abs/2601.17087)).
- **Benchmark artifacts.** In CatchBench, a trivial rule got perfect F1 on one source ([arXiv 2608.22808](https://arxiv.org/abs/2608.22808), preprint).

## 4. Failure Taxonomy Reviewer

### 4.1 Creation

Humans create the taxonomy in practice.

- A domain expert collects about 100 diverse traces and writes free-text notes. The expert notes only the first failure, because errors cascade. The expert annotates at least 30 traces before looking at any agent suggestion ([Hamel FAQ](https://hamel.dev/blog/posts/evals-faq/why-is-error-analysis-so-important-in-llm-evals-and-how-is-it-performed.html); [error-analysis skill](https://tessl.io/registry/skills/github/hamelsmu/evals-skills/error-analysis)).
- Axial coding groups notes into 5–10 categories, each with a name and a one-sentence definition. The process takes 2–3 merge and split rounds. A rough stop rule is no new category in the last 20 traces (same skill URL).
- An LLM can propose groupings after a human open-codes 30–50 traces. An LLM must not do the first open coding, validate the taxonomy, or make ground-truth labels ([Hamel automation FAQ](https://hamel.dev/blog/posts/evals-faq/what-parts-of-evals-can-be-automated-with-llms.html)). LLMs cluster by surface similarity, for example "app crashes" with "login is slow" ([skill](https://tessl.io/registry/skills/github/hamelsmu/evals-skills/error-analysis)).
- Research taxonomies use the same method. MAST experts refined 14 modes over three rounds to κ 0.88 ([MAST](https://arxiv.org/html/2503.13657)). AgentRx derived nine categories from 115 annotated failed trajectories ([AgentRx](https://www.microsoft.com/en-us/research/blog/systematic-debugging-for-ai-agents-introducing-the-agentrx-framework/)).
- Automatic tools (Clio, TnT-LLM, LangSmith Insights, Braintrust Topics, ErrorMap) all summarize each trace, cluster the text, and let an LLM name clusters. No vendor publishes evidence that its clusters match a human agent-failure taxonomy (Inference from [Clio paper](https://arxiv.org/html/2412.13678), [TnT-LLM](https://arxiv.org/html/2403.12173), [ErrorMap](https://arxiv.org/html/2601.15812v2), [LangSmith Insights](https://docs.langchain.com/langsmith/insights)).
- AdaMAST induced a taxonomy from a system's own traces. It aligned better with TRAIL experts than a handcrafted taxonomy: κ 0.725 versus 0.516 ([AdaMAST](https://www.alphaxiv.org/abs/2607.16387), preprint).

### 4.2 Assignment accuracy

| Study | Task | Result |
|---|---|---|
| MAST | o1, definitions + few-shot, versus humans | Accuracy 0.94, F1 0.80, κ 0.77 ([MAST](https://arxiv.org/html/2503.13657)) |
| Model or Harness (41 modes) | GPT-5.5 versus human, coarse / full mode | κ 0.76 / κ 0.71 ([alphaXiv 2607.28802](https://alphaxiv.org/abs/2607.28802), preprint) |
| TnT-LLM | Primary label / exact multi-label | GPT-4 κ 0.572–0.695 / κ 0.10–0.27 ([TnT-LLM](https://arxiv.org/html/2403.12173)) |
| TRAIL | Category F1 on GAIA | 0.389 ([TRAIL](https://arxiv.org/html/2505.08638)) |
| AgentErrorBench | Human-human, 5 modules | κ 0.55 ([alphaXiv overview](https://www.alphaxiv.org/overview/2509.25370v1), preprint) |
| AgentRx | Rules + LLM versus LLM prompting | +23.6 points localization, +22.9% root cause ([AgentRx](https://www.microsoft.com/en-us/research/blog/systematic-debugging-for-ai-agents-introducing-the-agentrx-framework/)) |

Agreement is higher on coarse categories and on one primary label. Exact multi-label sets and exact steps are much harder. Pipeline A labels only the first failure at a known step. So the taxonomy reviewer solves the easier problem, closer to MAST than to TRAIL (Inference).

### 4.3 Reviewer layers (Inference)

This layer design is an Inference that combines [AgentRx](https://www.microsoft.com/en-us/research/blog/systematic-debugging-for-ai-agents-introducing-the-agentrx-framework/), [Hamel FAQ](https://hamel.dev/blog/posts/evals-faq/), [Anthropic](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents), and [MAST](https://arxiv.org/html/2503.13657).

| Layer | Type | Output | Accuracy measure |
|---|---|---|---|
| L1 Rules | Code on tool calls and DB state | Rule code with evidence span | Precision and recall on labelled traces; precision is most important |
| L2 LLM classifier | Fixed taxonomy, one label for the first failure, few-shot, "other/unknown" exit | Category, rationale, confidence | Per-category TPR and TNR, κ on a frozen human set |
| L3 Human | Taxonomy owner | Final label for "other", low-confidence, L1–L2 conflicts, random 5–10% audit | Human change rate per category |
| L4 Taxonomy owner | Human approves; LLM clusters "other" notes | New taxonomy version with an old-to-new map | "Other" rate, coverage, κ on re-validation |

Precedence rules (Inference):

1. If an L1 rule fires on the first-failure step, its category wins.
2. The LLM gets the L1 log as evidence, as in AgentRx.
3. The LLM gives one primary label, because exact multi-label agreement is low.
4. Only a human creates, merges, splits, or renames a category.

### 4.4 Maintenance

- Re-run error analysis after new features, prompt rewrites, model changes, or incidents ([skill](https://tessl.io/registry/skills/github/hamelsmu/evals-skills/error-analysis)).
- Give LLM judges an "Unknown" option ([Anthropic](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)).
- Coverage benchmarks: ErrorMap 95.2%, TnT-LLM more than 99.5% ([ErrorMap](https://arxiv.org/html/2601.15812v2), [TnT-LLM](https://arxiv.org/html/2403.12173)).
- AdaMAST runs a merge, add, or relabel loop when κ falls below 0.75 ([AdaMAST](https://www.alphaxiv.org/abs/2607.16387), preprint).
- Gap: no source gives a tested novelty-detection method or a taxonomy versioning scheme.

### 4.5 Published taxonomies and fit to τ²-Bench Retail

| Taxonomy | Size | Fit to τ²-Bench Retail |
|---|---|---|
| τ-bench failure analysis ([PDF](https://arxiv.org/pdf/2406.12045)) | 4 groups: wrong argument, wrong information, wrong decision, partial resolution; wrong argument + wrong information ≈ 55% | Same domain, but coarse |
| AgentRx ([blog](https://www.microsoft.com/en-us/research/blog/systematic-debugging-for-ai-agents-introducing-the-agentrx-framework/)) | 9 categories, including plan adherence failure, invalid invocation, under-specified user intent, guardrails triggered | Strong fit; includes τ-bench data and separates user-intent issues |
| MAST ([HTML](https://arxiv.org/html/2503.13657)) | 14 modes, 3 groups | Partial; no inter-agent modes in a single-agent domain |
| TRAIL ([HTML](https://arxiv.org/html/2505.08638)) | 3 areas, 20+ leaf types | Good for tool and system errors; weak on policy errors |
| Model or Harness ([alphaXiv](https://alphaxiv.org/abs/2607.28802), preprint) | 41 modes; families User, Harness, Environment | Good for agent bug versus harness bug versus simulator bug |

No published taxonomy names the project's side-effect categories, such as "write without confirmation" or "duplicate write after timeout". AgentRx "plan adherence" and "guardrails" are the nearest parents (Inference).

Reviewer fit for typical Retail failure modes (Inference):

| Failure mode | Reviewer |
|---|---|
| Schema error, unknown ID, write without confirmation, duplicate write, second modify on one order, final DB state differs | L1 rule |
| Correct tool with wrong argument | L1 finds the diff; L2 names the reason |
| Wrong information to the user, policy misread, partial resolution | L2 LLM, binary per category |
| Misread user intent | L2 LLM, then human audit |
| Benchmark or simulator bug, new pattern | L3 human |

## 5. Routing Patterns Seen in Practice

Each row marks the evidence level. "Documented" means a named source describes the practice. "Inference" means no single source describes it.

| Pattern | Evidence level | What the sources show |
|---|---|---|
| Rules on all traffic, judges on a sample | Documented | Vendor sampling guidance (section 1.2). "Run cheap checks on every trace and an expensive judge on a nightly sample" ([Hamel FAQ](https://hamel.dev/blog/posts/evals-faq/)) |
| Rule writes directly to a dataset | Documented | LangSmith, Datadog, Confident AI. Datadog makes these datasets read-only ([Datadog](https://docs.datadoghq.com/llm_observability/configure/automation_rules/)) |
| Auto-add as a non-blocking candidate | Inference | Datadog read-only datasets separate automatic data from curated data. Anthropic adds a case only after an engineer confirms it ([claude.dev](https://claude.dev/blog/solving-code-review-at-anthropic/)) |
| Auto-pass of clean items | Documented | Intercom Auto-review passes a conversation when all AI criteria pass; people review failures and edge cases ([Fin Monitors](https://fin.ai/help/en/articles/13976253-monitors-explained)) |
| Auto-reject (do not add) | Partly documented | Zapier: do not turn every positive response into an eval ([Zapier talk](https://www.ai.engineer/talks/blrovBxxN9o-turning-fails-into-features-zapiers-hard-won)). Opik does not re-add removed items |
| Human queue on score condition | Documented | LangSmith, HoneyHive, Opik, Datadog score-based rules |
| Human queue on conflict | Documented | Dropbox routes conflicts between user behavior and LLM ratings to humans first |
| Human queue on judge instability | Secondary | Run the judge 8 times and route flips to humans. Flagged items are 2.1–3.0 times more likely to be contested, but flip rate catches only 16–31% of contested items ([orq.ai](https://orq.ai/blog/llm-as-a-judge-alignment-with-humans), vendor blog) |
| Human queue by risk tier | Documented (code review) | Anthropic risk tiers (section 2.2) |
| Random audit sample of automatic lanes | Documented as guidance | "Keep some random traces in every batch" ([Hamel FAQ](https://hamel.dev/blog/posts/evals-faq/)). Langfuse recommends sampled human review in production |
| Tiered sampling: 100% rules, 5–15% judge, about 1% human; recalibrate above 30% human overturn | Secondary | [channel.tel](https://www.channel.tel/blog/tiered-eval-sampling-production-agents), vendor blog |
| Promotion to a blocking suite needs a human | Documented for incidents; Inference for LLM-judged traces | Engineers confirm each incident case. "An agent's approval never counts as a person's" ([claude.dev](https://claude.dev/blog/solving-code-review-at-anthropic/)). No source describes automatic promotion of an LLM-judged failure into a blocking suite |
| Reproduction check before promotion | Inference | Chronicle turns an incident into a CI test by cut-point replay ([arXiv 2609.20625](https://arxiv.org/abs/2609.20625), preprint) |

Two further points are Inference. First, auto-accept needs a stricter threshold than auto-reject, because a false pass hides a failure, while a false fail costs only a review. AgentRewardBench supports this: judge precision on "success" is below 70% ([AgentRewardBench](https://arxiv.org/html/2504.08942)). Second, a plausible automation share for pass/fail is 70–80% automatic and 20–30% human, based on Trust or Escalate coverage and Agent-as-a-Judge alignment. No source tests this share on agent traces.

## 6. Generic Versus τ²-Bench-Specific Parts of Pipeline A

The Pipeline A stages match the vendor pattern in section 1.3 and the company loops in section 2. They do not depend on τ²-Bench (Inference). The generic stages are: trace capture through OpenTelemetry (03 §4), normalization and completeness checks, rule and judge detection, the review queue, the failure taxonomy, conversion to an eval case, and dataset versioning.

The τ²-Bench-specific parts sit at four adapter points. Each row compares the τ²-Bench case with production practice.

| Adapter point | τ²-Bench Retail | Production equivalent in the sources |
|---|---|---|
| State evidence | τ²-Bench saves no initial or final DB state. The evaluator rebuilds state by replay (03 §1, §2) | Langfuse and Braintrust link dataset items to source traces; Braintrust can store a whole-trace reference ([Braintrust](https://www.braintrust.dev/docs/kb/add-to-dataset-captures-root-span-only.md)). Side-effect state needs a recorder in both cases (Inference) |
| Replay environment | Fresh in-memory DB from `db.json`; LLM user simulator (03 §1) | Mocked APIs (DoorDash), recorded tool responses and synthetic environments (Zapier), data connectors (Intercom), forked sandboxes (Replit, Hex) (section 2.4) |
| Ground truth and reward | `reward_info`: DB hash match and NL assertions (03 §2) | Production has no gold DB. Ground truth comes from human labels, user signals, and incident confirmation (section 2). This is the largest difference (Inference) |
| Fault injection | None in τ²-Bench (03 §1) | Fault injection and counterfactual replay make labelled failure data in research ([AgenTracer](https://arxiv.org/html/2509.03312), [Who&When Pro](https://arxiv.org/abs/2607.09996), preprint) |

Two more τ²-Bench specifics affect review. First, the user simulator causes about 12% critical errors in Retail, so the review step needs a "simulator fault" label ([τ²-Bench](https://arxiv.org/html/2506.07982v1)). Production systems have real users instead, and Lost in Simulation shows that simulators are poor proxies for real users ([arXiv 2601.17087](https://arxiv.org/abs/2601.17087)). Second, the Retail policy is explicit, so more categories suit L1 rules than in a typical production domain (Inference).

Inference: if the four adapters sit behind interfaces, the same pipeline can run on another τ²-Bench domain or on production traces. A production adapter replaces the gold DB with human and user-signal ground truth.

## 7. Open Questions and Gaps

- **Secondary sources.** DoorDash details come from InfoQ, because the primary post returned HTTP 403. Replit details come from ZenML. Notion and the Zapier customer page are vendor case studies. The tiered-sampling and flip-rate numbers come from vendor blogs (channel.tel, orq.ai).
- **Preprints.** Many 2026 numbers come from arXiv or alphaXiv preprints with no completed peer review: 2606.10315, 2609.00038, 2609.20625, 2608.15242, 2608.22808, 2607.09996, AdaMAST (2607.16387), Model or Harness (2607.28802), and AgentErrorBench (2509.25370). The agents read abstracts or HTML full text through WebFetch, which summarizes with a small model.
- **Unverified items.** Patronus docs returned HTTP 401, so Patronus facts come from search excerpts. The Atla τ-bench annotation study page did not load (HTTP 404), so this addendum gives no Atla numbers. TRAIL reports "high" inter-annotator agreement with no number.
- **"Not checked" cells.** "N (nc)" in section 1.1 means the agent read no page on that feature. It does not mean the feature is absent.
- **Missing evidence.** No source gives an explicit near-duplicate threshold for eval cases. No source gives an auto-promotion criterion for LLM-judged production failures. No source gives a tested novelty-detection method or a taxonomy versioning scheme. No named company publishes human review cost per trace.
- **Companies with no usable source.** Klarna, Decagon, Ramp, LinkedIn, Uber, Airbnb, Netflix, and Cognition gave no usable primary source in the searches.
- **Open question for the design (DQ6).** The project must decide if auto-added cases go to a non-blocking candidate tier, and who confirms promotion to the blocking suite. The evidence supports human confirmation, but the decision belongs to 04.

## References

**Vendor documentation**
- https://docs.langchain.com/langsmith/rules
- https://docs.langchain.com/langsmith/online-evaluations-llm-as-judge
- https://docs.langchain.com/langsmith/improve-judge-evaluator-feedback
- https://docs.langchain.com/langsmith/insights
- https://langfuse.com/docs/evaluation/evaluation-methods/annotation-queues
- https://langfuse.com/guides/llm-as-a-judge-calibration-skill
- https://www.braintrust.dev/docs/observe/score-online
- https://www.braintrust.dev/docs/cookbook/recipes/FindingProductionIssues.md
- https://www.braintrust.dev/docs/kb/add-to-dataset-captures-root-span-only.md
- https://arize.com/docs/ax/evaluate/online-evals/setting-up-online-evals
- https://arize.com/docs/ax/observe/agent-insights
- https://www.datadoghq.com/blog/automations-annotation-queues/
- https://docs.datadoghq.com/llm_observability/configure/automation_rules/
- https://docs.datadoghq.com/llm_observability/investigate/annotation_queues/
- https://www.confident-ai.com/blog/launch-week-q1-2026-day-3-auto-ingest-traces
- https://www.comet.com/docs/opik/evaluation/advanced/annotation_queues.md
- https://docs.honeyhive.ai/v2/evaluation/annotation-queues
- https://docs.galileo.ai/concepts/metrics/autotune-llm-as-a-judge-metrics.md
- https://laminar.sh/docs/queues/quickstart.md
- https://raindrop.ai/docs/platform/signals.md
- https://humanloop.com/docs/v5/guides/migrating-from-humanloop
- https://docs.coreweave.com/weave/guides/evaluation/custom-monitors
- https://fin.ai/help/en/articles/13976253-monitors-explained

**Company and practitioner sources**
- https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents
- https://claude.dev/blog/solving-code-review-at-anthropic/
- https://anthropic.com/engineering/april-23-postmortem
- https://www.anthropic.com/research/clio
- https://hamel.dev/blog/posts/evals-faq/
- https://hamel.dev/blog/posts/evals-faq/what-parts-of-evals-can-be-automated-with-llms.html
- https://hamel.dev/blog/posts/evals-faq/why-is-error-analysis-so-important-in-llm-evals-and-how-is-it-performed.html
- https://hamel.dev/blog/posts/llm-judge/
- https://tessl.io/registry/skills/github/hamelsmu/evals-skills/error-analysis
- https://sierra.ai/blog/simulations-the-secret-behind-every-great-agent
- https://www.intercom.com/blog/announcing-evals-and-releases/
- https://shopify.engineering/building-production-ready-agentic-systems
- https://www.infoq.com/news/2026/03/doordash-llm-chatbot-simulator/
- https://www.braintrust.dev/blog/notion
- https://www.ai.engineer/talks/blrovBxxN9o-turning-fails-into-features-zapiers-hard-won
- https://www.zenml.io/llmops-database/closing-the-loop-continuous-evaluation-and-improvement-of-ai-coding-agents-at-scale
- https://github.blog/ai-and-ml/generative-ai/how-we-evaluate-models-for-github-copilot/
- https://dropbox.tech/machine-learning/llm-human-labeling-improving-search-relevance-dropbox-dash
- https://cursor.com/blog/tab-rl
- https://www.microsoft.com/en-us/research/blog/systematic-debugging-for-ai-agents-introducing-the-agentrx-framework/
- https://orq.ai/blog/llm-as-a-judge-alignment-with-humans
- https://www.channel.tel/blog/tiered-eval-sampling-production-agents

**Papers and preprints**
- https://arxiv.org/html/2410.10934
- https://arxiv.org/html/2503.13657
- https://arxiv.org/html/2504.08942
- https://arxiv.org/abs/2606.10315
- https://arxiv.org/abs/2609.00038
- https://arxiv.org/abs/2404.13076
- https://arxiv.org/abs/2502.04313
- https://arxiv.org/html/2507.08794
- https://arxiv.org/abs/2505.00212
- https://arxiv.org/html/2505.08638
- https://arxiv.org/html/2509.03312
- https://arxiv.org/abs/2608.15242
- https://arxiv.org/abs/2607.09996
- https://arxiv.org/abs/2608.22808
- https://arxiv.org/html/2404.12272
- https://arxiv.org/abs/2407.18370
- https://arxiv.org/html/2404.18796
- https://arxiv.org/abs/2301.09633
- https://arxiv.org/abs/2311.09476
- https://arxiv.org/abs/2609.20625
- https://arxiv.org/html/2506.07982v1
- https://arxiv.org/abs/2601.17087
- https://arxiv.org/html/2503.11926
- https://arxiv.org/html/2412.13678v1
- https://arxiv.org/html/2403.12173
- https://arxiv.org/html/2601.15812v2
- https://arxiv.org/pdf/2406.12045
- https://www.alphaxiv.org/abs/2607.16387
- https://alphaxiv.org/abs/2607.28802
- https://www.alphaxiv.org/overview/2509.25370v1
