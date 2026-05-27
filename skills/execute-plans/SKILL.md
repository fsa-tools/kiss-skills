---
name: execute-plans
description: Execute a plan from docs/fsa-tools/plans/ (auto-detects the most recent if no argument). Parallel-dispatches clusters via the Agent tool, respects the DAG, and uses /goal to terminate when the global Definition of Done passes. Use in a fresh session.
---

# fsa-tools:execute-plans

## Announce

After resolving the plan file: `Using fsa-tools:execute-plans on <path>.`

## Steps (in order)

### 1. Resolve the plan file

- With an argument: use the provided path.
- Without an argument: `ls -t docs/fsa-tools/plans/*.md | head -1`.
- If no file is found: emit `No plan found in docs/fsa-tools/plans/. Generate one with /fsa-tools:writing-plans.`

### 2. Parse the plan

Read the file and extract (full protocol in `parser.md`):

- Global Definition of Done (shell command from the "Definition of Done" section)
- Invariant Policy (bullet list)
- Worktree header (`recommended` | `required` | `none` | absent)
- Cluster list, and per-cluster the tasks with: id, model, `+reviewer` flag, intra-cluster dependencies, full prompt, verification command

### 3. Worktree setup

Invoke `fsa-tools:worktree`, passing the plan slug and the worktree header value. Receive back either `(absolute path, branch name)` or a `none` signal.

### 4. Activate /goal

The first command of the execution session:

```
/goal <DoD command from plan>
```

The harness will re-run that command after each task and stop the session when it passes.

### 5. TaskCreate per task

For each task in the plan:

- Name: `Cluster N / Task N.M [model]` — e.g. `Cluster 1 / Task 1.1 [sonnet]`
- Model and prompt: from the plan
- `addBlockedBy`: intra-cluster dependencies (task ids) plus inter-cluster dependencies (all task ids in the blocking cluster)

### 6. Dispatch loop

See `dispatcher.md` for the full protocol.

Summary:

1. `TaskList(status=pending, no owner, no open blockedBy)` returns the available set.
2. Send a single message with N parallel `Agent(model=task.model, prompt=task.composed_prompt)` calls.
3. Wait for all results.
4. For each result: run the verification command, optionally invoke `fsa-tools:review`, then `TaskUpdate(id, completed)`. See `verification-loop.md`.
5. Re-poll on every `TaskUpdate(completed)` — completion may unblock further tasks.

### 7. Spot-check after each wave

After every batch returns:

- `git diff --stat HEAD~N HEAD` (N = batch size) — verify the touched files match what the plan declared.
- Confirm zero file overlap between tasks in the same batch.
- Run a partial DoD check if applicable (test suite, type check).
- If anything looks off, escalate to the operator before launching the next wave.

### 8. Global review

When `/goal` detects the DoD is green, before declaring done:

1. Collect the full diff: `git diff <base-branch>...HEAD` (use `main` or `master` if no base branch is known).
2. Invoke `fsa-tools:review` with: the plan file path, the full diff, and a one-line summary of all tasks completed.
3. If the reviewer raises blockers (correctness bugs, invariant violations, missed requirements): escalate to the operator with the reviewer's findings before proceeding. Do not auto-fix.
4. If the reviewer is clean or raises only non-blocking findings: continue to the final report.

### 9. Final report

After the global review passes:

- Summary: N tasks completed in M waves, models used.
- Reviewer outcome: clean / non-blocking findings listed.
- Git state: branch, commit count, worktree path if any.
- Suggest: `/fsa-tools:finish-branch`.

## Internal references

- Parse protocol: `parser.md`
- Dispatch protocol: `dispatcher.md`
- Verification and retry protocol: `verification-loop.md`
