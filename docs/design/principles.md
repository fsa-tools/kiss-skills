# Design Principles

## P1 — Parallelism-first

Parallelism is the default; sequential is the exception. Concrete rules:

**For plan writing:**

- Tasks are independent by default. A `blockedBy` dependency must be justified in the plan's "Dependency justification" section.
- Decompose aggressively: a broad task with parallelizable parts becomes N tasks.
- File overlap forces a sequential split: if two tasks touch the same file, make them sequential.
- A cluster with 1 task is fine. A cluster with 6+ tasks is suspect and should be split.
- The quality metric of a plan is the fan-out of Phase 0 (the initial parallel batch).

**For execution:**

- Eager dispatch: every available task ships in a single message with N parallel `Agent` calls. No artificial ordering by model, alphabetical, or "I'll do that one first".
- Re-poll after each task completes. Completion may unblock other tasks; ship them immediately.
- Inter-cluster dependencies are honored only when they exist in the DAG. The dispatcher does not invent ordering.

## P2 — Autonomy

A plan is structured well enough that subagents can execute without back-and-forth.

- Every task has a `prompt` field that is self-contained: paths, context, constraints, expected output.
- Every task has a `verification` field: a shell command that returns exit 0 when the task is done.
- A task that requires the operator to "figure out X" is a smell — write the prompt clearly enough that the subagent can decide.

## KISS

Concrete simplifications:

- Plans are plain markdown. No YAML schema, no JSON validation.
- One global Definition of Done per plan. No per-cluster or per-task DoDs to aggregate.
- Termination via the plan's Definition of Done command, gated on its exit code. The execute skill runs it once at the end, not a per-loop done-detection heuristic. (`/goal` is a native UI command the operator may run, but the agent cannot invoke it from inside execution.)
- No retry-with-backoff. After 2 retries, escalate to the operator. Hide nothing.
- Markdown skill files live in `skills/<name>/SKILL.md`. Helper docs live next to the SKILL.md they support.
- No state outside the working directory and git.
