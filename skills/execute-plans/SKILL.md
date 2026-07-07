---
name: execute-plans
description: Execute a plan from docs/fsa-tools/plans/ (auto-detects the most recent if no argument). Parallel-dispatches clusters via the Agent tool, respects the DAG, and terminates when the global Definition of Done command passes (exit-code gated). Use in a fresh session.
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

### 4. Hold the Definition of Done

Keep the DoD command (from step 2) for the terminal gate in step 8. It is a shell command that returns exit 0 when the plan is complete.

Do **not** try to invoke `/goal` — it is a native UI command, run by the operator, and cannot be called via the Skill tool from inside execution. The agent owns termination itself, by running the DoD command via Bash and gating on its exit code (step 8).

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

### 8. Definition of Done gate + global review

Once the dispatch loop reports all tasks complete, before declaring done:

1. Run the plan's Definition of Done command via Bash. This exit-code gate is the termination signal — the agent owns it (there is no native `/goal` auto-stop). If it returns exit ≠ 0, escalate to the operator with the output; do not declare done.
2. Collect the full diff: `git diff <base-branch>...HEAD` (use `main` or `master` if no base branch is known).
3. Invoke `fsa-tools:review` with: the plan file path, the full diff, and a one-line summary of all tasks completed.
4. If the reviewer raises blockers (correctness bugs, invariant violations, missed requirements): escalate to the operator with the reviewer's findings before proceeding. Do not auto-fix.
5. If the reviewer is clean or raises only non-blocking findings: continue to the final report.

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
