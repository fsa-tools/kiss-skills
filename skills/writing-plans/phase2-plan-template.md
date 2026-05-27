# Phase 2 — Final Plan Template

The shape of the file saved at `docs/fsa-tools/plans/YYYY-MM-DD-<topic>.md`. This is the canonical reference used by `fsa-tools:writing-plans` when composing the final plan and by `fsa-tools:execute-plans` when parsing it.

## Full template

````markdown
# Plan — <Short Topic>

## Metadata

- **Generated:** YYYY-MM-DD
- **Worktree:** recommended | required | none

## Context

<2–6 lines: project root, subproject if any, languages and frameworks.>

## Baseline (current state)

```bash
<command(s) showing the current state — failing tests, lint errors, type errors, etc.>
```

## Objective

<1–3 lines in plain English.>

## Definition of Done (global)

Single verifiable command:

```bash
<shell command that returns exit 0 when the plan is done>
```

**Expected output:** `<line or substring that indicates success>`

## Policy (invariant)

- <Rule that applies to every task.>
- <Rule.>

## Dependency justification

- **Cluster N blockedBy Cluster M:** <why — what artifact does M produce that N consumes?>
- **Task X.Y blockedBy Task X.Z:** <why?>

## Clusters

### Cluster 1 — <Theme>

**Inter-cluster dependency:** none | depends on Cluster N

#### Task 1.1: <Action> [<model>]

**Files:**
- Create: `<path>`
- Modify: `<path>`

**Diagnosis:** <1–3 lines on the root cause or chosen approach.>

**Verification:** `<shell command that returns 0 when the task is done>`

**Prompt for subagent (Agent tool):**
```
<self-contained prompt: project path, files, constraints, verification, "return when X">
```

#### Task 1.2: <Action> [<model>] +reviewer

**Intra-cluster dependency:** 1.1

<...same shape as Task 1.1...>

### Cluster 2 — <Theme>

**Inter-cluster dependency:** depends on Cluster 1

<...tasks...>

## Launch order (DAG resolved)

### Phase 0 — parallel

- Cluster 1 / Task 1.1
- Cluster 1 / Task 1.2
- ...

**Fan-out Phase 0: N parallel tasks**

### Phase 1 — after Phase 0 completes

- Cluster 2 / Task 2.1
- ...
````

## Schema rules

1. **One global Definition of Done per plan.** No per-cluster or per-task DoDs.
2. **Worktree** is a plan-level header. Values: `recommended` (default — operator is asked at execution), `required` (forced), `none` (use current workspace).
3. **Dependencies are declared explicitly.** Inter-cluster in the cluster header. Intra-cluster in the task body. Absent both → independent.
4. **Justification is mandatory** for every declared dependency. Name the artifact produced by the blocker and consumed by the blocked task.
5. **Every task has a verification command** — a shell command that returns exit 0 when the task is done.
6. **Every task has a self-contained subagent prompt** — paths, context, constraints, expected output, the verification command, and a "return when" clause.
7. **No per-task worktree flag.** Worktree is a plan-level decision.
8. **No `effort` field on tasks.** Model selection (haiku/sonnet/opus) is the only tunable.
9. **The "Launch order" section is derived from the DAG**, not hand-written. It groups tasks into phases by dependency depth.

## Worktree header values

- `recommended` — at execution time the worktree helper asks the operator "Create worktree? [Y/n]" (default Y). Use this when the work is non-trivial but a worktree is not strictly required.
- `required` — worktree is created without asking. Use when the work touches many files, or when the operator is also active in the primary working tree.
- `none` — skip the worktree, use the current workspace. Use for single-file or trivially scoped work.

Absent header → treated as `recommended`.
