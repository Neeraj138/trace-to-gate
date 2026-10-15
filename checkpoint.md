# TraceToGate — Checkpoints

Running log of important decisions and context, newest last. Anyone (human or agent) picking up this repo should read this first, then `WORKFLOW_STATUS.md`.

## Orientation

- **What this is:** A learning project. Pipeline A turns agent execution traces into a versioned eval dataset. Pipeline B replays those evals to compare a baseline agent with a candidate and recommend PASS / FAIL / REVIEW for release.
- **Brief:** `trace-to-gate-brief-and-discussion.md`. It holds the motivation, learning goals, and a *draft* Pipeline A architecture. That architecture is not final.
- **Process:** HumanLayer RPI skill (`.agents/skills/rpi/`). Steps: questions → research → design → outline → worktree → implement → PR. The user reviews after each step before the next one starts.
- **Artifacts:** `.agents/artifacts/<task>/`, numbered `NN-*.md`. The RPI authors recommend not committing artifacts, but this repo does commit them so future agents have the full research and design context. Key decisions are also summarized here.
- **Git conventions (user):** Commit and push to GitHub (`origin/main`) regularly, after each workflow step. Do **not** add Claude as co-author or add AI attribution lines.

## Log

### 2026-10-15 — Project kickoff
- Reformatted the brief into clean Markdown (mermaid diagram rebuilt, tables restored). The content is unchanged.
- **Decision:** τ²-Bench Retail is the primary environment (from the brief).
- **Decision (user):** No architecture or coding until research is done. Follow RPI step by step and stop after each step.
- **Decision:** The RPI flow is *design discussion*, matching the user's pipeline (questions → research → design → outline → worktree → implement → PR).
- Initial observations to verify in research:
  - τ²-Bench tasks ship with their initial state and evaluation criteria, so failures are replayable almost by default. Real production traces are not. Consider treating traces as fixture-less and using τ²-Bench ground truth to score the trace-to-eval converter.
  - The agent and the user simulator are both LLMs. Release comparisons therefore need multiple trials and statistics, not single runs.

### 2026-10-15 — Step 1: Research questions
- Task directory: `.agents/artifacts/tracetogate-foundations/`.
- Saved `01-request-…` (original request) and `02-research-questions-…` (11 questions).
- The questions cover τ²-Bench internals, OTel GenAI conventions, Langfuse/Braintrust, industry trace-to-eval practice, replayability, failure detection, drift, release gating, and evaluator reliability.
- User approved the questions and asked to proceed.

### 2026-10-15 — Step 2: Research
- Ran in the same session with 5 parallel research subagents, each with a fresh context and only the neutral questions. τ²-Bench source was read at commit `5ade54e` and key claims were spot-checked by hand.
- Output: `03-research-tracetogate-foundations.md`. Facts most likely to shape design:
  - **τ²-Bench saves full trajectories but no DB snapshots.** The evaluator rebuilds final state by replaying messages on a fresh DB built from `db.json`. All 114 Retail tasks start from the same DB.
  - **Success is effectively final-DB-hash match.** 72 of 114 Retail tasks have no NL assertions, and those score 1.0 automatically. Policy violations that still reach the right end state pass.
  - **τ²-Bench has no OTel, no callback API, and no fault injection.** The only hook is a LiteLLM Langfuse callback behind a code constant. Tool calls execute outside LiteLLM.
  - **Variance is large.** Current models score about 74–84% pass^1 but about 52–61% pass^4 on Retail. The user simulator itself makes critical errors in about 12% of Retail conversations (τ²-bench paper).
  - **Published trajectory data exists:** the repo's `data/tau2/results/final/` (about 590 MB) and a public S3 leaderboard bucket.
  - **OTel GenAI conventions are all "Development" status** and moved to the `semantic-conventions-genai` repo. Nothing normative covers parallel tool calls.
  - **Langfuse** is MIT and self-hostable, with datasets linked to source traces and every item change versioned. **Braintrust** self-hosting is Enterprise-only; its free tier is 1 GB/month with 14-day retention.
  - **Industry trace→eval is a human-in-the-loop workflow** (Hamel/Shankar, Anthropic): error analysis, taxonomy, per-failure evaluator, judge validated with TPR/TNR. No first-party automated pipeline writeup was found.
  - **Failure attribution is weak.** LLM judges reach about 0.80 F1 on whole-trace classification (MAST), but step localization is only 5–18% (TRAIL) and 14% (Who&When).
  - **No drift tooling targets agent-behavior signals** or LLM noise floors. **No numeric release-gate standard exists.** The statistical guidance is paired, clustered SEs plus power analysis (Anthropic "error bars").
- 10 open questions are recorded in the research doc (seed honoring, S3 access, task_issues, LiteLLM OTel output for tools, and others).
- Status: awaiting user review before design.
