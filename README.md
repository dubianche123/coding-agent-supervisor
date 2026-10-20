<div align="center">

<img src="docs/readme/cover.svg" alt="Loop Warden project overview" width="100%" />

# Loop Warden · Bounded AI Coding Delegation

**The parent defines the boundary; the helper owns the bounded execution loop.**

[![License](https://img.shields.io/badge/License-MIT-3A9F82)](LICENSE)

**English** · [简体中文](README_CN.md)

[Try the entrypoint](#quick-look-no-model-calls) · [Request contract](#a-task-contract-before-an-executor) · [Quality modes](#quality-modes) · [Handoff](#two-tier-handoff)

</div>

If you've ever wired a top-level agent (like Hermes) to spawn local coding CLIs (`gemini`, `codex`), you run into the same unglamorous failures pretty quickly:

1. **Orphan processes**: a child CLI hangs on a pipe or times out, and its subprocess group keeps running in the background.
2. **Context bloat**: the child dumps 7,000+ tokens of raw implementation logs, test output, and review diffs straight back into the parent orchestrator's context window.
3. **Infinite fixup loops**: a fast model fails the same review check three times in a row, burning tokens without making progress while the parent agent has no idea whether to step in.

**Loop Warden** (`delegate_coding_cli.py`) is the supervisor script extracted from my daily Hermes setup to keep that local loop on a short leash: **the parent orchestrator defines the task contract; the helper owns the mechanical implement → test → review → fixup loop**. When it finishes (or hits its retry ceiling), it prints a ~320-token JSON brief to `stdout` and leaves the heavy logs on disk for on-demand inspection.

## Quick look (no model calls)

```bash
git clone https://github.com/dubianche123/coding-agent-supervisor.git
cd coding-agent-supervisor
python3 -B scripts/delegate_coding_cli.py --help
python3 -B scripts/delegate_coding_cli.py --check-health
```

`--help` inspects the CLI contract and `--check-health` reports active child process counts and workspace locks without starting any model.

## A task contract before an executor

Instead of handing a sub-agent a vague one-liner, the parent passes an explicit request contract—target repository, workspace isolation strategy (`direct`, `worktree`, or `copy`), executor (`gemini` or `codex`), review policy (`auto`, `fast`, or `safe`), and a hard fixup budget (`max_fixup_rounds`).

Save this as `/tmp/delegate-request.json` (pointing `repo` at a real Git repository with at least one commit):

```json
{
  "repo": "/absolute/path/to/task-repo",
  "task": "Fix the login loading state after a failed request; add a focused regression check.",
  "executor": "gemini",
  "mode": "implement",
  "workspace_mode": "direct",
  "quality_mode": "safe",
  "stdout_mode": "brief",
  "timeout": 1800,
  "max_fixup_rounds": 2
}
```

```bash
python3 -B scripts/delegate_coding_cli.py --request-json /tmp/delegate-request.json
```

| Parent decides | Helper enforces | Parent receives |
| --- | --- | --- |
| Task & repository | Path safety checks, Git HEAD check, and workspace lock | Selected workspace & artifact paths |
| Executor & `quality_mode` | Test runner, tiered step/final review, subprocess group tracking | Structured findings, test status & attempt count |
| Timeout & fixup budget | Bounded fixup rounds + full process-group cleanup on exit | `passed` or an explicit `needs_escalation` circuit-breaker signal |
| What to do with the diff | Never auto-pushes, merges, or deploys | ~320-token `stdout` brief + section-addressable `handoff.json` on disk |

## Runtime Architecture

![Bounded execution loop and output channels](docs/readme/flow.svg)

[Detailed C4 runtime view](docs/c4-delegation-flow.svg)

This C4-style diagram separates three responsibilities:

- **Parent orchestrator**: defines the task boundary and later decides whether to apply, retry, or escalate.
- **Deterministic helper**: owns workspace locking, model routing, retry limits, heartbeat emission, and structured handoff.
- **Model-backed CLIs**: perform implementation and review calls, but only inside the helper's bounded policy.

The design keeps stdout reserved for the final JSON brief, stderr reserved for progress heartbeats, and detailed logs on disk for selective inspection.

## What It Solves

Local coding-agent workflows often fail in boring but painful places:

| Problem | What Goes Wrong | What This Helper Does |
|:--|:--|:--|
| Process leaks | Child CLIs survive interrupts, timeouts, or stuck pipes. | Tracks process groups and reaps them during timeout, shutdown, and cleanup. |
| Token bloat | The orchestrator reads full logs, review output, and large handoff JSON. | Prints a compact stdout brief and keeps detailed artifacts on disk. |
| No visibility | A long helper run looks frozen until the final JSON appears. | Emits progress heartbeats to stderr without polluting stdout JSON. |
| Weak model loops | Fast models can keep failing the same review and burn retries. | Stops at the fixup boundary and signals `needs_escalation` to the parent. |
| Ad hoc review policy | Every step asks the orchestrator what to do next. | Resolves `quality_mode` once and follows a deterministic review policy. |

## Core Behavior

The helper separates responsibilities:

- **Implementation**: defaults to Gemini CLI with `gemini-3-flash-preview`; Codex CLI can be selected explicitly.
- **Step review**: defaults to `gemini-3.1-flash-lite-preview` for routine checks and fixup decisions.
- **Final review**: safe mode uses `gemini-3.1-pro-preview`; fast mode starts lighter and escalates only when risk warrants it.
- **Parent escalation**: when fixup attempts are exhausted and review still fails, the helper does not auto-upgrade itself. It returns `needs_escalation` so Hermes can resubmit with a stronger model.

## Reading the result without reading every log

1. Parse stdout as JSON. Check `success`, `handoff_status`, `followup_required` and `escalation_required`.
2. If the run passed, inspect the reported changed files and the relevant test/review evidence.
3. If it returned `needs_escalation`, read the findings and attempt history, then let the parent decide whether to retry with a stronger executor or narrow the task.
4. Read full logs only for the stage that needs diagnosis; stderr heartbeats are progress, not the final result.

A `passed` handoff records the helper's execution/review outcome. It is not proof that the change has been merged, deployed, or validated under every environment.

## Runtime Pipeline

1. Validate the target repository and requested workspace mode.
2. Create or select a controlled workspace.
3. Build a constrained delegation prompt.
4. Run implementation through Gemini CLI or Codex CLI.
5. Run detected tests when appropriate.
6. Run step review according to the selected policy.
7. Apply bounded fixup rounds when review finds actionable issues.
8. Stop with `needs_escalation` if max fixup attempts are exhausted and review still fails.
9. Otherwise run final review according to `quality_mode`.
10. Write `orchestrator_brief.json`, `summary.json`, `handoff.json`, and logs.
11. Print the final stdout payload as valid JSON.

## Progress Heartbeat

When `stdout_mode` is `brief`, stdout must stay machine-parseable. Progress messages are therefore written only to stderr:

```text
[Heartbeat] Starting Workspace: Preparing direct workspace for /path/to/repo.
[Heartbeat] Generating Implementation: Round 1: running gemini implementation.
[Heartbeat] Running Tests: Round 1: detecting and running configured tests.
[Heartbeat] Step Review [1]: Running first-pass review with gemini-3.1-flash-lite-preview.
[Heartbeat] Fixup Attempt [1]: Running targeted fixup with gemini.
[Heartbeat] Final Review: Running final review in safe mode.
[Heartbeat] Cleanup: Reaping active child processes after implementation run.
```

This gives humans and logs live observability while preserving stdout for downstream JSON parsing.

## Two-Tier Handoff

The default stdout payload is intentionally small:

```json
{
  "success": true,
  "handoff_status": "passed",
  "followup_required": false,
  "escalation_required": false,
  "next_recommended_action": "Inspect the reported diff and logs, then ask before applying, committing, pushing, or deploying.",
  "paths": {
    "handoff": ".hermes/delegate-runs/<timestamp>/handoff.json",
    "summary": ".hermes/delegate-runs/<timestamp>/summary.json",
    "brief": ".hermes/delegate-runs/<timestamp>/orchestrator_brief.json"
  }
}
```

Full artifacts stay under:

```text
.hermes/delegate-runs/<timestamp>/
```

Read only the section the orchestrator actually needs:

```bash
python3 -B scripts/delegate_coding_cli.py --read-handoff .hermes/delegate-runs/<timestamp>/handoff.json --section findings
python3 -B scripts/delegate_coding_cli.py --read-handoff .hermes/delegate-runs/<timestamp>/handoff.json --section tests
python3 -B scripts/delegate_coding_cli.py --read-handoff .hermes/delegate-runs/<timestamp>/handoff.json --section changed_files
```

Available sections are `brief`, `findings`, `tests`, `changed_files`, `attempts`, `logs`, and `full`.

## Run Artifacts

Every helper run writes its intermediate files under:

```text
<workspace>/.hermes/delegate-runs/<timestamp>/
```

The final stdout brief includes the exact `paths.log_dir`, `paths.summary`, and `paths.handoff` values. The most useful files are:

| File | Meaning |
|:--|:--|
| `delegate_prompt.md` | Initial implementation prompt sent to the executor. |
| `delegate_prompt_round_N.md` | Fixup prompt for implementation round `N`. |
| `implementation.stdout.log` / `implementation.stderr.log` | Latest implementation executor output. |
| `implementation_round_N.stdout.log` / `implementation_round_N.stderr.log` | Implementation executor output for fixup round `N`. |
| `tests.json` | Latest detected test results. |
| `tests_round_N.json` | Test results for fixup round `N`. |
| `gemini_review_prompt.txt` | Latest step review prompt. |
| `gemini_review.stdout.log` / `gemini_review.stderr.log` | Latest step review output, normally from `gemini-3.1-flash-lite-preview`. |
| `gemini_review_round_N.stdout.log` / `gemini_review_round_N.stderr.log` | Step review output for fixup round `N`. |
| `gemini_review_pro.stdout.log` / `gemini_review_pro.stderr.log` | Pro confirmation output when a step review escalates. |
| `gemini_review_pro_round_N.stdout.log` / `gemini_review_pro_round_N.stderr.log` | Pro confirmation output for fixup round `N`. |
| `gemini_final_review_flash.stdout.log` / `gemini_final_review_flash.stderr.log` | Final flash-lite review output in fast mode. |
| `gemini_final_review.stdout.log` / `gemini_final_review.stderr.log` | Final pro review output in safe mode, or when fast final review escalates. |
| `orchestrator_brief.json` | The compact payload intended for the parent orchestrator. |
| `summary.json` | Full run summary, including model routing and artifact paths. |
| `handoff.json` | Structured handoff for follow-up, escalation, or detailed inspection. |

## Token Efficiency Metrics

Every completed run records heuristic token estimates in `token_efficiency`:

```json
{
  "token_efficiency": {
    "orchestrator_stdout": {
      "brief_estimated_tokens": 320,
      "full_summary_estimated_tokens": 7800,
      "estimated_tokens_saved_vs_full_summary": 7480,
      "estimated_reduction_vs_full_summary": 0.959
    },
    "artifact_io": {
      "by_category": {
        "implementation_prompts": { "estimated_tokens": 1800 },
        "step_review_outputs": { "estimated_tokens": 420 },
        "final_pro_outputs": { "estimated_tokens": 260 }
      }
    }
  }
}
```

These numbers are estimates, not provider billing. They use a simple heuristic: CJK characters divided by 1.5 plus other characters divided by 4. They answer two practical questions:

- Did compact stdout save orchestrator context compared with reading the full summary or handoff?
- Which stage produced the most prompt/output volume: implementation, flash-lite review, or pro review?

For exact model cost, compare these run artifacts with provider usage or billing logs. For routing decisions, use these estimates as the first warning light: if a task class saves little or nothing, Hermes should handle that class directly next time.

## Dynamic Escalation Signal

The helper is deliberately bounded. If step review still fails after the configured fixup budget, the helper stops and returns:

```json
{
  "success": false,
  "handoff_status": "needs_escalation",
  "followup_required": true,
  "escalation_required": true,
  "next_recommended_action": "Task exceeded max fixup attempts with current executor. Recommend escalating to a stronger model (e.g., gemini-3.1-pro-preview) and resubmitting the task."
}
```

The exact breaker reason is stored in `summary.json` as `error`, `escalation_reason`, and `escalation_error`. The helper does not perform the model upgrade itself; that decision belongs to the parent orchestrator.

## Quality Modes

| Mode | Behavior |
|:--|:--|
| `auto` | Resolves once to `fast` or `safe` from task risk signals. |
| `fast` | Keeps step review rare; final review starts with flash-lite and calls pro only when risk warrants it. |
| `safe` | Runs step review by default, permits high-risk step review confirmation, and always finishes with pro final review unless the circuit breaker stops first. |

## Delegation Boundary

Delegation is not free. The helper is useful when the saved implementation/review context is larger than the orchestration overhead.

Prefer direct Hermes handling for one-line edits, small README changes, git/admin tasks, questions, and targeted fixes likely to touch 1-2 files under roughly 50 lines.

Prefer delegation for likely 3+ files, roughly 100+ changed/generated lines, nontrivial tests, broad debugging, cross-file refactors, substantial code review, or explicit requests to run the bounded Gemini/Codex helper.

Use `quality_mode: fast` for medium work where pro should be rare. Use `quality_mode: safe` only when the final pro review is worth paying for.

## Installation & Request Reference

To install as a Hermes skill:

```bash
mkdir -p ~/.hermes/skills/devops
cp -R . ~/.hermes/skills/devops/delegate-coding-cli
chmod +x ~/.hermes/skills/devops/delegate-coding-cli/scripts/delegate_coding_cli.py
```

Requirements: **Python 3**, **Git**, and at least one configured executor (**Gemini CLI** and/or **Codex CLI**).

Key request fields for `--request-json`:

| Field | Values | Notes |
|:--|:--|:--|
| `mode` | `plan`, `implement`, `review` | `plan` and `review` run read-only; `implement` permits workspace edits |
| `executor` | `auto`, `gemini`, `codex` | Defaults to `gemini` (`gemini-3-flash-preview`); `codex` uses `gpt-5.4-mini` |
| `quality_mode` | `auto`, `fast`, `safe` | Resolved once per run to control step/final review depth |
| `workspace_mode` | `direct`, `worktree`, `copy` | `direct` requires a Git repo with `HEAD`; `worktree` and `copy` isolate changes |
| `review` | `auto`, `always`, `never` | Strict enum—use `"never"` (not `"none"` or `"off"`) to skip review |
| `stdout_mode` | `brief`, `summary`, `full` | Keep `brief` (default) for orchestrators; `full` is for manual debugging |
| `max_fixup_rounds` | Non-negative integer | Max self-repair rounds before halting with `needs_escalation` |

## Portability

The core helper is portable to Codex CLI or Gemini CLI because it is just a Python entry point that accepts request JSON and writes machine-readable artifacts. A caller only needs to create a request file and run:

```bash
python3 -B scripts/delegate_coding_cli.py --request-json /tmp/delegate-request.json
```

What is Hermes-specific is the skill wrapper and the instruction style around when to invoke it. To use the same design from another CLI, add a small wrapper prompt or command convention that:

- writes the request JSON;
- launches `delegate_coding_cli.py`;
- reads stdout as the compact brief;
- reads `handoff.json` sections only when details are needed;
- avoids recursive loops when the parent CLI and executor CLI are the same tool.

In practice, Codex CLI or Gemini CLI can use this as a local helper today. Turning it into a first-class skill for another agent environment mostly means rewriting the wrapper instructions, not rewriting the Python execution engine.

## Safety Boundaries

The helper is conservative by design:

- It refuses broad or sensitive repository paths.
- It does not push, merge, deploy, or publish.
- It sanitizes sensitive environment variables before child CLI calls.
- It writes prompts to files and invokes subprocesses without shell interpolation.
- It uses workspace locks to avoid concurrent helper runs in the same workspace.
- It tracks child process groups and cleans them up on timeout, interrupt, and shutdown.

## Explore the implementation

The repository is intentionally small. In [delegate_coding_cli.py](scripts/delegate_coding_cli.py), follow these functions rather than reading every log-handling branch first:

| Function | Question it answers |
| --- | --- |
| `normalize_request()` | Which request values and defaults will this run use? |
| `resolve_quality_mode()` | How does `auto` become `fast` or `safe`? |
| `create_workspace()` / `acquire_workspace_lock()` | Where can work happen, and how is concurrent ownership controlled? |
| `run_implementation_round()` / `run_final_review_round()` | Which execution/review calls are made? |
| `terminate_process_group()` / `cleanup_active_processes()` | What survives a timeout or interrupt? |
| `build_orchestrator_brief()` / `build_handoff_section()` | What does the parent read first, and how does it fetch details? |

The [SKILL.md](SKILL.md) wrapper explains how Hermes should choose and invoke the helper. The execution engine itself is a Python entrypoint; other orchestrators can reuse the request/brief contract.

## Repository Layout

```text
.
├── SKILL.md
├── README.md
├── README_CN.md
├── LICENSE
├── docs
│   └── c4-delegation-flow.svg
└── scripts
    └── delegate_coding_cli.py
```

## Project Status

This is an MVP extracted from an active Hermes workflow. The goal is not to replace high-level orchestration; it is to make the local execution layer bounded, observable, and cheap to hand off.
