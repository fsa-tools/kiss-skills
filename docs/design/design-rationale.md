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

## Why the Definition of Done command as the termination signal

A plan declares a single global Definition of Done: a shell command that returns exit 0 when the work is finished. Once the dispatch loop reports all tasks complete, the execute skill runs that command via Bash and gates on its exit code — exit 0 means done, exit ≠ 0 escalates to the operator.

This keeps termination declarative and verifiable, without a fuzzy per-loop "are we done?" heuristic: the stop criterion is a concrete exit code the agent owns end-to-end.

`/goal` (a native UI command that re-runs a condition and auto-stops the session) can express the same criterion, but only the operator can run it — the agent cannot invoke `/goal` from inside execution, so the skill does not depend on it.

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

## Why re-observation instead of recorded state

A plan that spans sprints ends its session at every sprint boundary. The next session starts cold: no memory of the previous run, just the repository and a continuity file. What that resumed session needs to know splits cleanly into two kinds of fact — what the repository already proves, and what only a human decision or an observed run could have told it.

Task completion belongs entirely to the first kind. A task is done exactly when its `verification:` command exits 0 — that is the whole purpose of the field, not an incidental property of it. Re-running that command at resume time costs nothing conceptually: it is the same check the task was defined against, just executed a session later. The same holds for branch, HEAD, dirty paths, and per-task and DoD exit codes — each one is a fact the working tree and git already hold, obtainable by asking rather than by remembering.

Recording a completion list alongside that would create a second source of truth. The moment a task's verification result and a stored "done" flag can diverge — a revert, a manual fix, a flaky check that passed once and regressed — the plan has two answers to "is this task finished" and no principled way to prefer one. That divergence is the exact failure mode the sprint boundary exists to prevent: a resumed session that trusts a stale record instead of the state actually in front of it.

The continuity file is left holding only what re-observation cannot recover: decisions made along the way, gotchas discovered, and metrics observed during the run. None of that lives anywhere else in the repository, so it is the one category worth the cost of writing down.

That cost is real. Re-running every verification command on resume is not free — some are slow, some touch external systems — and it is the price paid deliberately for one source of truth instead of two.

## Why blockedBy edges instead of a dispatch-time filter

`execute-plans` declares sprint boundaries "binding, not advisory," but for a while the mechanism did not actually back that claim. Step 6 (`TaskCreate`) built `addBlockedBy` from intra-cluster and inter-cluster dependencies only, so a sprint N+1 task with no declared dependency on anything in sprint N was, as far as the dispatcher could tell, simply available. Step 7's eager poll does not know about sprints; it dispatches whatever is available. That gap is not a plan-authoring mistake to be closed by declaring a better dependency — sprint slicing is orthogonal to the cluster DAG by design, which is precisely why it needs a mechanism of its own. Any sprint N+1 cluster with no cluster-DAG dependency reaching back into sprint N, `Inter-cluster dependency: none` included, would therefore have shipped in the very first wave alongside sprint N — exactly the outcome the boundary was meant to rule out.

Two ways to close that gap were on the table. One was to give every task in sprint N+1 an `addBlockedBy` on every task in sprint N at `TaskCreate` time, so the boundary becomes an ordinary dependency edge — indistinguishable from the ones the plan already declares. The other was a dispatch-time filter: teach step 7 to narrow the available set to the current sprint before dispatching, leaving `TaskCreate` untouched.

The edges won. They reuse the dependency machinery the dispatcher already trusts instead of adding a second, parallel notion of ordering that the loop would have to know about. A dispatch-time filter would have made `dispatcher.md` sprint-aware; the edges keep it byte-identical, and its eager-dispatch rule — if four tasks are available, all four ship together — stays unconditionally true, because the availability set itself now reflects the sprint boundary rather than the dispatch logic special-casing it. It also protects a specific sentence in `parser.md`: rule 7 says an absent sprint slice "yields a single implicit sprint... the dispatch loop then needs no special case." A dispatch-time filter would have made that false the day sprints were introduced. Edges keep it true, since a one-sprint plan gains no additional `addBlockedBy` entries and behaves exactly as it did before sprints existed.

The edges do real work on the ordinary path, not only the exotic one. A sprint N+1 cluster carrying no cluster-DAG dependency is available at the very first poll — before sprint N has completed a single task, and long before the boundary stop of step 9 has any state to fire on. Without the edges that cluster ships in wave 1; with them it is held back, and nothing else in the mechanism would have caught it.

The honest cost is narrower than it first looks. A sprint edge is a partial duplicate only where a sprint N+1 cluster's declared inter-cluster dependencies already reach every cluster of sprint N; short of that it widens the block from the named clusters to the whole preceding sprint, which is a real constraint and not a restatement of one. The rest of the value sits in the paths that are not the common one: a resumed session where several sprints hold pending tasks, or a future change that lets execution continue past a boundary instead of ending the session there. A rule enforced only by the code path that also happens to stop execution is a rule that quietly stops being enforced the first time that path changes — the edges are what still holds even then.

## KISS decisions

- Plans are markdown files, not YAML or JSON. Operator readability beats schema enforcement.
- A single global Definition of Done per plan, not per-cluster. Composite DoDs become unwieldy.
- No automatic retries with exponential backoff. After 2 failed retries, escalate. Failure modes should surface, not hide.
- No analytics, no telemetry, no version-check hooks. The suite does one thing.
