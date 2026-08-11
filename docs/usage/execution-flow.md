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
- The optional sprint slice: `- **Sprints:** N` header plus `## Sprint N — <Theme>` sections wrapping clusters. Absent means a single implicit sprint containing every cluster.

### 4. Resume check

Before worktree setup, look for `docs/fsa-tools/continuity/<plan-slug>.md`.

- Absent: first run. Proceed to worktree setup.
- Present: a prior session stopped here. Do not trust the file's claims — re-observe. Read HEAD and `git status --short`, then run the `verification:` command of every task in the plan.
- Exit 0 is the only completion signal. There is no stored completion list to read instead.
- The continuity file's `Sprint progress` section only tells you which sprint to scan first — it does not replace the scan.

### 5. Worktree setup

Invoke `fsa-tools:worktree`, passing the plan slug and the worktree header value. The helper either creates a worktree and returns `(path, branch)`, or skips when the header is `none`. An existing worktree on branch `fsa/<plan-slug>` is reused rather than recreated — this is what makes a second session possible.

### 6. Hold the Definition of Done

Keep the DoD command for the terminal gate in step 12. The agent does not invoke `/goal` — that is a native UI command only the operator can run. Termination is agent-owned: run the DoD command via Bash at the end and gate on its exit code.

### 7. TaskCreate per task

For each task in the plan:

- Name: `Cluster N / Task N.M`
- Model: from the plan
- Prompt: from the plan
- `addBlockedBy`: intra-cluster deps (task ids), inter-cluster deps (all tasks in the blocking cluster), and sprint deps (every task of the preceding sprint)

Sprint edges are what make a boundary binding: without them the eager poll in step 8 ships sprint N+1's cluster in the very first wave whenever it declares no inter-cluster dependency reaching back into sprint N, and the boundary stop in step 10 never fires. A plan with one implicit sprint gains no sprint edges — it behaves exactly as before.

Create every task first, then mark the ones the resume check (step 4) already saw pass as `TaskUpdate(id, completed)`, all before the first poll — the edges need ids to reference, so every task is created even when most are already done. This is what consumes the resume check's "dispatch only what failed." On a first run it marks nothing.

That marking happens outside the dispatch loop, so a resumed session that marks a whole finished sprint completed has not triggered that sprint's boundary stop (step 10) — it passed that boundary in an earlier session.

### 8. Dispatch loop

See `skills/execute-plans/dispatcher.md` for the full protocol.

Summary:

1. `TaskList(status=pending, no owner, no open blockedBy)` returns the available set.
2. Single message with N parallel `Agent(model=task.model, prompt=task.prompt)` calls.
3. Wait for all results.
4. Per result: run the verification command, optionally invoke review, `TaskUpdate(id, completed)`.
5. Re-poll: completion may unblock more tasks.

### 9. Spot-check after each wave

After every dispatch batch returns:

- `git diff --stat HEAD~N HEAD` (N = batch size) — verify the touched files match the plan.
- Confirm zero file overlap between tasks in the same batch.
- Run a partial DoD check if applicable (test suite, type check).
- If anything looks off, escalate to the operator before the next wave.

### 10. Sprint boundary stop

When the last task of a non-final sprint completes, stop before touching the next sprint.

- Spot-check the wave as usual (step 9).
- Write the continuity file (step 11).
- End the session: `Sprint N/M complete — open a fresh session to continue.`
- The Definition of Done does not run here. It is global and single — it only runs at the plan's true end, in step 12.

### 11. Continuity write

Append to `docs/fsa-tools/continuity/<plan-slug>.md` at the end of every run — boundary stop, DoD success, or escalation.

- Holds: decisions made, gotchas hit, a `Sprint progress` hint, and one observed-run line.
- Never holds: git state, exit codes, or a completion list — those are re-derived by the resume check (step 4), not stored.

### 12. Final report

Once all tasks in the final sprint are complete, run the DoD command via Bash — this gate applies only to the plan's last sprint; a non-final sprint stops at step 10 instead. Exit ≠ 0 escalates to the operator; exit 0 means done:

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

### Resuming a sliced plan

A plan declares `- **Sprints:** 2`. Sprint 1 has tasks 1.1, 1.2. Sprint 2 has 2.1, 2.2 (depends on sprint 1).

Session 1:

```
t=0    Resume check: no continuity file. First run.
       Worktree setup: new worktree on fsa/<plan-slug>.
       TaskList → available = [1.1, 1.2]
       Dispatch a single message with 2 Agent calls (parallel).
       Both return. Verify both. TaskUpdate(1.1, completed). TaskUpdate(1.2, completed).
t=1    Sprint 1's last wave just closed. Spot-check.
       Write continuity file: decisions, gotchas, Sprint progress = "1/2 done".
       End session: "Sprint 1/2 complete — open a fresh session to continue."
```

Session 2:

```
t=0    Resume check: continuity file present. Re-observe, don't trust it.
       Read HEAD and `git status --short`.
       Run verification for 1.1 and 1.2 → both exit 0.
       Worktree setup: existing worktree on fsa/<plan-slug> is reused.
       TaskCreate all four tasks, addBlockedBy wired — sprint deps included.
       TaskUpdate(1.1, completed). TaskUpdate(1.2, completed). Before the first poll.
t=1    TaskList → available = [2.1, 2.2]  (sprint deps satisfied, sprint 2 unblocked)
       Dispatch a single message with 2 Agent calls (parallel).
       Both return. Verify, mark completed.
t=2    All tasks complete, final sprint reached. Run the DoD command → exit 0. Done.
```

## Spot-check protocol

After every wave:

1. **Scope check.** Each task declared which files it would touch. Compare with `git diff --name-only HEAD~N HEAD`. Out-of-scope edits → escalate.
2. **Overlap check.** Two tasks in the same wave editing the same file is a sign of dispatch error. Should never happen if the DAG audit was thorough.
3. **DoD progress check.** If the DoD is incremental (e.g., a test count), run it. Regression → escalate.
4. **Policy check.** Re-read the plan's Policy section. Any task that violated a policy rule → escalate, do not auto-retry.
