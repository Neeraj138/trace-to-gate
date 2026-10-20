---
date: "2026-10-20"
type: review
status: proposal
topic: "Design gap review of the TraceToGate system design with the known-unknowns framework"
reviews: "04b-final-architecture-tracetogate-foundations.md (revision 2)"
inputs:
  - "03-research-tracetogate-foundations.md"
  - "03a-research-addendum-production-trace-to-eval.md"
  - "trace-to-gate-brief-and-discussion.md"
repos:
  - identifier: "github.com/sierra-research/tau2-bench"
    sha: "5ade54e282a4555e4d6c52d6b9b861ea8872afa6"
---

# TraceToGate Design Review: Known Unknowns

This review looks for what the system design (04b) does not cover before the build plan starts. It uses the four-quadrant known-unknowns matrix. The text follows ASD-STE100 Simplified Technical English.

**Evidence marks.** "Verified" means that we read it in the τ²-Bench source at commit `5ade54e`, or computed it from data in that clone. "Documented" means that a cited document states it. "Inference" means that it is our own reasoning. τ²-Bench paths are relative to the repository root.

**Abbreviations.** API: application programming interface. CI: confidence interval. DB: database. FWER: family-wise error rate. KK, KU, UK, UU: known knowns, known unknowns, unknown knowns, unknown unknowns. LLM: large language model. MDE: minimum detectable effect. OTel: OpenTelemetry. PII: personally identifiable information. RNG: random number generator. SD: standard deviation. TPR and TNR: true positive rate and true negative rate.

## Summary

- **Pipeline A can see the answer key.** The τ²-Bench reward is an L1 rule, an input to the router, and "hard evidence" for automatic promotion. The router, the validator, and the human reviewers also use privileged data: the original task, the gold path, and the fault events. So Pipeline A results do not transfer to production. We recommend a strict information boundary ("production mode") as a design rule now.
- **The reward rule can auto-promote broken tests.** "Rule evidence and v1 fails 2 of 4" is true for any task that fails often, because `reward = 0` counts as a rule. In the published GPT-4.1 runs, 30 of 114 Retail tasks fail in 2 or more of 4 trials, and 11 fail in all 4 (verified). Task defects and repeated simulator errors can therefore enter the blocking tier with no human step.
- **The grading path breaks with our wrapper.** τ²-Bench grades each run inside the runner with strict replay. Replay builds the environment from the domain registry, so it goes through our wrapped `get_response()` again. Any exception makes the runner run the full conversation again, up to 3 more times (verified). The design does not say how to switch off faults and spans during grading.
- **Three gate rules have unclear or unsafe meaning.** Blocking tests are v1 failures by construction, so an absolute "pass threshold" can FAIL a v2 that fixes most of them. "Any hard-safety flag in any v2 trial gives FAIL" nearly always fires when the rule has a small false-flag rate. The rule "a decisive metric depends on an LLM judge gives REVIEW" collides with τ²-Bench's own reward, which uses an LLM judge in 11 of the 40 test tasks (verified).
- **The gate has less power than the design states.** The design gives an MDE of about 8 points for 114 cases. The capability suite has only 40 tasks. Published runs give a paired-difference SD of 0.28–0.40, so the MDE on 40 tasks is about 13–18 points at 80% power (verified computation). The gate cannot detect a GPT-4.1 to GPT-4.1-mini downgrade on the test split (−5.6 points).
- **Data hygiene is not specified.** The design does not say which task split Pipeline A uses, so regression tests can come from the 40 capability tasks. Seeds for selection, validation, and the gate come from the same default list, so v1 gate trials can repeat the trials that selected the failure (verified seed derivation).
- **Human capacity is the main unknown.** One person is the L3 reviewer, the L4 taxonomy owner, the release owner, and the developer of v2. The design asks for two reviewers and 100–200 labels per failure mode. Neither is likely. Without them, no judge becomes validated and the automatic lanes stay empty.
- **The four pending proposals help.** Adopt production mode (extended) and funnel settings in the design now. Add implicit signals as routing-only signals, scored against the reward afterward. Defer clustering of review notes.

## The matrix at a glance

| | **We know the answer** | **We do not know the answer** |
|---|---|---|
| **We are aware of the item** | **Known knowns (section 1).** τ²-Bench structure and limits: one tool boundary, fresh DB per run, strict replay, reward mostly from the DB, NL judge in 40 tasks. Retail: 114 tasks, 74 train, 40 test. Large run-to-run variance (pass^1 about 74%, 60 of 114 tasks pass 4 of 4). LLM judges label well but locate steps poorly. No numeric gate standard. Cost about USD 0.07 per GPT-4.1 run. | **Known unknowns (section 2).** Models for the three roles. Seed support per provider. Budget. Langfuse and OTel convention pins. Simulator error rate under our setup. Judge accuracy before labels. Real power. Human reviewer time. Base rate of safety flags on v1. Converter yield. Fault recoverability. |
| **We are not aware of the item** | **Unknown knowns (section 3).** Facts that the field or our own research knows, but the design does not use: reward visible to Pipeline A, reward as hard evidence, grading replay through the wrapper, split and seed hygiene, the any-trial safety rule, NL judge inside the reward, reviewer blinding, solo roles, label noise and drift, judge self-preference, funnel caps, implicit signals, retries and idempotency, schema versions, shared RNG across threads. | **Unknown unknowns (section 4).** Cannot be listed. Mechanisms to surface them: pilot batch, A/A replicates, planted failures and canaries, chaos tests, red-team agents, audit samples, "other" bucket watch, sealed holdout, surprise log, pre-mortem. |

## 1. Known knowns

These are the facts and decisions that the design relies on. The column "Solid?" gives the evidence level. The last column flags items that look wrong or weakly supported.

| # | Fact or decision the design relies on | Solid? | Source | Flag |
|---|---|---|---|---|
| KK1 | All tool calls pass through `Environment.get_response()`, so one wrapper sees them all. | Verified | `src/tau2/environment/environment.py` L465; 03 §2 | Partly wrong. `set_state()` replay also calls `self.get_response()` (`environment.py` L293–411, call at about L390). The wrapper also runs during grading. See G03. |
| KK2 | Each run builds a fresh DB from `db.json`. `db.json` is 2.8 MB. | Verified | 03 §2; file size in clone | Hashing the full DB twice per write is affordable (Inference). |
| KK3 | The DB hash is SHA-256 of `json.dumps(sort_keys=True, default=str)`. | Verified | `src/tau2/utils/utils.py` L39–45 | `default=str` hides type changes. Use the same function for the live-state grade, or hashes do not match (Inference). |
| KK4 | Strict replay raises when a recorded write result differs. Default is strict. | Verified | `evaluator.py` L96; 03 §2 | The runner grades inline (`runner/simulation.py` L76) and retries any exception (`runner/progress.py` L19–100, `DEFAULT_MAX_RETRIES = 3` in `config.py` L40). The design does not handle this. See G03. |
| KK5 | Success is mostly a final-DB match. 40 of 114 tasks define NL assertions. | Verified | `evaluator_nl_assertions.py` L37–43; count from `tasks.json` | 11 of the 40 test tasks use the NL judge (`gpt-4.1`) inside the reward. This conflicts with section 5.3 ("outcome always from the DB") and with gate rule 7. See G09. |
| KK6 | Retail has 114 tasks: 74 train, 40 test. | Verified | `split_tasks.json` | The design does not say which split Pipeline A uses. See G06. |
| KK7 | Run-to-run variance is large. | Verified | GPT-4.1 published runs: 60 tasks pass 4 of 4, 11 fail 4 of 4, 43 are mixed; pass^1 0.741 | Strong. |
| KK8 | MDE is about 8 points with SD 0.3 and 114 cases. | Computation correct, basis wrong | 04b §13, D.3 | The gate runs 40 capability tasks plus N regression tests, not 114. On 40 tasks, the MDE is 13–18 points. See G08. |
| KK9 | Matched seeds make both arms see the same conditions. | Weak | 03 §2 (not verified) | Seeds are best-effort at OpenAI, and the Responses API has no seed ([OpenAI cookbook](https://developers.openai.com/cookbook/examples/reproducible_outputs_with_the_seed_parameter); [forum](https://community.openai.com/t/support-for-seed-parameter-in-the-responses-api/1230489)). The Anthropic Messages API has no seed parameter (Inference from its API reference). τ²-Bench derives one seed list per trial index for all tasks (`runner/batch.py` L742–743). See G07. |
| KK10 | The user simulator makes critical errors in about 12% of Retail conversations. | Documented, small sample | τ²-bench paper: 6 of 50 conversations, GPT-4.1 as agent and user | The rate for our agent and user models is unknown. See KU7. |
| KK11 | L1 rules are exact. | Weak for one rule | `policy.md` L16: "obtain explicit user confirmation (yes)" | "Write without confirmation" needs a reading of free user text. It is a heuristic, not an exact rule. Score its TNR before you trust it as hard evidence. |
| KK12 | `transfer_to_human_agents` is a sign of failure (pending proposal 2). | Partly wrong | `domains/retail/tools.py` L731–745; `tasks.json` | The tool is policy-sanctioned. 4 tasks expect it, 2 of them in the test split. It is not a write tool (`toolkit.py` L82–83). The GPT-4.1 runs call it 17 times in 456 runs. |
| KK13 | A run with full text is 50–200 KB of spans. | Estimate, plausible | GPT-4.1 published runs: 28 KB of JSON per run, 26.6 messages on average | Spans add overhead, so the range is plausible (Inference). |
| KK14 | LLM judges reach κ 0.77 with a fixed taxonomy and few-shot examples; step localization is 5–46%. | Documented | MAST, TRAIL, Who&When; 03 §8 | Strong. The design uses it correctly. |
| KK15 | No source auto-promotes an LLM-judged failure to a blocking suite. | Documented (absence) | 03a §5 | The design's rule-based auto-promotion is a research choice. G02 shows a hole in it. |
| KK16 | Langfuse accepts OTLP over HTTP only, and has annotation queues with no rule-based routing. | Documented | 03 §5; 03a §1.2 | Routing must be our code. The design already does this. |
| KK17 | One GPT-4.1 Retail run costs about USD 0.058 (agent) plus USD 0.013 (user). | Verified from data | GPT-4.1 published runs, mean `agent_cost` and `user_cost` | Prices change. One gate of 800 runs is then about USD 57 before judge calls (Inference). |

## 2. Known unknowns

Each item is a question that we know is open. "Blocks" names the work that cannot finish without an answer. "Default" is what the build plan uses if the question stays open.

| # | Question | Why it matters | Blocks | How and when to resolve | Default if unresolved |
|---|---|---|---|---|---|
| KU1 | Which LLMs for agent v1 and v2, the user simulator, the L2 judge, and the converter? | Cost, variance, and judge self-preference depend on it ([arXiv 2404.13076](https://arxiv.org/abs/2404.13076)). | All runs | Build plan step 0. Pick dated model snapshots, not aliases. | Agent v1 GPT-4.1 (dated snapshot), user simulator GPT-4.1 (as in published runs), judge from a different model family than the agent. |
| KU2 | Do the chosen providers honor `seed`? | Paired seeds reduce variance only if they work. Same seeds can also make reproduction self-fulfilling. | Trial design, G07 | Spike S2: run 5 tasks × 5 repeats with the same seed. Count identical transcripts. Record `system_fingerprint` where it exists. | Assume seeds do not work. Keep pairing for bookkeeping only. Base the power analysis on unpaired variance. |
| KU3 | What is the API budget per month, per gate, and per Pipeline A batch? | One gate is about USD 57, and Pipeline A adds 8 runs per failing run (Inference from KK17). | Scale of every phase | Owner decision before the outline. | A hard cost cap per batch, with a stop (see G13). |
| KU4 | Which Langfuse version, and does its API export annotation-queue scores with the reviewer and the observation ID? | Label export and the label contract depend on it. | Label export, L3 | Spike S3 in phase 1: create a queue item by API, score it, export it. | Pin the current v3 release of the Compose file. If export fails, store labels in a local review CLI instead. |
| KU5 | Which identifier pins the OTel GenAI conventions? | The new `semantic-conventions-genai` repository has no schema URL yet (03 §4). | Trace contract | Phase 1. | Pin the repository commit SHA in `ttg.semconv.version`. |
| KU6 | Who emits `chat` spans for the agent and for the user simulator? Does LiteLLM's callback add spans? | If a global LiteLLM callback is on, it also traces simulator and judge calls, and the span count changes. | Completeness check | Spike S1 (trace completeness, the brief's A-01). | The agent emits its own `chat` spans. The LiteLLM OTel callback stays off. Simulator calls get their own spans under a `ttg.role=user_sim` attribute. |
| KU7 | What is the simulator error rate with our agent and user models? | It sets the noise in reproduction and the size of the "simulator fault" queue. | Router thresholds, gate noise | Pilot P1: label 50 runs by hand for simulator faults. | Assume 12% critical (paper value). |
| KU8 | How accurate is the L2 judge before human labels exist? | Unvalidated judges cannot feed automatic lanes. | Automatic share | Phase 3, after 100+ human labels. | All L2 output goes to humans. Report automatic share as 0%. |
| KU9 | What are power and MDE with our real variance? | Margins below the MDE make the gate a coin toss. | Gate margins | Pilot P2 and A/A replicates (section 4). | Use published data: SD 0.3–0.4. MDE about 13–18 points on 40 tasks. |
| KU10 | How many reviewer hours per week are available? | Taxonomy needs about 100 failing traces. Judge validation needs 100–200 labels per mode (03a §2.3). | L3, L4, judge validation | Owner decision. Time the first 30 reviews in pilot P1. | 3 hours per week. Validate at most 2 failure modes. |
| KU11 | What is the base rate of hard-safety flags on v1? | The any-trial FAIL rule depends on it (G05). | Gate rule 2 | Pilot P2: run L1 on 400 v1 trials. | Assume more than 0. Use a comparative rule. |
| KU12 | How many traces become valid tests with the fixture-blind converter? | It sets dataset size and the share of `privileged` tests. | Dataset size | Phase 4 metric. | Expect a low yield. Keep the privileged fallback. |
| KU13 | Which injected faults can a correct agent recover from? | The live-state grade punishes a correct agent when the gold DB cannot be reached (G10). | Fault profiles, grade rule | Spike S4: run the gold path under each fault kind. | Use only faults where the gold path still succeeds with one retry or one check. |
| KU14 | How many failing runs can Pipeline A make, given the train split and a budget? | The regression suite size sets power and cluster count (G18). | Gate design | Pilot P1. | 74 train tasks × 4 trials gives about 300 runs and about 80 failures (Inference from pass^1 0.74). |
| KU15 | When do the chosen model snapshots retire? | v1 must stay runnable for every gate in the project. | Gate reruns | Check provider deprecation pages at build start. | Archive the v1 gate evidence. Plan a baseline change if a snapshot retires. |

## 3. Unknown knowns

These are things that the project or the field already knows, but the design does not use or state. Each one maps to a gap ID in section 5.

### 3.1 Information leaks from the answer key (G01, G02)

- **The reward is visible to Pipeline A.** L1 includes "τ²-Bench reward = 0" (04b §4.3), and the router uses "reward = 1" for `AUTO_REJECT` (04b §4.4, C.9). The L1 outcome detectors are then scored against the same reward (04b D.1). This is circular: the rule is the truth. Section 8 says "production has no correct DB", but the pipeline uses it.
- **Other privileged inputs** are less visible:
  - The router's reproduction check reruns the *original τ²-Bench task* before the converter runs (04b §4.1, step 6). Production cannot rerun a customer's conversation. Reproduction must use the converted test.
  - The validator's "gold path passes" check uses the task's gold actions. Production has no gold path.
  - Human reviewers see `ttg.fault.injected` events in Langfuse. Their labels then encode the injected cause, and L2 is scored against labels that saw the answer (Inference).
- **Assessment of proposal 1 ("production mode").** Adopt it, but widen it. Define one table that says which component may read the reward, the fault manifest, the gold actions, and the task text. Only the ground-truth scorer, the grader, and Pipeline B may read them. Run Pipeline A in production mode by default. Keep an "oracle mode" only to measure how much the leaks help.
- **Reward as hard evidence for promotion.** "A fixed rule found the failure" includes `reward = 0`. So every reproducible task failure is auto-promoted to blocking. Task defects reproduce well, because they do not depend on the agent. τ²-Bench stores known issue simulations for tasks 4, 5, and 7 (`data/tau2/domains/retail/task_issues/`). The "every agent fails every trial" flag runs only at validation, where only v1 runs, so it does not catch these. Restrict hard evidence to behavior rules that point at a step.

### 3.2 Grading path and runner behavior (G03, G19, G20)

- The evaluator builds both the predicted and the gold environments with `registry.get_env_constructor(domain)` (`evaluator.py` L158, L173, L198; `evaluator_env.py` L85, L97). For domain `retail_ttg`, that is our wrapper. Replay then injects faults again and emits spans again (Inference from verified code).
- `run_simulation()` grades inline (`runner/simulation.py` L76). `run_with_retry()` catches any exception and reruns the whole simulation (`runner/progress.py` L19–100). With 3 retries, one faulted run can cost 4 conversations and still end as a failed placeholder.
- Each attempt gets a new `uuid4` simulation ID (`runner/batch.py`, `run_single_task`). Failed attempts can leave spans in `evidence/raw`. The design has no attempt ID and no rule for which attempt counts.
- `run_domain` sets the global `random` seed (`runner/batch.py` L742). If the fault injector uses the global `random` module, 3 threads share one RNG, and fault placement is not reproducible (Inference).

### 3.3 Split, seed, and selection hygiene (G06, G07, G26)

- Pipeline A can draw failures from the 40 test tasks. Then the regression suite duplicates the capability suite, and the developer of v2 tunes on the test split. This is adaptive overfitting of a public test set (Goodhart's law).
- Selection uses v1 trials. If the gate reuses the same seeds for v1, and if seeds work, v1 fails again by construction. v2 gets an advantage that is not real (Inference: selection bias).
- Nothing prevents the v2 developer from copying test scenarios into the v2 prompt or few-shot examples. That leaks the test into the agent.

### 3.4 Gate rules (G04, G05, G09, G17, G18)

- **Blocking tests are v1 failures.** Anthropic's "graduated" cases are tasks that the agent already passes near 100% (03 §6). In 04b, blocking tests are tests that v1 fails at least 2 of 4 times. So "falls below its pass threshold" has no clear meaning. A v2 that fixes 9 of 10 targets but not the tenth can FAIL. A test changes role when a release fixes it: it is a fix target first, and a regression guard later. The design does not have this life cycle.
- **Any-trial safety FAIL.** With a per-trial false-flag rate *f* and *N* v2 trials, P(at least one flag) = 1 − (1 − *f*)^*N*. With *f* = 0.5% and *N* = 400, that is 87% (Inference). The null pair would then FAIL. The rule must compare v2 with v1.
- **LLM judge inside the reward.** 11 of 40 test tasks include the NL-assertion judge in the reward (verified). Gate rule 7 then gives REVIEW for almost every gate, or the design must exempt it.
- **Many rules, one false FAIL each.** The gate has at least 5 FAIL paths over 2 suites, 2 kinds, and 2 tiers. Each has its own false alarm rate, so the FWER is higher than any one rule (Inference). One null pair cannot measure this. It needs several A/A replicates.
- **Few clusters.** Regression tests cluster by source task. If 60 tests come from 20 tasks, the cluster bootstrap has 20 clusters. Bootstrap CIs with few clusters are too narrow (standard statistics; Inference for this case).

### 3.5 Grading under faults (G10)

The live-state grade compares the final DB with the gold DB. Some faults make the gold DB unreachable for a correct agent, for example a write that errors every time. The correct action then is to tell the user or to transfer. The grade then marks correct behavior as a failure (Inference).

### 3.6 Humans and labels (G12, G15, G16)

- **Solo roles.** The brief describes one person (brief, "Motivation"). That person is L3, L4, the release owner, and the v2 developer. The design asks for two reviewers on a sample (04b §4.3). A solo project cannot measure inter-reviewer agreement. Test-retest agreement (the same person relabels a blind sample after two weeks) is a practical substitute.
- **Label noise and criteria drift** are documented ([EvalGen](https://arxiv.org/abs/2404.12272)). Labels are the truth for L2, so label noise caps the measurable judge accuracy.
- **Label budget.** 100–200 labels per failure mode (03a §2.3) for 5–10 modes is 500–2,000 labels. That is not realistic for one person. The Langfuse example gate used 35 items (03a §1.2).
- **The label line (C.4) is too thin.** It has no free-text note (open coding needs it), no first-failure step, no simulator-fault or task-defect flags, no queue reason, and no flag for whether the reviewer saw the reward. Without the queue reason, audit error rates cannot be computed without bias.

### 3.7 Judges over time (G24)

- Judges prefer outputs from their own model family ([arXiv 2404.13076](https://arxiv.org/abs/2404.13076)). The design does not separate judge and agent families.
- Providers change model behavior under one name ([arXiv 2307.09009](https://arxiv.org/abs/2307.09009)). Rogan-Gladen correction per arm assumes the same TPR and TNR for both arms (03 §11). The design does not state this assumption or check it.

### 3.8 Funnel and signals (G13, G14)

- **Funnel settings (proposal 3).** Production practice runs rules on all traces, judges on a sample, and sends a capped stream to humans (03a §1.3). Examples: Datadog caps queues at 5% and 1,000 items, and Confident AI caps at 200 traces per day. LangSmith pauses judges at a weekly cost cap. The design has no audit rate, no confidence threshold, no queue cap, and no cost cap. Adopt the proposal. At this volume, run L2 on 100% of runs, but keep the knobs so that the design carries to production.
- **Implicit signals (proposal 2).** Production teams use user signals (03a §2.1: Zapier, Cursor, Dropbox). The design has none, so its "production" story depends on the reward. Adopt them as routing signals only. They must never be hard evidence. Three τ²-Bench facts limit them:
  - Transfer is sometimes correct (KK12).
  - The "user" is an LLM, so frustration and repeats are simulated. Simulators are poor proxies for real users ([arXiv 2601.17087](https://arxiv.org/abs/2601.17087)).
  - Conversations are short: 26.6 messages on average and 34 at the 90th percentile in the GPT-4.1 runs (verified). Set "long conversation" relative to this distribution.
- The useful research result is the precision and recall of each implicit signal against the reward, measured in oracle mode. That number shows what a production team would see.

### 3.9 Data contracts and operations (G11, G21, G23, G25)

- **Gate report identity is incomplete.** C.7 names v1, v2, and the dataset. It does not name the user simulator model, the judge models and versions, the NL-assertion model, the τ²-Bench commit, the fault profile versions, or the gate config hash. Each one changes results. Agent success varies up to 9 points across user models ([arXiv 2601.17087](https://arxiv.org/abs/2601.17087)).
- **No schema versions.** `RunRecord`, `EvalCase`, label lines, and gate reports have no `schema_version`. "Manifest hash = SHA-256 of file bytes" needs canonical serialization: sorted keys, fixed float format, and LF line ends.
- **Cost accounting.** The gate cost ratio must count agent cost only, not simulator or judge cost. LiteLLM can return no price for a new model, and the cost then reads as 0 (Inference). Treat a missing price as an error.
- **Order inside one turn.** The normalizer orders steps by parent ID and `ttg.turn_idx`. Several tool calls in one assistant message probably share one turn index (Inference). Add a sequence number per tool call.

### 3.10 Later concerns (G27, G28, G29, G30)

- **Clustering of notes (proposal 4).** An LLM can propose groups after a human has coded 30–50 traces, but LLMs cluster by surface similarity (03a §4.1). Useful later, and low risk if a human decides. Defer.
- **PII path.** Raw evidence is "never deleted" (Rule 2). With real users, erasure requests conflict with this. Redaction must run before the first write to `evidence/raw`, not only before export.
- **Baseline rotation.** Gate runs of v2 feed Pipeline A. The replayable check says "fails on v1". For a v2 trace, it must say "fails on the source version". After a release, v2 becomes the baseline.
- **Model retirement** (KU15).

## 4. Unknown unknowns

We cannot list these. We can add mechanisms that make them show up early. Each mechanism names when it runs.

| Mechanism | What it surfaces | When |
|---|---|---|
| **Pilot batch P1**: 20 tasks × 4 trials on the train split, full pipeline, all output read by a human | Contract gaps, span loss, cost per run, reviewer time per trace, surprise failure kinds | Before Pipeline A is built past ingest |
| **Pilot P2**: one gate on v1 versus v1 | Real variance, safety-flag base rate, gate runtime | Before margins are set |
| **A/A replicates**: at least 5 null-pair gates, with new seeds each time | The real false FAIL and false REVIEW rate of the whole rule list | Before the first real gate result is trusted |
| **Planted failures (canaries)**: one known-bad agent per failure mode, and synthetic traces with a known label put into the review queue | Detector and reviewer miss rates; router errors | Each Pipeline A release |
| **Judge canary set**: 30 frozen labelled traces re-scored by each judge version | Judge drift after a silent provider change | Each batch |
| **Chaos tests on the pipeline**: kill the process in a batch, stop Langfuse, fill the disk, force API rate limits, corrupt one span file | Idempotency, retry, and completeness bugs | Phase 1 and after each ingest change |
| **Red-team agents**: an agent that always transfers, one that asks for confirmation twice, one that writes text that games the NL judge, one that is cheaper but worse on rare tasks | Gate rules that a bad agent can pass | Before the gate is called done |
| **Audit samples** of `AUTO_REJECT`, `AUTO_CANDIDATE`, and auto-promoted tests | Silent errors in automatic lanes | Continuous, at a fixed rate |
| **"Other" bucket watch**: the share of `other` labels and of `other` L2 output per batch | New failure modes; taxonomy decay | Each batch |
| **Sealed holdout**: a set of runs or seeds that no one looks at until the final report | Overfitting to the regression and capability suites | Final report only |
| **Surprise log**: one line for each result that nobody expected, with a link to evidence | Patterns of unknown unknowns | Continuous |
| **Pre-mortem review** at each phase end (below) | Risks that the team does not want to say aloud | Each phase end |

### 4.1 Pre-mortem

It is April 2027. The project failed, or its results were misleading. These are the most likely reasons, each with a mitigation.

| # | Why it failed | Mitigation | Gap |
|---|---|---|---|
| PM1 | Pipeline A looked good only because it read the reward. The "production" claims were false. | Production mode. Report oracle-mode results only as a comparison. | G01 |
| PM2 | The blocking tier filled with task defects and simulator errors. The gate gave false FAILs, and the owner stopped trusting it. | Remove reward as hard evidence. Audit auto-promotions. | G02 |
| PM3 | Faulted runs crashed in grading, retried 4 times, and the budget ran out in the first month. | Grading mode in the wrapper. Spike S5. Cost cap. | G03, G13 |
| PM4 | Every gate said FAIL (any-trial safety rule) or REVIEW (NL judge rule). The gate never gave information. | Comparative safety rule. Separate DB-only and NL components. | G05, G09 |
| PM5 | The gate said PASS for a weaker model, because the effect was below the MDE. | State the MDE in each report. Give REVIEW, not PASS, when the CI is wider than the margin. | G08 |
| PM6 | The v2 "fix" looked good because the regression tests came from the test split and the v1 trials reused the selection seeds. | Train split for Pipeline A. Separate seed namespaces. Sealed holdout. | G06, G07 |
| PM7 | The solo reviewer ran out of time. No judge became validated, so every run went to the queue and the backlog grew. | Funnel caps. Validate few modes. Plan reviewer hours. | G13, G16 |
| PM8 | Labels were biased, because the reviewer saw the reward and the fault events. Judge accuracy was overstated. | Blind review view. Test-retest sample. | G15 |
| PM9 | Results could not be reproduced, because the gate report did not name the user simulator or the judge versions. | Full identity block in the gate report. | G11 |
| PM10 | A provider retired or changed a model in the middle of the project. Old and new numbers did not compare. | Dated snapshots. Judge canary. Archive evidence. | G24, G30 |
| PM11 | A schema change broke old manifests and hashes. Old gate reports could not be checked again. | Schema versions. Canonical serialization. | G21 |
| PM12 | The tests passed on τ²-Bench, but the implicit signals had low precision. Nothing transferred to a real agent. | Measure each implicit signal against the reward and report it. | G14 |

## 5. Gap list

Impact: H = the result can be wrong or the build stops; M = the result is weaker or costs more; L = quality or future work. "Design now" means that 04b must change before the outline. "Build plan" means that the outline holds it as a spike or a phase. "Later" means after the first gate works.

| ID | Gap | Quadrant | Impact | Proposed design change | Where in 04b | Resolve in |
|---|---|---|---|---|---|---|
| G01 | Pipeline A reads the answer key: reward in L1 and router, original task in reproduction, gold path in the validator, fault events in review | UK | H | Add production mode as a design rule. Add an access table: the reward, the fault manifest, the gold actions, and the task text go only to the ground-truth scorer, the grader, and Pipeline B. Reproduction runs on the converted test, after the converter. "Gold path passes" becomes an oracle-only metric. Keep oracle mode for comparison. | §4.1, §4.3, §4.4, §4.5, §6.1, §8, C.9 | Design now |
| G02 | `reward = 0` counts as hard rule evidence, so reproducible task failures (defects, simulator errors) auto-promote to blocking | UK | H | Hard evidence = a behavior rule with a step reference only (for example, duplicate write, write without confirmation). Outcome-only failures need a human to promote. Run a second agent version on each auto-promoted test before it becomes blocking. | §4.3, §4.6, C.9 | Design now |
| G03 | Grading replay goes through the wrapper. Inline strict grading plus the retry loop reruns faulted conversations. | UK | H | Wrapper has a grading mode (no faults, no spans) that the evaluator path uses. For faulted runs, skip τ²-Bench inline grading and use only the live-state grade. Spike S5 confirms this before any fault work. | §4.2, §5.3, C.8, B.1 | Design now (rule); build plan (spike S5) |
| G04 | Blocking tests are v1 failures, so "pass threshold" has no clear meaning | UK | H | Add a test life cycle: `fix_target` (the baseline fails) and `guard` (the baseline passes). Gate FAIL on guards uses a paired drop against v1. Fix targets are reported as fixed or not fixed and never FAIL. A release that ships turns fixed targets into guards. | §4.6, §5.5, C.5 | Design now |
| G05 | Any hard-safety flag in any v2 trial gives FAIL | UK | H | FAIL when the v2 safety-flag rate is higher than v1 beyond a margin, or when a flag that a human confirmed appears on a guard test. Score each safety rule's TNR on v1 before it can gate. | §5.5 | Design now |
| G06 | No split rule. Regression tests can come from the capability split. | UK | H | Pipeline A runs on the 74 train tasks only. The capability suite is the 40 test tasks. Keep a sealed holdout of seeds or personas for the final report. | §4.1, §5.1, §6 | Design now |
| G07 | Selection, validation, and gate seeds come from one list | UK | M | Give each stage its own seed namespace (selection, reproduction, validation, gate). Never reuse a selection seed in a gate. Record the namespace in each run. | §5.2, C.3 | Design now |
| G08 | The MDE statement uses 114 cases, but the capability suite has 40 | KK (wrong) | H | Correct §13 and D.3: MDE 13–18 points on 40 tasks with SD 0.3–0.4. Each gate report states its MDE. Margins cannot be smaller than the MDE. When the CI is wider than the margin, the result is REVIEW. | §13, D.3, C.7 | Design now |
| G09 | The τ²-Bench reward uses an LLM judge in 11 of 40 test tasks, which collides with gate rule 7 | UK | M | Report the DB part and the NL part of the reward separately. The gate decides on the DB part. The NL part is a judge metric, and it can only give REVIEW. Correct §5.3 ("always from the DB"). | §5.3, §5.5, C.8 | Design now |
| G10 | The live-state grade punishes correct behavior when a fault makes the gold DB unreachable | UK | M | Each fault profile states if it is recoverable. Faulted tests use only recoverable profiles, or carry an expected outcome per profile. Spike S4 checks the gold path under each fault. | §4.2, §5.3, C.2, C.8 | Design now (rule); build plan (S4) |
| G11 | The gate report and the run record do not name the user simulator, judge versions, NL-assertion model, fault profile versions, or config hash | UK | M | Add an identity block with these fields to C.3 and C.7. | C.3, C.7, §12.3 | Design now |
| G12 | The label line lacks note, step, flags, queue reason, and blind flag | UK | M | Extend C.4 with `note`, `first_failure_step`, `is_sim_fault`, `is_task_defect`, `queue_reason`, `saw_oracle`, and a Langfuse score ID for export without duplicates. | C.4 | Design now |
| G13 | No funnel settings: audit rate, confidence thresholds, queue cap, cost cap (proposal 3) | UK | M | Add a funnel config: L1 on 100%; L2 on 100% at this volume, with a sampling knob; audit 10% of each automatic lane; human queue cap per day; hard cost cap per batch and per gate that stops the run. Put the values in config and in each report. | §4.4, §12.1, new config section | Design now (knobs); build plan (values) |
| G14 | No implicit signals (proposal 2) | UK | M | Add soft L1 signals: transfer when the task does not need it, repeated user request, conversation length above the 90th percentile, user frustration (judge-based). Routing only, never hard evidence. Score their precision and recall against the reward in oracle mode. | §4.3, §8, D.1 | Design now (contract); build plan (phase 3) |
| G15 | Solo roles; no reviewer blinding; inter-reviewer agreement not possible | UK | M | State the roles that one person holds. Give the reviewer a blind view without reward and fault events. Replace two-reviewer agreement with test-retest on 10% of labels after at least 2 weeks. | §4.3, B.3, D.1 | Design now |
| G16 | The label budget for judge validation is not realistic for one person | KU | M | Validate at most 2–3 failure modes. Report TPR and TNR with CIs. Set the "validated" bar together with a minimum label count. | §4.3, §6.2, D.2 | Build plan |
| G17 | The gate's false FAIL rate across all rules is not controlled. One null pair cannot measure it. | UK | M | Run at least 5 A/A gates to estimate the rate for the whole rule list. Set margins so that the total false FAIL rate is at most a set target, for example 5%. | §5.5, §6.3 | Build plan |
| G18 | Few clusters in the regression suite make the cluster bootstrap too narrow | UK | M | State a minimum cluster count (for example, 20 source tasks). Below it, use a wider method or give REVIEW. | §5.4, D.3 | Build plan |
| G19 | Retried attempts and resumed batches leave partial evidence. No attempt ID. | UK | M | Add `attempt` to the run identity and a "used" flag. The normalizer keeps only the used attempt, and it counts the others as cost. | C.1, C.3, E.1 | Design now |
| G20 | A shared global RNG across 3 threads makes fault placement not reproducible | UK | L | The fault injector uses its own RNG, seeded from run ID and fault profile. | B.1 | Build plan |
| G21 | No schema versions; manifest hash over bytes without canonical form | UK | M | Add `schema_version` to every contract. Define canonical JSON for hashed files. | C.3–C.7 | Design now |
| G22 | The "2 of 4" reproduction rule drops intermittent failures: P(kept) is 0.35 at a 30% failure rate and 0.69 at 50% | UK | M | State this bias. Let failures with 1 of 4 enter the candidate tier as `flaky`. Report pass^k drops on them. | §4.5, §4.6 | Build plan |
| G23 | Cost accounting is not defined for the gate cost ratio | UK | L | The cost ratio counts agent cost only. A missing price is an error. | §12.1, C.7 | Build plan |
| G24 | Judge drift and self-preference are not handled | UK | M | The judge is from a different model family than the agent. Re-score the judge canary set each batch. Re-validate when the model or prompt changes. Check that TPR and TNR are similar for both arms before Rogan-Gladen correction per arm. | §4.3, §6.2, D.2 | Build plan |
| G25 | No order key for several tool calls in one turn | UK | L | Add `ttg.tool.seq` per tool call. | C.1, B.2 | Build plan |
| G26 | v2 can leak test content into its prompt | UK | L | Rule: no case text in agent prompts. Check n-gram overlap between prompt and manifest in the gate. | §6.3, §11 | Build plan |
| G27 | No clustering help for open coding (proposal 4) | UK | L | Optional LLM clustering of notes, after 30–50 human-coded traces. A human accepts or rejects each group. | §4.3 (L4) | Later |
| G28 | Raw evidence immutability conflicts with PII erasure for real traces | UK | L | Redact before the first write to `evidence/raw`. Allow erasure of whole runs with a tombstone record. | §7.1, §12.4 | Later |
| G29 | "Fails on v1" breaks for traces from v2 and after a baseline change | UK | L | Use "fails on the source version". Define baseline rotation after a release. | §4.5, §5.6 | Later |
| G30 | Model retirement can stop v1 reruns | KU | L | Use dated snapshots. Archive gate evidence. Note the retirement date in the report. | §9, §13 | Build plan |

## 6. Recommendation

### 6.1 Fix in the design before the outline

These gaps change contracts or the meaning of results. If the outline starts first, the build will encode them.

1. **G01 Production mode and access table.** This is proposal 1, widened to cover reproduction, the validator, and reviewer views.
2. **G02 No auto-promotion on outcome alone.**
3. **G03 Grading mode for the wrapper**, and the rule that faulted runs skip inline τ²-Bench grading.
4. **G04, G05, G08, G09 Gate semantics.** Add the test life cycle, the comparative safety rule, the corrected MDE, and the separated DB and NL reward parts.
5. **G06, G07 Split and seed hygiene.**
6. **G10 Recoverable fault profiles** (rule only; S4 confirms).
7. **G11, G12, G19, G21 Contract fields**: identity block, label line fields, attempt ID, schema versions.
8. **G13, G14, G15 Funnel knobs, implicit signals as routing-only signals, roles and blind review.** This adopts proposals 2 and 3.

These changes are mostly text and contract changes. They add little build work, but they prevent most of the pre-mortem causes.

### 6.2 Put in the outline as spikes and phases

- **Spike S1** trace completeness (KU6). **Spike S2** seed honoring (KU2). **Spike S3** Langfuse label export (KU4). **Spike S4** gold path under each fault (KU13, G10). **Spike S5** grading with the wrapped domain (G03). Run S5 and S1 first.
- **Pilot P1** on the train split (KU7, KU10, KU14). **Pilot P2** v1 versus v1 gate (KU9, KU11).
- **A/A replicates** and margin calibration (G17), cluster minimum (G18), and flaky tier (G22) in the gate phase.
- **Judge phase**: scope to 2–3 modes (G16), judge canary and family separation (G24).
- Smaller items in the phase where they belong: G20, G23, G25, G26, G30.
- The unknown-unknown mechanisms in section 4 go into the outline as recurring tasks, each with a phase.

### 6.3 Can wait

- **G27** clustering of notes (proposal 4). Low value until 30–50 traces are coded by hand.
- **G28** PII and erasure. Needed only for real traces.
- **G29** baseline rotation. Needed only after the first release decision.

The separate review of decision models is out of scope here.

## Sources

**τ²-Bench at `5ade54e`** (verified in the local clone): `src/tau2/environment/environment.py` (L275, L293–411, L465); `src/tau2/evaluator/evaluator.py` (L96, L158, L173, L198); `src/tau2/evaluator/evaluator_env.py` (L85, L97); `src/tau2/evaluator/evaluator_nl_assertions.py` (L37–43); `src/tau2/runner/simulation.py` (L76); `src/tau2/runner/progress.py` (L19–100); `src/tau2/runner/batch.py` (L344–470, L520–700, L742–743); `src/tau2/config.py` (L6–40); `src/tau2/utils/utils.py` (L39–45); `src/tau2/environment/toolkit.py` (L65–88); `src/tau2/domains/retail/tools.py` (L731–745); `data/tau2/domains/retail/{policy.md, tasks.json, split_tasks.json, db.json, task_issues/}`; `data/tau2/results/final/*_retail_*_4trials.json` (statistics computed for this review).

**Computed from published runs** (GPT-4.1, GPT-4.1-mini, o4-mini, Claude 3.7 Sonnet; Retail; 4 trials; seed 300). Per-task paired-difference SD against GPT-4.1 is 0.28–0.39 on 114 tasks and 0.30–0.40 on the 40 test tasks. The MDE at 80% power and α = 0.05 is 2.8 × SD / √n. An A/A split of GPT-4.1 trials gives an implied 4-trial SD of 0.21, which is an MDE of 9 points on 40 tasks. GPT-4.1-mini is 5.6 points below GPT-4.1 on the test split.

**External**: [τ²-bench paper](https://arxiv.org/abs/2506.07982); [OpenAI seed cookbook](https://developers.openai.com/cookbook/examples/reproducible_outputs_with_the_seed_parameter); [OpenAI forum: seed in the Responses API](https://community.openai.com/t/support-for-seed-parameter-in-the-responses-api/1230489); [OpenAI forum: seed deprecation](https://community.openai.com/t/is-the-seed-parameter-getting-deprecated/1363139); [LLM self-preference, arXiv 2404.13076](https://arxiv.org/abs/2404.13076); [EvalGen, arXiv 2404.12272](https://arxiv.org/abs/2404.12272); [Lost in Simulation, arXiv 2601.17087](https://arxiv.org/abs/2601.17087); [ChatGPT behavior drift, arXiv 2307.09009](https://arxiv.org/abs/2307.09009). Other claims cite 03 and 03a by section.
