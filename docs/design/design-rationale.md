# Design Rationale

This document explains the why behind the design choices in this skill suite.

## Why parallelism is the default

Most multi-step tasks (refactors, test cleanups, multi-file features) decompose naturally into independent sub-tasks. Sequential execution wastes wall-clock time and reads context unnecessarily into the orchestrator. Treating parallel as the default and sequential as the exception keeps plans short, fast, and easy to reason about.

The DAG audit (in `skills/writing-plans/dag-audit.md`) makes the cost of sequential dependencies explicit. Each declared `blockedBy` must be justified in the plan; absent justification, tasks are independent.

## Why two-phase planning

A single-pass plan-and-execute approach forces the planner to either:

- spend deep tokens reading every file upfront (slow, expensive), or
- guess at unknowns and produce a fragile plan (low quality).

Two-phase planning splits the problem:

1. **Phase 1 — shortlist.** High-level decomposition with a DAG sketch. The operator checkpoint here is cheap and catches scope misalignment before deep work begins.
2. **Phase 2 — full plan.** File-level investigation, subagent prompts, verification commands. Only happens after the shortlist is approved.

The checkpoint is the key element. It puts expensive work behind a confirmation gate.

## Why /goal as the termination signal

A plan declares a single global Definition of Done: a shell command that returns exit 0 when the work is finished. The execute skill activates `/goal <command>` at the start of execution; the harness re-runs that command after each task and stops when it passes.

This removes the need for the execute skill to manage its own "are we done?" loop. Termination is declarative and verifiable from outside the skill.

## Why model-per-task

A flat refactor across many files is well-served by Haiku. A subtle root-cause debug needs Opus. Forcing every task through one model wastes tokens (Opus for boilerplate) or risks quality (Haiku for hard problems).

The plan declares the model per task. The dispatcher passes `model=<x>` to the Agent tool per dispatch. Model choice becomes part of the plan, reviewable and tunable.

## Why eager dispatch

When N tasks become available simultaneously, dispatching them as N parallel `Agent` calls in a single message is strictly better than serializing: same total work, parallel wall-clock, no inter-task dependencies to track.

The dispatcher re-polls after each task completes — completion may unblock further tasks, and those should ship immediately too.

## Why a worktree helper

Multi-file refactors touch many files. Running them on the operator's primary working tree blocks parallel work. A dedicated worktree per plan keeps execution isolated; the finish-branch skill handles integration when done.

The worktree is optional (header `worktree: recommended` or `none`). For trivial single-file tasks, the overhead of a worktree exceeds the benefit.

## Why review is opt-in

Most well-scoped tasks with a verifiable Definition of Done do not need a second pair of eyes. Mandatory review doubles cost and slows iteration. Instead, the plan author marks specific tasks with `+reviewer` when the stakes warrant it (security, public API surface, irreversible operations).

Spec compliance runs before code quality. A patch that does not do what was asked is not worth a style review.

## KISS decisions

- Plans are markdown files, not YAML or JSON. Operator readability beats schema enforcement.
- A single global Definition of Done per plan, not per-cluster. Composite DoDs become unwieldy.
- No automatic retries with exponential backoff. After 2 failed retries, escalate. Failure modes should surface, not hide.
- No analytics, no telemetry, no version-check hooks. The suite does one thing.
