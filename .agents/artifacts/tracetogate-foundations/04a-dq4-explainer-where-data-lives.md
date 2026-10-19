---
date: "2026-10-19"
type: explainer
topic: "DQ4 explained: where the pipeline data lives, and why the design needs this decision"
related: "04-design-discussion-tracetogate-foundations.md (DQ4)"
---

# DQ4 Explained: Where Does the Data Live?

This document uses ASD-STE100 Simplified Technical English. It explains the problem behind design question 4 (DQ4). It does not make the decision. You make the decision after you read it.

## 1. The problem in one sentence

The pipeline makes data in many steps, and many later steps read that data again, so we must decide which copy is the true copy and how it can change.

## 2. Start with one run

The agent handles one customer conversation in τ²-Bench. We call this a **run**. One run makes these items:

| Item | Who makes it | What it contains |
|---|---|---|
| Spans | Our agent and our instrumented environment | Each model call and each tool call, with inputs, outputs, and timing |
| Results JSON | τ²-Bench | The messages and the τ²-Bench reward (1 = pass, 0 = fail) |
| Fault manifest | Our fault injector | The faults that we injected, for example "timeout after commit on `return_items`" |
| State evidence | Our instrumented environment | The DB hash before and after each write, and the changed records |

The exporter also sends a copy of the spans to Langfuse, so that a person can look at the trace.

So, after one run, the same facts are in two or three places: the local files, the τ²-Bench output, and Langfuse.

## 3. Many later steps use this data

The data from one run is not used once. Many steps read it again, sometimes weeks later:

1. The **normalizer** joins the spans, the Results JSON, and the fault manifest into one clean record for each run.
2. The **detectors** and the **router** read the clean records and find suspect runs.
3. A **human reviewer** reads the trace in Langfuse and writes a label.
4. The **judge check** compares LLM judge labels with human labels (TPR and TNR).
5. The **converter** turns a confirmed failure into an eval case.
6. The **dataset** collects eval cases into a numbered version, for example v3.
7. **Pipeline B** runs v1 and v2 of the agent on dataset v3 and gives PASS, FAIL, or REVIEW.
8. Later, a person asks "Why did the gate say FAIL?" and follows the links back to the source traces.

Each step trusts the data that the step before it wrote. If the data is wrong, missing, or changed, every later step gives a wrong result.

## 4. What goes wrong if we do not decide

These problems happen in real projects when nobody decides where the true data lives.

| Problem | Example in this project |
|---|---|
| **Two copies disagree** | Langfuse shows 14 spans for run `r-0412`. The local file shows 15. Which one do the detectors use? |
| **Data disappears** | A hosted platform deletes data after 14 or 30 days. A drift experiment in week 6 needs runs from week 1. |
| **We cannot fix a bug** | In week 3, we find a bug in the normalizer. We must run it again on the original data. If we kept only the processed data, the original is gone. |
| **The dataset changes silently** | Someone removes 3 hard cases from the dataset. The pass rate goes up. The gate says PASS, but the agent did not improve. |
| **Results cannot be compared** | The week 1 gate used 27 cases. The week 5 gate used 31 cases. Without a version number, nobody knows that the two results use different tests. |
| **We cannot explain a decision** | The gate said FAIL. Nobody can find which traces caused the FAIL. |
| **One vendor controls the pipeline** | All logic reads from the Langfuse API. A change to Braintrust means rewriting the pipeline. |

DQ4 exists to prevent these problems. It is a design step because the fix is a rule about data, not a piece of code.

## 5. The key term: system of record

A **system of record** is the one place that holds the true copy of some data. All other places hold copies. When two copies disagree, the system of record wins.

Example from a bank: your phone app shows your balance, but the bank's ledger is the system of record. If the app shows a different number, the ledger is correct and the app is wrong.

In this project, Langfuse is like the phone app. It is good for looking at traces. DQ4 asks what our "ledger" is.

## 6. Three kinds of data, three rules

The data falls into three kinds. Each kind needs a different rule.

| Kind | Examples | Can it change? | Who reads it |
|---|---|---|---|
| **Raw evidence** | Spans, Results JSON, fault manifests | Never. It is a record of what happened. | Normalizer, pipeline-fault experiments, audits |
| **Derived records** | The clean table with one row per run | Yes. We can delete it and build it again from the raw evidence. | Detectors, router, drift experiments |
| **Curated data** | Eval cases, dataset versions, human labels, the failure taxonomy | Only by a new version. An old version never changes. | Pipeline B, judge checks, reviewers |

The important point: raw evidence and curated data must never change in place. Derived records can change at any time, because we can always build them again.

## 7. The questions inside DQ4

DQ4 is three small questions:

1. **Where is the true copy of the raw evidence?** Langfuse, local files, or a database?
2. **Where do the derived records live, and what tool queries them?**
3. **Where do dataset versions and human labels live, and how do we version them?**

## 8. The three options

### Option A: Langfuse is the system of record

- Raw evidence: in Langfuse only. The pipeline reads traces through the Langfuse API.
- Derived records: made from the API data each time.
- Datasets and labels: Langfuse datasets and Langfuse annotation scores.

**Good:** Little work at the start. Langfuse already has datasets and item versions.

**Bad:** The pipeline logic depends on the Langfuse API. Langfuse does not version changes to the item schema. It has no public endpoint to create experiment runs (R§5). If the collector or Langfuse fails, the true data is lost. This is exactly the failure that the pipeline-fault experiments (DQ5 option C) want to test.

### Option B: Local files are the system of record, and Langfuse is a mirror

- Raw evidence: files in `evidence/raw/`. The pipeline writes them once and never changes them.
- Derived records: a Parquet file `evidence/canonical/runs.parquet`. DuckDB queries it with SQL, with no server.
- Datasets: one `manifest.jsonl` file for each version, in git. Each line is one eval case.
- Langfuse: gets a copy of the spans and a copy of each dataset version, for viewing and review only.

**Good:** The pipeline works without Langfuse. A normalizer bug is easy to fix: run it again on the raw files. `git diff v3 v4` shows exactly which eval cases changed. A change to another platform affects only the exporter and the sync step.

**Bad:** One more step (the sync to Langfuse). We must keep the local files safe, because they are the true copy.

### Option C: One Postgres database for everything

- All data goes into tables in one database.

**Good:** One place for everything. Standard SQL.

**Bad:** One more service to install and operate. At a few thousand runs, it gives no benefit over files. Dataset versions in a database are harder to review than a git diff.

## 9. Three scenarios that show the difference

| Scenario | Option A | Option B | Option C |
|---|---|---|---|
| Week 3: a normalizer bug | Read all traces again through the API, if they still exist | Run the normalizer again on `evidence/raw/` | Possible, if the raw data was kept in a separate table |
| The collector stops for 10 minutes | Those spans are lost from the true copy | The local file still has them. The difference measures the loss. | Depends on how spans reach the database |
| "Why did the gate say FAIL in week 5?" | Find the Langfuse dataset version and hope that it did not change | The gate report names `retail-regression v3` and its hash. Git shows the cases. Each case links to its source trace. | Possible, with extra version tables |

## 10. One gap: human labels

A reviewer writes labels in the Langfuse annotation queue (DQ2, DQ6). So, in all three options, the labels are first in Langfuse. But the labels are curated data: they are the ground truth for the judge checks.

With option B, a label that is only in Langfuse breaks the rule "Langfuse is only a mirror". The recommended fix: an export step copies each label to `datasets/labels/labels.jsonl` in git. The step only adds lines and never changes them. Each line has the reviewer, the time, and the taxonomy version. The label files and the taxonomy versions go into git with the dataset manifests.

## 11. The decision you make

Answer these two questions:

1. **Which option is the system of record: A, B, or C?** The recommendation is B.
2. **Do human labels get exported from Langfuse to local files?** The recommendation is yes.

If you choose B, the folder layout is:

```text
evidence/                          # not in git (large); keep a backup
├── raw/<run_id>/spans.otlp.json       # never changed
├── raw/<run_id>/fault_manifest.json   # never changed
├── raw/<batch_id>/results.json        # never changed
└── canonical/runs.parquet             # can be rebuilt at any time
datasets/                          # in git
├── retail-regression/v3/manifest.jsonl   # never changed after publication
└── labels/labels.jsonl                   # only new lines are added
taxonomy/                          # in git
└── v2.yaml                               # with a map from v1 codes to v2 codes
```

Size estimate (inference): one run with content capture is about 50–200 KB of spans. So 10,000 runs are about 0.5–2 GB. This is why `evidence/` stays out of git.
