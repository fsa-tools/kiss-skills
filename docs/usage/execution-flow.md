# Execution Flow

`fsa-tools:execute-plans` reads a plan, dispatches subagents per cluster in DAG-respecting waves, and stops when the Definition of Done command passes (exit-code gated).

## Steps

### 1. Resolve the plan file

- With an argument: use the provided path.
- Without an argument: `ls -t docs/fsa-tools/plans/*.md | head -1`.
- No file found: emit `No plan found in docs/fsa-tools/plans/. Generate one with /fsa-tools:writing-plans.`

### 2. Announce

After resolving: `Using fsa-tools:execute-plans on <path>.`

### 3. Parse

Extract from the plan (full protocol in `skills/execute-plans/parser.md`):

- The global Definition of Done command
- The invariant Policy rules
- The Worktree header value (`recommended` | `required` | `none` | absent)
- The cluster list, and per-cluster the tasks: id, model, `+reviewer` flag, intra-cluster dependencies, prompt, verification command
- The inter-cluster dependencies from each cluster header

### 4. Worktree setup

Invoke `fsa-tools:worktree`, passing the plan slug and the worktree header value. The helper either creates a worktree and returns `(path, branch)`, or skips when the header is `none`.

### 5. Hold the Definition of Done

Keep the DoD command for the terminal gate in step 9. The agent does not invoke `/goal` — that is a native UI command only the operator can run. Termination is agent-owned: run the DoD command via Bash at the end and gate on its exit code.

### 6. TaskCreate per task

For each task in the plan:

- Name: `Cluster N / Task N.M`
- Model: from the plan
- Prompt: from the plan
- `addBlockedBy`: intra-cluster deps (task ids) plus inter-cluster deps (all tasks in the blocking cluster)

### 7. Dispatch loop

See `skills/execute-plans/dispatcher.md` for the full protocol.

Summary:

1. `TaskList(status=pending, no owner, no open blockedBy)` returns the available set.
2. Single message with N parallel `Agent(model=task.model, prompt=task.prompt)` calls.
3. Wait for all results.
4. Per result: run the verification command, optionally invoke review, `TaskUpdate(id, completed)`.
5. Re-poll: completion may unblock more tasks.

### 8. Spot-check after each wave

After every dispatch batch returns:

- `git diff --stat HEAD~N HEAD` (N = batch size) — verify the touched files match the plan.
- Confirm zero file overlap between tasks in the same batch.
- Run a partial DoD check if applicable (test suite, type check).
- If anything looks off, escalate to the operator before the next wave.

### 9. Final report

Once all tasks are complete, run the DoD command via Bash. Exit ≠ 0 escalates to the operator; exit 0 means done:

- Summary: N tasks completed in M waves, models used.
- Git state: branch, commit count, worktree path if any.
- Suggest: `/fsa-tools:finish-branch`.

## End-to-end example

A plan has 4 tasks: 1.1, 1.2 (no deps), 2.1, 2.2 (both depend on cluster 1).

```
t=0    TaskList → available = [1.1, 1.2]
       Dispatch a single message with 2 Agent calls (parallel).
       Both return.
       Verify both. TaskUpdate(1.1, completed). TaskUpdate(1.2, completed).
t=1    Re-poll. available = [2.1, 2.2]  (cluster 2 unblocked)
       Dispatch a single message with 2 Agent calls (parallel).
       Both return.
       Verify, mark completed.
t=2    All tasks complete. Run the DoD command → exit 0. Done.
```

Wall-clock: 2 waves versus 4 if serialized.

## Spot-check protocol

After every wave:

1. **Scope check.** Each task declared which files it would touch. Compare with `git diff --name-only HEAD~N HEAD`. Out-of-scope edits → escalate.
2. **Overlap check.** Two tasks in the same wave editing the same file is a sign of dispatch error. Should never happen if the DAG audit was thorough.
3. **DoD progress check.** If the DoD is incremental (e.g., a test count), run it. Regression → escalate.
4. **Policy check.** Re-read the plan's Policy section. Any task that violated a policy rule → escalate, do not auto-retry.
