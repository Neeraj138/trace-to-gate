---
type: assessment
flow: design-discussion
date: "2026-10-20"
status: proposal
topic: "Decision models (such as TypeSafe Jev) and small judges: do they fit Pipeline A of TraceToGate?"
relates_to: "04b-final-architecture-tracetogate-foundations.md"
---

# Decision Models in Pipeline A: Assessment

This document asks one question. Should Pipeline A of TraceToGate use a "decision model", such as Jev, in place of or next to the LLM judge? The user asked us not to add a tool for the sake of adding it. So a "no" was an acceptable result from the start.

The text follows ASD-STE100 Simplified Technical English. Each factual claim has a source link. The label **Inference** marks our own reasoning. The label **Vendor claim** marks a claim that only the seller makes and that nobody else has checked.

## Answer

- **Do not replace the L2 classifier with Jev.** The L2 task is "find the first failure in a long trace and give it one label". This task needs ordered, multi-step reading of 10k–50k tokens. Jev's own documents say that it is weak at exactly this: long noisy input, multi-hop questions, and the order of events.
- **The cost argument is weak at our volume.** Pipeline A sees hundreds to low thousands of runs. At that scale, an LLM judge costs a few dollars per batch. The agent and the user simulator cost much more (**Inference**, from section 12.1 of the design).
- **The "calibrated probability" claim is not proven.** TypeSafe publishes no calibration error figure. One independent test found poor calibration. So a Jev probability needs the same calibration step as an LLM confidence value.
- **The real gap is calibration, not the model.** The router uses a "confidence value" with thresholds, but the design does not say how to make that value trustworthy. We propose one change: calibrate every judge's confidence on human labels, and choose router thresholds from a target audit error rate. This change helps with any judge.
- **Only here, as an optional experiment:** the pending implicit-signal detectors (user frustration, rephrased requests). These are short, narrow, yes/no questions on the user turns only. Langfuse, our review tool, already has a Jev template for them.
- **Later, production only:** a cheap first-pass filter when the volume is millions of traces and the judge cannot run on every trace. This is outside the current τ²-Bench scope.

## 1. What a decision model is

### 1.1 Jev in short

**Who makes it.** TypeSafe AI makes Jev. TypeSafe is a San Francisco lab. Its founders are Diogo Almeida, Erik Gafni, and Sasha Sheng ([InfoQ](https://infoq.com/news/2026/10/typesafe-ai-jev-released)). TypeSafe calls Jev its first "System One Model" ([typesafe.ai](https://typesafe.ai/)).

**What it does.** Jev does not write text. You give it a "state" (the text to judge) and a set of typed questions. It returns one typed answer per question ([Opik TypeSafe integration](https://www.comet.com/docs/opik/integrations/typesafe)). There are three question types ([TypeSafe primitives](https://docs.typesafe.ai/primitives)):

| Question type | What it answers | What it returns |
|---|---|---|
| Noul | Is this yes/no statement true? | A probability from 0 to 1 |
| Choice | Which option from a fixed list fits? (2 to 255 options) | The option, a probability per option, and a confidence value |
| Score | Which level on an ordered scale fits? (2 to 10 levels) | The level, a probability per level, and a confidence value |

The limits on options and levels come from the [LangSmith decision-model docs](https://docs.langchain.com/langsmith/online-evaluations-decision-models). The output can never be outside the schema. But it can still be a wrong value inside the schema ([InfoQ](https://infoq.com/news/2026/10/typesafe-ai-jev-released); [KDnuggets](https://www.kdnuggets.com/what-everyone-is-getting-wrong-about-typesafe-ais-jev)).

**What is not known.** TypeSafe does not publish the model size, the architecture, or the training data ([KDnuggets](https://www.kdnuggets.com/what-everyone-is-getting-wrong-about-typesafe-ais-jev)). It names one training method, "Reinforcement Learning for Calibrated Decisions" (RLCD), with no further detail ([InfoQ](https://infoq.com/news/2026/10/typesafe-ai-jev-released)). One independent author says that Jev sits on a pretrained transformer ([Maximum Effort](https://maximumeffort.substack.com/p/jev-is-poorly-calibrated)).

**How you customize it.** You customize it only through the question text, the options, and their descriptions. The TypeSafe documents describe no fine-tuning and no few-shot examples ([TypeSafe primitives](https://docs.typesafe.ai/primitives); [TypeSafe docs index](https://docs.typesafe.ai/llms.txt)). So you cannot train it on our human labels. **Inference:** this removes one benefit of a small model. We cannot make it learn our taxonomy from our labels.

### 1.2 Claims and evidence

| Topic | What the sources say | Who says it |
|---|---|---|
| Calibration | "returns a calibrated probability" ([Opik changelog](https://www.comet.com/docs/opik/changelog/2026/10/5)). TypeSafe's confidence page gives no measured calibration figure. It says that the correct thresholds "depend on your domain" and advises you to "test with your own data" ([TypeSafe confidence](https://docs.typesafe.ai/confidence)). | Vendor claim, no figure |
| Calibration, independent | One test on 1,000 physics questions found a mean total variation of 0.518. A flat guess gave 0.546. The author found the answers "too peaky" and overconfident ([Maximum Effort](https://maximumeffort.substack.com/p/jev-is-poorly-calibrated)). | Independent, one blog post |
| Accuracy | About 68% on TypeSafe's own workflow test, where frontier models gave the reference answers ([KDnuggets](https://www.kdnuggets.com/what-everyone-is-getting-wrong-about-typesafe-ais-jev)). | Vendor claim |
| Accuracy vs an LLM | On 6,003 rubric checks, Jev agreed with Claude Fable 5.1 in 91.5% of cases. DeepSeek V4.1 Flash agreed in 93.5% ([Langfuse blog](https://langfuse.com/blog/2026-09-18-using-typesafes-jev-for-evals)). This is agreement with one model, not with humans. | Third party (Good Start Labs), reported by Langfuse |
| Speed | 70–500 ms per call. A user survey gave a median of 76 ms ([InfoQ](https://infoq.com/news/2026/10/typesafe-ai-jev-released)). "193.6x faster" ([typesafe.ai](https://typesafe.ai/)). | Vendor claim and user reports |
| Price | $0.042 per million input tokens. Output is free ([TypeSafe models](https://docs.typesafe.ai/models)). | Vendor price list |

Two points about the evidence:

1. The independent calibration test measures how well Jev copies a known probability distribution. It does not measure the calibration of yes/no labels on real data. So it is a warning, not a final verdict (**Inference**).
2. The headline chart on the TypeSafe home page shows $0.000081 vs $0.013880 and 0.114 s vs 8.566 s ([typesafe.ai](https://typesafe.ai/)). These figures give about 171x cheaper and 75x faster, not the headline 444.6x and 193.6x (**Inference**, our arithmetic).

### 1.3 Limits that matter for agent traces

TypeSafe documents these limits for `jev-1.13` ([Jev 1.13 jaggedness](https://docs.typesafe.ai/model-jaggedness/jev-1.13.md); [TypeSafe models](https://docs.typesafe.ai/models)):

- **Context.** The default limit is 32k tokens per question and 64k per request. A "danger zone" header raises it to 262k, but TypeSafe says that this is "slower and less accurate".
- **Context rot.** "Accuracy falls as the state grows with content unrelated to the decision."
- **Multi-step reasoning.** A question that "requires multiple hops of reasoning costs accuracy."
- **Order and time.** It reads dates as text and does not order them well.
- **Counting.** It "does not count reliably."
- **Option order.** It "leans toward the option that comes first."
- **Adversarial text.** It does not treat the state as hostile, so injected text can change answers.
- **No reasons.** Decision models "return typed verdicts, not written reasoning" ([Langfuse Jev docs](https://langfuse.com/docs/evaluation/evaluation-methods/jev-as-a-judge)).

**Inference:** one τ²-Bench run is about 50–200 KB of spans (design section 7.2). At about 4 bytes per token, this is about 12k–50k tokens. A rendered transcript is smaller than the raw span JSON, but many runs will still be near or above 32k tokens.

### 1.4 Availability, data, and lock-in

- **API only.** We found no open weights, no self-host option, and no license for the weights ([typesafe.ai](https://typesafe.ai/); [InfoQ](https://infoq.com/news/2026/10/typesafe-ai-jev-released)). TypeSafe is in early access ([Opik TypeSafe integration](https://www.comet.com/docs/opik/integrations/typesafe)).
- **Data.** TypeSafe says that it does not train on user data. It offers zero data retention only to enterprise customers ([TypeSafe legal](https://docs.typesafe.ai/legal.md)). So trace text leaves the machine, as it does with any hosted LLM judge.
- **Versions.** You can pin a version such as `jev-1.13.0`. The alias `jev-latest` moves ([TypeSafe models](https://docs.typesafe.ai/models)).
- **Platform support.** Langfuse has a "decision model evaluator" for Jev, with templates such as "Rate Customer Frustration" and "User Conversation Signal" ([Langfuse Jev docs](https://langfuse.com/docs/evaluation/evaluation-methods/jev-as-a-judge)). LangSmith supports Jev only in its user interface, not in its SDKs ([LangSmith decision models](https://docs.langchain.com/langsmith/online-evaluations-decision-models)). Opik rules can use Jev through OpenRouter ([Opik changelog](https://www.comet.com/docs/opik/changelog/2026/10/5)).
- **Lock-in.** Only TypeSafe serves Jev. **Inference:** the risk is small if we call it through our own judge interface and record the judge ID and version, as the design already does in `findings[]`.

### 1.5 Other small judges

| Option | What it is | Open weights? | Fit for Pipeline A (**Inference**) |
|---|---|---|---|
| Galileo Luna-2 | 3B and 8B evaluators built on fine-tuned Llama models. Customer tuning needs about 4,000 labelled samples. Up to 128k tokens. Enterprise tier only, with an on-prem option ([Galileo docs](https://docs.galileo.ai/concepts/luna/luna)). F1 0.95 at $0.02 per million tokens (**Vendor claim**, same page). | No | Needs about 4,000 labels and an enterprise contract. Too heavy for this project. |
| Patronus Lynx | 8B and 70B models that detect hallucination against a given document. Returns PASS or FAIL with reasoning ([Lynx paper](https://arxiv.org/html/2407.08488v2); [Patronus blog](https://patronus.ai/blog/lynx-state-of-the-art-open-source-hallucination-detection-model)). | Yes | Wrong task. Our failures are policy and tool-use failures, not retrieval faithfulness. |
| Patronus Glider | About 3.8B, built on Phi-3.5-mini. Gives a score with reasons and highlighted phrases ([Glider paper](https://arxiv.org/pdf/2412.14140); [Hugging Face](https://huggingface.co/PatronusAI/glider)). The hosted copy shows a non-commercial licence ([featherless.ai](https://featherless.ai/models/PatronusAI/glider)). | Yes | Possible local judge. No evidence on agent traces. |
| Atla Selene Mini | 8B judge built on Llama-3.1-8B, Apache-2.0 ([featherless.ai](https://featherless.ai/models/AtlaAI/Selene-1-Mini-Llama-3.1-8B/readme); [Atla](https://www.atla-ai.com/post/selene-1-mini)). Beats GPT-4o-mini on 11 benchmarks (**Vendor claim**). | Yes | Possible local judge, so no data leaves the machine. No evidence on agent traces. |
| Raindrop rd-signal-2 | Binary classifiers per issue, built from production traces. Code first filters the traces, and a model judges only the unclear cases. Precision 71%, recall 87%, median 100 ms ([Raindrop blog](https://raindrop.ai/blog/signals-2-frontier-classification/)) (**Vendor claim**). | No, hosted service | Good pattern, not a component. It confirms "code first, model only for the unclear rest", which our L1-then-L2 order already does. |
| Own fine-tuned small classifier | A small model trained on our own labels. | Our choice | Needs hundreds of labels per failure mode. We will not have them in the first phase. |

### 1.6 Calibration methods for any judge

A confidence value is only useful for thresholds when it matches the real error rate. LLMs that state their own confidence "tend to be overconfident" ([Xiong et al.](https://arxiv.org/abs/2306.13063)). So the design needs a calibration step, whatever judge it uses.

- **Platt scaling and isotonic regression** map a raw score to a calibrated probability. You fit them on a labelled set that the judge did not see ([scikit-learn calibration guide](https://scikit-learn.org/stable/modules/calibration.html)).
- **Conformal methods** give a guarantee on the error rate of the cases that you accept, for a given confidence level ([Angelopoulos and Bates](https://arxiv.org/abs/2107.07511)).
- **Selective evaluation ("Trust or Escalate")** uses this idea for LLM judges. The judge decides only when its confidence passes a threshold. The rest goes to a stronger judge or to a human. The authors report over 80% human agreement at almost 80% coverage ([Jung, Brahman, and Choi](https://arxiv.org/abs/2407.18370)).

**Inference:** "Trust or escalate" is the same idea as our router. The router accepts a case automatically or sends it to `HUMAN_QUEUE`. The paper gives a principled way to set the router thresholds.

## 2. Step-by-step fit

Legend: **Use**, **Optional experiment**, **Do not use**.

| Pipeline A step | Does a decision model help? | Main risk | Recommendation |
|---|---|---|---|
| 1–3. Run, export, normalize, completeness check | No. These steps are deterministic and count things. Jev "does not count reliably" ([jaggedness](https://docs.typesafe.ai/model-jaggedness/jev-1.13.md)). | Wrong flags on complete runs. | Do not use |
| 4. L1 rule checks | No. Rules are exact, cheap, and scored against the fault manifest. A model adds error and cost. | Replaces hard evidence with soft evidence. | Do not use |
| 5. L2 classifier: one first-failure label | Weak. A Choice question can give one label from the taxonomy with probabilities. But "first failure" needs ordered, multi-hop reading of a long trace. Jev documents weakness at long noisy state, multi-hop questions, and order. No few-shot, no reasons for the reviewer. | Context near or above 32k tokens. Unproven calibration. First-option bias. | Do not use as the primary L2. Optional experiment as a shadow judge only (section 4). |
| 5b. L2 per-mode yes/no checks on a short excerpt | Maybe. A Noul question on a small window, for example the turns around an L1 finding, avoids the long-context problem. Binary checks per failure mode are also the practice that practitioners recommend ([Hamel Husain and Shreya Shankar](https://hamel.dev/blog/posts/evals-faq/)). | Needs a good excerpt rule. Gains are small at our volume. | Optional experiment, after a frozen label set exists |
| 6. Router thresholds | The model itself does not help. A calibrated probability helps, from any judge. Jev's probability is not proven calibrated, so it needs the same calibration step. | Thresholds set on raw, overconfident values. | Do not use Jev here. **Use** a calibration step for all judges (change C1, C2). |
| 6b. Random audit sample | No. The audit must be random to measure error per lane. | — | Do not use |
| 7. L3 human review queue | No. Opik can fill annotation queues automatically ([Opik changelog](https://www.comet.com/docs/opik/changelog/2026/10/5)). Our router already does this, with reasons we control. | A second, hidden queue-filling rule. | Do not use |
| 8. Label export | No. It is a copy step. | — | Do not use |
| 9. L4 taxonomy owner; optional clustering of notes | No. Only a human changes the taxonomy (design 4.3). Jev cannot write cluster names. A Choice question could only map notes to existing modes, and that hides new modes. | Hides new failure modes. | Do not use |
| Judge validation (TPR, TNR on human labels) | The validation method applies to any judge. Add calibration error to it. | — | **Use** the extended validation (change C1) |
| 10. Converter | No. The converter writes text. Jev "is not trained to generate text" ([jaggedness](https://docs.typesafe.ai/model-jaggedness/jev-1.13.md)). | — | Do not use |
| 11. Replayability check | No. It runs the case and compares DB hashes. | — | Do not use |
| Pending: production mode (detectors cannot see the τ²-Bench reward) | Indirect. Without the reward, judges carry more weight. But this raises the need for accuracy, not for speed. | — | Do not use now. Revisit with production volume. |
| Pending: implicit-signal detectors | Mixed. "Handoff to human", "long conversation", and "repeated tool call" are countable, so rules are better. "User frustration", "rephrased request", and "user correction" are semantic. They read short user turns only. Langfuse has a Jev template for exactly these signals ([Langfuse Jev docs](https://langfuse.com/docs/evaluation/evaluation-methods/jev-as-a-judge)). **Inference:** the user is an LLM simulator, so frustration is a weak signal in τ²-Bench. | The signal may mean little with a simulated user. | Rules for countable signals. **Optional experiment** for semantic signals: the only place where a decision model is a natural fit. |
| Pending: funnel settings (judge sampling rate, queue cap, cost cap) | Only at high volume. A cheap judge lets the system judge more traces within one cost cap. At hundreds of runs we can judge every run. | — | Do not use now. Record the option for production. |

## 3. Proposed design changes

These are proposed edits to `04b-final-architecture-tracetogate-foundations.md`. We did not make them. Changes C1 and C2 are worth making with or without a decision model. Changes C3 and C4 are small and optional.

**C1. Add calibration to judge validation.**
- *Section 4.3, paragraph "An LLM judge is validated only after…"*: add one sentence. "A validated judge must also have a calibrated confidence. We fit a calibration map (isotonic regression, or Platt scaling for small sets) on the dev split, and we measure the expected calibration error (ECE) on the test split."
- *Appendix B.3, "Judge calibration" card*: add "fits the calibration map per judge version" to what it does, and "calibration map" to what it writes.
- *Appendix D.1, row "L2 classifier and judges"*: add "ECE, Brier score" to the metrics.
- *Appendix D.2*: add a rule. "The router uses only calibrated confidence values. A new judge version needs a new calibration map."
- *Why:* the router depends on a confidence value. LLM confidence values are often overconfident ([Xiong et al.](https://arxiv.org/abs/2306.13063)). Today the design does not say how to make them trustworthy.

**C2. Set router thresholds from a target error rate.**
- *Section 4.4, paragraph "The two automatic lanes do not have the same threshold"*: replace "a stricter threshold" with a rule. "For each lane, we choose the lowest threshold whose error on the dev split stays below a target. The target for `AUTO_REJECT` is stricter than the target for `AUTO_CANDIDATE`. The audit sample checks the real error rate."
- *Appendix C.9*: replace "high confidence" with "calibrated confidence ≥ the lane threshold".
- *Section 13, "To decide in the build plan"*: add "target error rate per automatic lane".
- *Why:* this is the selective-evaluation idea from "Trust or Escalate" ([Jung, Brahman, and Choi](https://arxiv.org/abs/2407.18370)). It turns "high confidence" into a number that we can test.

**C3. Keep the judge backend replaceable (small).**
- *Appendix B.2, "L2 LLM classifier" card*: add "The judge runs behind one interface. Each finding records the judge backend, the model ID, and the pinned version."
- *Why:* this lets us test a decision model or a local open-weight judge without other changes. It also limits lock-in. The `findings[]` field in C.3 already has `rule_or_judge_id` and `version`, so this is mostly a note.

**C4. Record the experiment as an open item.**
- *Section 13, "Planned later"*: add "Shadow test of a decision model (for example Jev) and a local open-weight judge against the L2 LLM judge, on the frozen human label set. Adopt only if it passes the rule in section 4 of the decision-model assessment."
- *Pending implicit-signal proposal*: when it enters the design, say that countable signals are L1 rules, and that semantic signals (frustration, rephrase, correction) are judges. They follow the same validation as any judge.

We propose no change to L1, L3, L4, the converter, the validator, or the promotion rule.

## 4. How to test

Run this test only after the judge calibration split exists. It needs human labels that the judges never saw (design Appendix B.3).

**Setup.**
1. Freeze one test split of human labels from `labels.jsonl`. Record its hash.
2. Run three judges on the same runs: (a) the L2 LLM judge, (b) Jev with a Choice question over the taxonomy plus `other`, and (c) Jev with one Noul question per failure mode on a short excerpt. Optionally add (d) a local open-weight judge, such as Atla Selene Mini.
3. Pin every model version. Run the shadow judges in shadow mode only: they never feed a router lane.
4. Fit each calibration map on the dev split. Measure on the test split only.

**Metrics.**

| Metric | Why |
|---|---|
| TPR and TNR per failure mode | The design's validation rule (design D.2) |
| Cohen's κ against humans | Agreement beyond chance |
| ECE (10 bins) and Brier score, before and after calibration | Tests the "calibrated probability" claim |
| Coverage at the target lane error | Share of runs that the router can accept automatically. This is the number that saves human time. |
| Share of runs over 32k tokens, and accuracy by trace length | Tests the long-context risk |
| Agreement after option order is shuffled | Tests the first-option bias |
| Cost per 1,000 traces, and p50 / p95 latency | Cost at volume |

**Decision rule (our proposal).**
- Adopt a decision model for a step only if all three are true. First, TPR and TNR are not worse than the LLM judge by more than 0.03 (lower end of a 95% confidence interval). Second, coverage at the target lane error is higher. Third, it saves real money at our volume, or the step is a production-volume step.
- If the model is cheaper but has lower coverage, do not adopt it. At our volume, human review time costs more than judge calls (**Inference**).

**Sample size (Inference).** To estimate a TPR near 0.90 to about ±0.08 (95% Wilson interval), each failure mode needs about 50 positive labels in the test split. Modes with fewer labels stay unvalidated, for every judge.

## 5. Sources

Decision models and Jev:
- LangSmith, LLM-as-judge online evaluations: https://docs.langchain.com/langsmith/online-evaluations-llm-as-judge
- LangSmith, decision model online evaluators: https://docs.langchain.com/langsmith/online-evaluations-decision-models
- Opik changelog, 5 October 2026: https://www.comet.com/docs/opik/changelog/2026/10/5
- Opik, TypeSafe integration: https://www.comet.com/docs/opik/integrations/typesafe
- TypeSafe home page: https://typesafe.ai/
- TypeSafe primitives: https://docs.typesafe.ai/primitives
- TypeSafe models (context, price): https://docs.typesafe.ai/models
- TypeSafe confidence: https://docs.typesafe.ai/confidence
- TypeSafe Jev 1.13 jaggedness (known limits): https://docs.typesafe.ai/model-jaggedness/jev-1.13.md
- TypeSafe legal (data handling): https://docs.typesafe.ai/legal.md
- TypeSafe docs index: https://docs.typesafe.ai/llms.txt
- InfoQ, Jev release news: https://infoq.com/news/2026/10/typesafe-ai-jev-released
- KDnuggets, analysis of Jev: https://www.kdnuggets.com/what-everyone-is-getting-wrong-about-typesafe-ais-jev
- Dylan Black, "Jev Is Poorly Calibrated": https://maximumeffort.substack.com/p/jev-is-poorly-calibrated
- Langfuse, Jev as a judge: https://langfuse.com/docs/evaluation/evaluation-methods/jev-as-a-judge
- Langfuse blog, using TypeSafe's Jev for evals: https://langfuse.com/blog/2026-09-18-using-typesafes-jev-for-evals

Other small judges:
- Galileo Luna-2: https://docs.galileo.ai/concepts/luna/luna
- Patronus Lynx paper: https://arxiv.org/html/2407.08488v2
- Patronus Lynx blog: https://patronus.ai/blog/lynx-state-of-the-art-open-source-hallucination-detection-model
- Patronus Glider paper: https://arxiv.org/pdf/2412.14140
- Patronus Glider on Hugging Face: https://huggingface.co/PatronusAI/glider
- Glider hosted copy and licence: https://featherless.ai/models/PatronusAI/glider
- Atla Selene Mini: https://www.atla-ai.com/post/selene-1-mini
- Atla Selene Mini hosted copy and licence: https://featherless.ai/models/AtlaAI/Selene-1-Mini-Llama-3.1-8B/readme
- Raindrop rd-signal-2: https://raindrop.ai/blog/signals-2-frontier-classification/

Calibration and selective evaluation:
- Xiong et al., confidence elicitation in LLMs: https://arxiv.org/abs/2306.13063
- scikit-learn, probability calibration: https://scikit-learn.org/stable/modules/calibration.html
- Angelopoulos and Bates, conformal prediction: https://arxiv.org/abs/2107.07511
- Jung, Brahman, and Choi, "Trust or Escalate": https://arxiv.org/abs/2407.18370
- Hamel Husain and Shreya Shankar, evals FAQ: https://hamel.dev/blog/posts/evals-faq/

Design under review:
- `04b-final-architecture-tracetogate-foundations.md` (same folder)
