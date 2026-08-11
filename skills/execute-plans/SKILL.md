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
- Sprint slice (`- **Sprints:** N` header and `## Sprint N` sections). Absent → a single implicit sprint containing every cluster.
- Cluster list, and per-cluster the tasks with: id, model, `+reviewer` flag, intra-cluster dependencies, full prompt, verification command

### 3. Resume check (cross-session)

A plan can outlive the session that executes it. This step finds out what is already done by looking at the repository, never by trusting a record. Full contract in `continuity.md`.

1. Look for `docs/fsa-tools/continuity/<plan-slug>.md`. Absent → this is a first run; continue to step 4 unchanged. The absence is not an error.
2. Present → read it for decisions, gotchas, and the sprint-progress hint. Then **re-observe rather than trust**: read `HEAD` and `git status --short`, and run the `verification:` command of every task in the plan. Exit 0 means the task is done. That exit code is the only completion signal — there is no stored completion list, by design.
3. The `Sprint progress` section orders the scan so the likely-pending sprint is checked first. It never shortcuts it.
4. Report divergences explicitly and escalate to the operator rather than deciding silently: a task the hint calls complete whose verification now fails, a dirty working tree, an unexpected branch.
5. Dispatch only the tasks whose verification did not exit 0, in DAG order, stopping at the first sprint boundary that follows them.

Re-running every verification is not free on a large plan. It is the price of having one source of truth instead of two, and verification commands are cheap by construction. If it proves slow, the fix is faster verification commands — not a cached completion list.

### 4. Worktree setup

Invoke `fsa-tools:worktree`, passing the plan slug and the worktree header value. Receive back either `(absolute path, branch name)` or a `none` signal.

### 5. Hold the Definition of Done

Keep the DoD command (from step 2) for the terminal gate in step 11. It is a shell command that returns exit 0 when the plan is complete.

Do **not** try to invoke `/goal` — it is a native UI command, run by the operator, and cannot be called via the Skill tool from inside execution. The agent owns termination itself, by running the DoD command via Bash and gating on its exit code (step 11).

### 6. TaskCreate per task

For each task in the plan:

- Name: `Cluster N / Task N.M [model]` — e.g. `Cluster 1 / Task 1.1 [sonnet]`
- Model and prompt: from the plan
- `addBlockedBy`: intra-cluster dependencies (task ids), inter-cluster dependencies (all task ids in the blocking cluster), and sprint dependencies — every task of the preceding sprint. The sprint edges are what make a boundary binding: without them step 7's eager poll ships a sprint N+1 cluster whose `Inter-cluster dependency:` is `none` in the first wave, and the boundary declared in step 9 fires only once sprint N's last task closes — by which point the sprint N+1 work has already landed, too late to bind anything. A plan with one implicit sprint gains no sprint edges, so nothing changes for it.

**Then, before the first poll.** For every task the resume check (step 3) observed passing, `TaskUpdate(id, completed)`. Create first, mark second: the sprint and cluster edges need task ids to reference, so every task in the plan is created even when most are already done. Marking them completed before step 7's first poll is what consumes the resume filter — without it the loop re-dispatches work whose verification already exits 0.

This marking happens outside the dispatch loop. It does not trigger the sprint boundary stop of step 9, which fires only on a completion the dispatch loop produced. A resumed session that marks a whole finished sprint completed here has not reached that sprint's boundary — it passed it in an earlier session.

On a first run the resume check found nothing and this marks nothing.

### 7. Dispatch loop

See `dispatcher.md` for the full protocol.

Summary:

1. `TaskList(status=pending, no owner, no open blockedBy)` returns the available set.
2. Send a single message with N parallel `Agent(model=task.model, prompt=task.composed_prompt)` calls.
3. Wait for all results.
4. For each result: run the verification command, optionally invoke `fsa-tools:review`, then `TaskUpdate(id, completed)`. See `verification-loop.md`.
5. Re-poll on every `TaskUpdate(completed)` — completion may unblock further tasks.

### 8. Spot-check after each wave

After every batch returns:

- `git diff --stat HEAD~N HEAD` (N = batch size) — verify the touched files match what the plan declared.
- Confirm zero file overlap between tasks in the same batch.
- Run a partial DoD check if applicable (test suite, type check).
- If anything looks off, escalate to the operator before launching the next wave.

### 9. Sprint boundary stop

Sprint boundaries are binding, not advisory. A boundary the executor may ignore is decoration.

When the dispatch loop (step 7) completes the last task of a **non-final** sprint:

1. Run the per-wave spot-check (step 8).
2. Write the continuity file (step 10).
3. End the session with `Sprint N/M complete — open a fresh session to continue`.

The trigger is an event in the dispatch loop, not a condition on the task list. Completions produced by step 6's resume marking do not fire it: a resumed session that opens with a whole sprint already marked complete crossed that boundary in an earlier session, and must dispatch the next sprint rather than stop at it again.

The global Definition of Done does not run here. It is global and single, so a partial run would be meaningless. `fsa-tools:finish-branch` is not invoked either — it runs only after the final sprint's Definition of Done passes.

A plan with no sprint slice has exactly one sprint, which is also the final one, so this step never fires.

### 10. Continuity write

At the end of every run — sprint boundary stop, Definition of Done success, or escalation — append to `docs/fsa-tools/continuity/<plan-slug>.md` following `continuity.md`: decisions taken during execution, gotchas discovered, the sprint-progress hint, and one observed-run line (`tasks=`, `waves=`, `diff_lines=`, `duration=`).

Persist only what the repository cannot re-derive. Never record git state, exit codes, file hashes, or a per-task completion list — those are re-observed at resume time (step 3).

On escalation this is what makes the failure resumable instead of lost.

### 11. Definition of Done gate + global review

Once every task of the final sprint is complete — whether the dispatch loop closed it or step 6's resume marking did — run this gate before declaring done.

1. Run the plan's Definition of Done command via Bash. This exit-code gate is the termination signal — the agent owns it (there is no native `/goal` auto-stop). If it returns exit ≠ 0, escalate to the operator with the output; do not declare done.
2. Collect the full diff: `git diff <base-branch>...HEAD` (use `main` or `master` if no base branch is known).
3. Invoke `fsa-tools:review` with: the plan file path, the full diff, and a one-line summary of all tasks completed.
4. If the reviewer raises blockers (correctness bugs, invariant violations, missed requirements): escalate to the operator with the reviewer's findings before proceeding. Do not auto-fix.
5. If the reviewer is clean or raises only non-blocking findings: continue to the final report.

### 12. Final report

After the global review passes:

- Summary: N tasks completed in M waves, models used.
- Reviewer outcome: clean / non-blocking findings listed.
- Git state: branch, commit count, worktree path if any.
- Suggest: `/fsa-tools:finish-branch`.

## Internal references

- Parse protocol: `parser.md`
- Continuity protocol: `continuity.md`
- Dispatch protocol: `dispatcher.md`
- Verification and retry protocol: `verification-loop.md`
