---
name: autoresearch
description: "Run an orchestrator-led autonomous optimization loop (Karpathy/Pi-style) against one primary metric per phase with gated subgoals, hard correctness backpressure, isolated-worktree worker delegation, git-ratchet keep/revert, and seeded + evidence-driven + web-researched exploration. Use when setting up or running iterative optimization loops (accuracy, quality, latency, cost, throughput, model/prompt/code tuning) or optimizing quality first before unlocking speed/cost subgoals."
argument-hint: "[goal, target metric, seeded directions, or resume]"
user-invocable: true
---

# Autoresearch

Run an orchestrator-led **hypothesize → delegate → measure → keep/discard** optimization loop on a dedicated git branch, optimizing **one primary number at a time**.

## Core Rules

1. **One overarching orchestrator.** The main agent stays in the Orchestrator seat: it owns strategy, state files, budget, phase transitions, and the experiment branch, and delegates implementation, failure analysis, and web exploration to disposable workers so its own context stays clean across long runs.
2. **One number per phase.** Never blend competing objectives (e.g. accuracy + speed) into a mushy composite score. Optimize Phase 1's single primary metric until its unlock gate is met, lock Phase 1's best score as a hard floor in `autoresearch.checks.sh`, and only then promote Phase 2's primary metric (e.g. latency or cost).
3. **No hacks, no metric gaming.** Never edit `autoresearch.sh` or the eval harness to inflate scores, hardcode benchmark cases/IDs, suppress failures, or bypass checks. Every kept commit must be a clean, generalizable improvement.
4. **Hard backpressure beats raw score.** `autoresearch.checks.sh` must exit `0` for any candidate to be kept. If checks fail, discard — no exceptions.
5. **Monotonic git ratchet.** Only verified wins land on `autoresearch/<slug>`. Discards, failed checks, and crashes never advance the branch.
6. **Exploit and explore.** Hill-climb from concrete failure evidence, and break out of local maxima with novel swings and web research.
7. **Respect the autonomy boundary.** Move fast inside the mutable scope; gatekeep external tools, new dependencies/models, spend overrides, and eval/spec changes.

## Roles: Orchestrator vs. Workers

Use whatever subagent / isolated-worktree delegation primitive the host harness provides (e.g. worktree-isolated delegates, subagents). If the host has no delegation tools, execute the same loop sequentially in the main agent and `git reset --hard <last_keep_commit>` on non-keeps.

### Main Agent — Orchestrator
- Owns `autoresearch.md`, `autoresearch.ideas.md`, `autoresearch.jsonl`, `proposals.md`, budget tracking, and the `autoresearch/<slug>` branch.
- **Never bloats its own context** with raw eval dumps, full tracebacks, or long web-search transcripts — reads compact structured reports from workers instead.
- Chooses hypotheses, dispatches workers (1 at a time, or 2–3 independent bets in parallel when budget allows), verifies winners, lands the best valid commit onto `autoresearch/<slug>`, logs every outcome, and manages phase promotion.

### Disposable Workers (Fresh Context Per Task)
- **Experiment Worker** *(isolated git worktree from `<last_keep_commit>`)*:
  - Receives one crisp hypothesis brief: hypothesis, mutable file scope, immutable boundaries, exact commands (`autoresearch.sh`, optional fast pre-screen, `autoresearch.checks.sh`), and the current best metric to beat.
  - Implements cleanly on its isolated branch, runs measurement + checks (never edits `STATE_DIR` files or the eval harness), and returns a compact report: commit SHA/branch, parsed `METRIC` values, check status, diff summary, cost/runtime, and concise failure notes.
- **Failure Analyst** *(read-only)*:
  - Inspects the worst-scoring `train`/dev cases from the current best run (never held-out `test`) and returns top root-cause patterns + concrete `Exploit` hypotheses.
- **Explorer / Scout** *(read-only + web search)*:
  - Dispatched on plateaus (after 3 consecutive non-keeps), every 5th iteration, or for open-ended seeded directions.
  - Searches papers, OSS repos, benchmarks, and alternative architectures/prompting/algorithmic techniques, returning ranked `Explore` hypotheses (and gatekept pitches for `proposals.md`).

## Session Layout

Keep all session state in one directory (`STATE_DIR`, default `.autoresearch/` or an external data workspace path when code and data are separated). **Keep `STATE_DIR` outside the code repo or gitignored** so branch operations and resets never touch logs, memory, or proposals.

```text
<STATE_DIR>/
├── autoresearch.md         # Living brief, phases, scope, caps, wins & dead ends (survives resets)
├── autoresearch.sh         # Immutable benchmark script; emits `METRIC <name>=<value>` lines
├── autoresearch.checks.sh  # Correctness/invariants gate + locked floors from completed phases
├── autoresearch.ideas.md   # Backlog in three lanes: Seeded, Exploit, Explore
├── autoresearch.jsonl      # Append-only experiment ledger (never edited or truncated)
├── proposals.md            # Gatekept ideas (tools/deps, big bets) & dataset/spec incongruities
└── iterations/             # Per-iteration evidence for post-run review (never loaded bulk into Orchestrator)
    └── iter-NNN/
        ├── brief.md        # Hypothesis & instructions dispatched to the worker
        ├── diff.patch      # Exact code/prompt diff tested (kept even for discards)
        ├── metrics.log     # Raw stdout/stderr from autoresearch.sh
        ├── checks.log      # Raw stdout/stderr from autoresearch.checks.sh
        └── report.md       # Worker's compact structured summary
```

### Metric Output Format (`autoresearch.sh`)

`autoresearch.sh` exits `0` on a valid run and prints machine-readable lines (one primary, any number of secondary metrics for visibility):

```text
METRIC val_score=0.9280
METRIC train_score=0.9220
METRIC latency_s=119.4
METRIC cost_usd=0.31
```

### Ledger Entry Format (`autoresearch.jsonl`)

Append one JSON object per experiment (written only by the Orchestrator):

```json
{"iter": 1, "ts": "2026-11-24T13:00:00Z", "phase": 1, "lane": "exploit", "commit": "a1b2c3d", "hypothesis": "...", "primary_name": "val_score", "primary": 0.9410, "secondary": {"train_score": 0.9380, "latency_s": 114.2, "cost_usd": 0.30}, "checks": "pass", "status": "keep", "duration_s": 128, "artifact_dir": "iterations/iter-001", "notes": "..."}
```

Valid `status` values: `baseline`, `keep`, `discard`, `checks_failed`, `crash`.

## Setup (New Session)

Before iteration 1, the Orchestrator:

1. **Inspects the workspace.** Reads repo docs (`AGENTS.md`, `README.md`), existing eval scripts, tests, and any data/code split or spend-ledger rules.
2. **Locks the contract** with the user (infers what was already stated; asks concisely only for missing decisions that change the objective):
   - **Phases & unlock gates:** Phase 1 primary metric (`name`, `higher|lower`, `min_delta`, confirmation rule, target threshold or plateau threshold) and any Phase 2+ subgoals (e.g. minimize `latency_s` or `cost_usd` with Phase 1 locked as a floor).
   - **Scope:** mutable files/directories vs. strictly immutable files (eval harness, datasets, held-out splits).
   - **Backpressure (`autoresearch.checks.sh`):** unit tests, schema/invariant checks, zero-hallucination/zero-regression assertions.
   - **Caps & mode:** per-session iteration/time/spend cap, max parallel experiment workers (default `1`; up to `2–3` when budget allows), optional fast pre-screen, and run mode (**interactive** vs. **unattended**).
   - **Seeded directions:** initial ideas or exploratory angles from the user.
3. **Creates the experiment branch:** `git checkout -b autoresearch/<slug>` from a clean working tree. Verifies `STATE_DIR` is outside the repo or gitignored.
4. **Writes the contract files** (`autoresearch.md`, `autoresearch.sh`, `autoresearch.checks.sh`, `autoresearch.ideas.md`, empty `autoresearch.jsonl` and `proposals.md`) and makes the `.sh` scripts executable.
5. **Runs the baseline:** executes `autoresearch.sh` and `autoresearch.checks.sh`, confirms both pass cleanly, logs `status: "baseline"` to `autoresearch.jsonl`, and records baseline numbers + commit SHA in `autoresearch.md`.

## The Orchestrator Loop

To resume a session (or recover after any context reset), start at step 1 by reading `autoresearch.md`, the tail of `autoresearch.jsonl`, and `autoresearch.ideas.md`.

Repeat until the final phase goal is met, a cap is reached, or the user stops the loop:

1. **Orient.** Read `autoresearch.md`, recent `autoresearch.jsonl` entries, and `autoresearch.ideas.md`. Check remaining budget/iterations and current phase.
2. **Replenish ideas when needed.**
   - After a new `keep` (or when `Exploit` is empty) → dispatch a **Failure Analyst** on the latest best run's worst `train`/dev cases to generate concrete `Exploit` hypotheses.
   - After **3 consecutive non-keeps** (plateau), every **5th iteration**, or when seeded directions call for novelty → dispatch an **Explorer / Scout** with web search to generate `Explore` hypotheses.
3. **Select hypothesis (or small parallel batch).** Pick the highest-expected-value queued item(s) across `Seeded`, `Exploit`, and `Explore`.
4. **Enforce the Autonomy Boundary.**
   - **Autonomous (proceed):** code, prompt, config, or architecture changes within scoped files & existing dependencies; web research; clean refactors.
   - **Gatekept (do not execute silently):** installing a new external tool, package, API, or model provider; changing the eval harness, metric, or output schema; exceeding spend/time caps; resolving an apparent benchmark or reference ambiguity.
   - **Interactive mode:** pause and ask the user with a concise pitch.
   - **Unattended mode:** never stall the loop — append a structured proposal to `proposals.md` (what, why, source link, expected gain, cost/risk), mark the item `parked` in `autoresearch.ideas.md`, and pick the next autonomous hypothesis.
5. **Dispatch Experiment Worker(s).** Launch isolated worktree worker(s) branched from `<last_keep_commit>` with a tight spec (one testable hypothesis per worker so outcomes stay attributable). Never run parallel workers that exceed remaining spend budget.
6. **Verify & Ratchet.** When worker(s) report back:
   - Reject any candidate that crashed, failed `autoresearch.checks.sh`, touched immutable files, or did not beat `best` by at least `min_delta`.
   - For stochastic/LLM evals with a confirmation rule configured, run confirmation before accepting a candidate win.
   - Among valid winners, pick the strongest primary metric, review its actual diff (verify zero hacks / zero memorization), fast-forward or cherry-pick it onto `autoresearch/<slug>`, and re-verify `autoresearch.checks.sh` on the branch head.
   - Discard losing worktrees cleanly.
7. **Record & Reflect.**
   - Ensure the iteration's evidence folder (`iterations/iter-NNN/`: `brief.md`, `diff.patch`, `metrics.log`, `checks.log`, `report.md`) is saved cleanly — for both keeps and discards — so post-run review has the full trail.
   - Append one JSON entry per evaluated candidate to `autoresearch.jsonl`.
   - Update `autoresearch.ideas.md` (mark `tried` / `dead-end`, add new follow-ups).
   - Update `autoresearch.md` with the new best commit/metrics and a 1–2 line lesson so dead ends are never re-tried.
8. **Check Phase Promotion.** When the active phase hits its target threshold — or plateaus after the configured `Explore` attempts — and another phase is queued:
   1. Lock the completed phase's best metric into `autoresearch.checks.sh` as a hard floor (with configured tolerance, e.g. `>= phase1_best - tolerance`).
   2. Advance `active_phase` and primary metric in `autoresearch.md` to the next subgoal.
   3. Log the current head as the new phase's `baseline` in `autoresearch.jsonl`.

## Guardrails for Stochastic, ML, and LLM Evals

When optimizing LLM pipelines, prompts, or ML models, include these in `autoresearch.md` and `autoresearch.checks.sh`:

- **Split discipline:** diagnose failures on `train`, gate `keep` on `val` (or require both to improve with zero `val` regression), and never read or evaluate `test` inside the loop.
- **Anti-memorization:** forbid case/entity-specific names or lookup hacks in prompts or code; add a deterministic grep check in `autoresearch.checks.sh` when applicable.
- **Noise margin:** set a non-zero `min_delta` larger than baseline run-to-run jitter and/or require a confirmation replicate on candidate wins.
- **Spend cap & fast-fail:** enforce a hard per-session budget ceiling and prefer a small hard-case pre-screen before paying for full eval passes.
- **Spec/label ambiguity:** if a recurring error looks like an inconsistent reference or ambiguous spec rather than a real system bug, log it in `proposals.md` instead of twisting the system to overfit noisy labels.

## Wrapping Up

When stopping (all phases achieved, cap reached, or user-stopped):

1. Verify `autoresearch/<slug>` is clean at `<last_keep_commit>` and `autoresearch.checks.sh` passes.
2. Report concisely:
   - Baseline → best per phase (primary + secondary metrics)
   - Kept commits with deltas and why each worked
   - Notable dead ends (so nobody repeats them)
   - Iterations run, workers dispatched, and total cost/time
   - Open items in `proposals.md` (new tools, big bets, dataset/spec incongruities) ready for user review
   - Path to `STATE_DIR` (ready for an offline post-run review)
3. For longer sessions, offer a self-contained HTML summary report if helpful.
