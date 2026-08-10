# Plan — Sprint Enforcement at Dispatch

## Metadata

- **Generated:** 2026-08-10
- **Worktree:** none

## Context

Project root: `/Users/fabiosiqueira/dev/projetos/kiss-skills`. This is the `fsa-tools` Claude Code plugin — plain markdown skills, no build step, no runtime, no test suite. The "code" is the skill prose itself, so every verification command here is a `grep`/`test` assertion against a file.

**Precondition: the cross-session round must be committed before this plan executes.** This plan edits steps 6 and 7 of `skills/execute-plans/SKILL.md` as renumbered by `2026-08-10-cross-session-execution.md`. Those edits currently live in the working tree, uncommitted. Executing against a dirty tree would mix two rounds into one diff and leave the global review without a clean baseline. Run the operator's close-out flow first.

Three facts that shape the decomposition:

- **`Worktree: none` is deliberate**, for the same reason as the previous plan: every verification is a `grep` against a path, and under a worktree the orchestrator's Bash cwd and the tree the subagents edited are not reliably the same tree. With no build and no tests, a worktree buys nothing and adds a failure class. It would also branch from `HEAD`, which is the wrong baseline for this work.
- **No self-modification in flight.** The running skill is loaded from the plugin cache; the edits land in this repo. Verification is therefore textual — it asserts that the prose says X, never that the dispatch behaves that way at runtime.
- **The narrative already assumes the fix.** `docs/usage/execution-flow.md:119` says sprint 2 "depends on sprint 1" and `:141` says `available = [2.1, 2.2] (sprint 2, now unblocked)`. The doc describes edges that `SKILL.md` step 6 never creates. This is a gap between stated and implemented behaviour, not a missing design.

## Baseline (current state)

```bash
cd /Users/fabiosiqueira/dev/projetos/kiss-skills
grep -n 'addBlockedBy' skills/execute-plans/SKILL.md          # line 58 — intra + inter cluster only, no sprint dimension
grep -c 'preceding sprint' skills/execute-plans/SKILL.md      # 0 — sprint edges do not exist
grep -c 'before the first poll' skills/execute-plans/SKILL.md # 0 — resume filter has no consumer
grep -n 'now unblocked' docs/usage/execution-flow.md          # line 141 — narrative already assumes the edges
grep -c '^### ' skills/execute-plans/SKILL.md                 # 12 — step count must not change
grep -c '^### ' docs/usage/execution-flow.md                  # 13 — 12 steps + the resume example
```

## Objective

Make sprint boundaries binding in the mechanism, not just in the prose, and give the resume check's "dispatch only what failed" a consumer. Both changes live in `execute-plans` step 6; the dispatch loop and the parser stay untouched.

## Definition of Done (global)

Single verifiable command:

```bash
cd /Users/fabiosiqueira/dev/projetos/kiss-skills && set -e && \
grep -q 'every task of the preceding sprint' skills/execute-plans/SKILL.md && \
grep -q 'before the first poll' skills/execute-plans/SKILL.md && \
grep -q 'make a boundary binding' skills/execute-plans/SKILL.md && \
grep -q 'outside the dispatch loop' skills/execute-plans/SKILL.md && \
test "$(grep -c '^### ' skills/execute-plans/SKILL.md)" -eq 12 && \
grep -q 'needs no special case' skills/execute-plans/parser.md && \
grep -q 'SKILL.md step 11' skills/execute-plans/dispatcher.md && \
grep -q 'sprint edges' docs/usage/execution-flow.md && \
grep -q 'before the first poll' docs/usage/execution-flow.md && \
test "$(grep -c '^### ' docs/usage/execution-flow.md)" -eq 13 && \
grep -q 'Why blockedBy edges' docs/design/design-rationale.md && \
grep -qi 'dispatch-time filter' docs/design/design-rationale.md && \
echo "DOD_PASS"
```

**Expected output:** `DOD_PASS`

The two `grep -c '^### '` assertions and the two greps against `parser.md` / `dispatcher.md` are guards, not goals: they fail if a task adds a step, removes one, or "helpfully" edits a file this plan deliberately leaves alone.

## Policy (invariant)

- **English only.** Every file committed to this repo ships in the plugin. No pt-BR in any edited file.
- **No new steps.** The step count is fixed: 12 `### ` headings in `skills/execute-plans/SKILL.md`, 13 in `docs/usage/execution-flow.md`. This change edits existing steps; it does not add one.
- **Do not edit `skills/execute-plans/dispatcher.md` or `skills/execute-plans/parser.md`.** The whole point of enforcing at `TaskCreate` is that the dispatch loop and the parser need no change. A task that believes otherwise stops and reports.
- **Surgical edits only.** Change exactly the lines the task names. No reflow of untouched paragraphs, no rewording of neighbouring prose, no section reordering, no "while I'm here" improvements.
- **Stay inside the declared Files list.** A task that believes it needs another file stops and reports instead of editing it.
- **Do not commit, do not stage.** The operator's close-out flow owns commits.
- **Never touch `CHANGELOG.md` or `.claude-plugin/plugin.json`.** Versioning and changelog belong to the close-out flow.

## Dependency justification

None. No task declares `blockedBy`.

The dependency that looks real — the docs (2.1, 2.2) needing the mechanism (1.1) to exist first — is false. Each doc prompt carries the full specified behaviour inline, so the doc is written against the specification rather than against the file. This is stub-and-integrate with the specification in the stub's role. The previous plan used the same technique for its narrative task and it held.

File-overlap audit: three tasks, three disjoint files.

## Clusters

### Cluster 1 — Enforcement at dispatch

**Inter-cluster dependency:** none

#### Task 1.1: Wire sprint edges into TaskCreate and consume the resume filter [opus] +reviewer

**Files:**
- Modify: `skills/execute-plans/SKILL.md`

**Diagnosis:** Step 6 builds `addBlockedBy` from intra-cluster and inter-cluster dependencies only. Sprint membership plays no part, so step 7's eager poll (`TaskList(status=pending, no owner, no open blockedBy)`) will dispatch a sprint 2 cluster whose `Inter-cluster dependency:` is `none` in the very first wave — while step 9 claims boundaries are "binding, not advisory". Separately, step 3.5 says to dispatch only the tasks whose verification failed, but step 6 creates every task as pending and step 7 dispatches every unblocked pending task, so a resumed session re-runs completed work. Both are fixed in step 6: sprint edges make the boundary binding, and marking the already-passing tasks completed before the first poll gives the resume filter its consumer.

**Verification:** `cd /Users/fabiosiqueira/dev/projetos/kiss-skills && grep -q 'every task of the preceding sprint' skills/execute-plans/SKILL.md && grep -q 'before the first poll' skills/execute-plans/SKILL.md && grep -q 'make a boundary binding' skills/execute-plans/SKILL.md && grep -q 'outside the dispatch loop' skills/execute-plans/SKILL.md && test "$(grep -c '^### ' skills/execute-plans/SKILL.md)" -eq 12`

**Prompt for subagent (Agent tool):**
````
Project: /Users/fabiosiqueira/dev/projetos/kiss-skills
File to modify: skills/execute-plans/SKILL.md (this is the ONLY file you may touch)

This repo is the `fsa-tools` Claude Code plugin — markdown skills, no build, no tests. All content is written in English. Read the whole file first, plus skills/execute-plans/dispatcher.md and skills/execute-plans/parser.md for context (read only — you may not edit either).

CONTEXT — the bug being fixed

The file has twelve numbered steps. Step 9 ("Sprint boundary stop") opens with "Sprint boundaries are binding, not advisory. A boundary the executor may ignore is decoration." Nothing in the file makes that true.

Step 6 ("TaskCreate per task") currently reads:

### 6. TaskCreate per task

For each task in the plan:

- Name: `Cluster N / Task N.M [model]` — e.g. `Cluster 1 / Task 1.1 [sonnet]`
- Model and prompt: from the plan
- `addBlockedBy`: intra-cluster dependencies (task ids) plus inter-cluster dependencies (all task ids in the blocking cluster)

Sprint membership plays no part in `addBlockedBy`. Step 7 then polls `TaskList(status=pending, no owner, no open blockedBy)` and dispatches everything available in one eager wave (see dispatcher.md: "If Phase 0 has 4 available tasks, all four ship together"). A cluster belonging to sprint 2 whose `Inter-cluster dependency:` is `none` is therefore available in wave 1 and ships before sprint 1 finishes. The boundary is decoration, exactly as step 9 warns.

Second, related defect: step 3 ("Resume check") item 5 says "Dispatch only the tasks whose verification did not exit 0". Step 6 creates every task in the plan as pending and step 7 dispatches every unblocked pending task, so nothing consumes that filter — a resumed session re-dispatches work that already passed.

The chosen fix for both, which you must implement and must not redesign: enforce at `TaskCreate`, not at the dispatch poll. Sprint edges become ordinary `addBlockedBy` edges, so the dispatch loop needs no change and `parser.md` rule 7 ("the dispatch loop then needs no special case") stays true. Tasks are created first — the edges need ids to reference — and the already-passing ones are marked completed immediately after, before step 7's first poll.

TASK
Two edits, both inside step 6. Nothing else.

EDIT 1 — replace the third bullet of step 6. It currently reads:

- `addBlockedBy`: intra-cluster dependencies (task ids) plus inter-cluster dependencies (all task ids in the blocking cluster)

Replace it with, verbatim:

- `addBlockedBy`: intra-cluster dependencies (task ids), inter-cluster dependencies (all task ids in the blocking cluster), and sprint dependencies — every task of the preceding sprint. The sprint edges are what make a boundary binding: without them step 7's eager poll ships a sprint N+1 cluster whose `Inter-cluster dependency:` is `none` in the first wave, and the boundary declared in step 9 never fires. A plan with one implicit sprint gains no edges, so nothing changes for it.

EDIT 2 — append this subsection at the end of step 6, immediately before the `### 7. Dispatch loop` heading. Content, verbatim:

**Then, before the first poll.** For every task the resume check (step 3) observed passing, `TaskUpdate(id, completed)`. Create first, mark second: the sprint and cluster edges need task ids to reference, so every task in the plan is created even when most are already done. Marking them completed before step 7's first poll is what consumes the resume filter — without it the loop re-dispatches work whose verification already exits 0.

This marking happens outside the dispatch loop. It does not trigger the sprint boundary stop of step 9, which fires only on a completion the dispatch loop produced. A resumed session that marks a whole finished sprint completed here has not reached that sprint's boundary — it passed it in an earlier session.

On a first run the resume check found nothing and this marks nothing.

CONSTRAINTS
- Do not modify any other file. In particular, do not edit dispatcher.md or parser.md — the fix is designed so that neither needs to change, and editing them would contradict it.
- Do not add or remove a numbered step. The file must still have exactly 12 `### ` headings when you are done.
- Do not renumber anything. Do not reword any step other than step 6.
- Do not touch step 3 or step 9 — they already describe the intended behaviour correctly; this task makes the mechanism match them.
- Do not commit or stage anything.
- Write in English.

RETURN
A summary of both edits, a one-line statement of what a sprint-2 task's `addBlockedBy` now contains, plus the output of:
grep -c '^### ' skills/execute-plans/SKILL.md && sed -n '/^### 6\./,/^### 7\./p' skills/execute-plans/SKILL.md

Return when this command exits 0 (the heading count fails if you added or removed a step):
cd /Users/fabiosiqueira/dev/projetos/kiss-skills && grep -q 'every task of the preceding sprint' skills/execute-plans/SKILL.md && grep -q 'before the first poll' skills/execute-plans/SKILL.md && grep -q 'make a boundary binding' skills/execute-plans/SKILL.md && grep -q 'outside the dispatch loop' skills/execute-plans/SKILL.md && test "$(grep -c '^### ' skills/execute-plans/SKILL.md)" -eq 12
````

### Cluster 2 — Narrative and rationale

**Inter-cluster dependency:** none

#### Task 2.1: Sync the execution-flow narrative [sonnet]

**Files:**
- Modify: `docs/usage/execution-flow.md`

**Diagnosis:** This doc is the operator-facing narrative of `execute-plans` and mirrors its numbered steps. Its step 7 lists the same three `addBlockedBy` inputs as the skill and omits sprint. Notably, the doc is already *ahead* of the skill: line 119 says sprint 2 "depends on sprint 1" and line 141 says `available = [2.1, 2.2] (sprint 2, now unblocked)`, describing edges the skill never creates. The task makes the mechanism explicit rather than inventing it.

**Verification:** `cd /Users/fabiosiqueira/dev/projetos/kiss-skills && grep -q 'sprint edges' docs/usage/execution-flow.md && grep -q 'before the first poll' docs/usage/execution-flow.md && test "$(grep -c '^### ' docs/usage/execution-flow.md)" -eq 13`

**Prompt for subagent (Agent tool):**
````
Project: /Users/fabiosiqueira/dev/projetos/kiss-skills
File to modify: docs/usage/execution-flow.md (this is the ONLY file you may touch)

This repo is the `fsa-tools` Claude Code plugin — markdown skills, no build, no tests. All content is written in English. Read the whole file first; it is the operator-facing narrative of skills/execute-plans/SKILL.md and mirrors its twelve numbered steps. Its voice is short, declarative, no hedging.

CONTEXT — the behaviour being documented

`fsa-tools:execute-plans` is gaining real enforcement of sprint boundaries. Another task is editing the SKILL.md itself; this task keeps the narrative in sync. The behaviour, which you must document and must not redesign:

- Step 7 ("TaskCreate per task") now builds `addBlockedBy` from three inputs, not two: intra-cluster dependencies, inter-cluster dependencies, and **sprint dependencies — every task of the preceding sprint**.
- Those sprint edges are what make a boundary binding. Without them the eager poll of step 8 ships a sprint N+1 cluster whose inter-cluster dependency is `none` in the very first wave, and the boundary stop of step 10 never fires. A plan with one implicit sprint gains no edges and behaves exactly as before.
- Enforcement lives at `TaskCreate`, not at the dispatch poll. The dispatch loop is unchanged.
- Also in step 7, after creating the tasks and before the first poll: every task the resume check (step 4) observed passing is marked `TaskUpdate(id, completed)`. Create first, mark second — the edges need ids to reference, so every task is created even when most are already done. This is what consumes the resume check's "dispatch only what failed"; without it the loop re-dispatches work whose verification already exits 0. On a first run it marks nothing.
- That marking happens outside the dispatch loop, so it does not trigger the sprint boundary stop of step 10. A resumed session that marks a whole finished sprint completed has not reached that sprint's boundary — it passed it in an earlier session. Your prose for EDIT 1 must say this in one sentence, and the EDIT 2 trace must not read as if the boundary fires again.

TASK
Two edits, nothing else.

EDIT 1 — in `### 7. TaskCreate per task`, the bullet list currently ends with:

- `addBlockedBy`: intra-cluster deps (task ids) plus inter-cluster deps (all tasks in the blocking cluster)

Extend that bullet to name sprint dependencies as the third input, and add two to four lines below the list, in this file's voice, covering: that the sprint edges are what make the boundary binding; that a plan with one implicit sprint gains none; and the create-then-mark rule that consumes the resume filter before the first poll.

Your prose must contain the exact phrases `sprint edges` and `before the first poll`. Everything else is yours to phrase.

EDIT 2 — in the `### Resuming a sliced plan` example, session 2 currently reads:

```
t=0    Resume check: continuity file present. Re-observe, don't trust it.
       Read HEAD and `git status --short`.
       Run verification for 1.1 and 1.2 → both exit 0. Nothing to re-dispatch.
       Worktree setup: existing worktree on fsa/<plan-slug> is reused.
t=1    TaskList → available = [2.1, 2.2]  (sprint 2, now unblocked)
```

The trace says "Nothing to re-dispatch" and "now unblocked" without showing the mechanism. Adjust the `t=0`/`t=1` lines so the trace shows tasks 1.1 and 1.2 being created and then marked completed before the first poll, which is both why nothing is re-dispatched and why 2.1/2.2 become available. Keep the existing trace style and keep the example the same length or within two lines of it.

CONSTRAINTS
- Do not modify any other file.
- Do not add or remove a `### ` heading. The file must still have exactly 13 of them (12 steps plus the resume example).
- Do not renumber the steps.
- Leave the `## Spot-check protocol` section and the first end-to-end example unchanged.
- Do not reword existing step content beyond the two edits named above.
- Do not commit or stage anything.
- Write in English.

RETURN
A summary of both edits, plus the output of:
grep -c '^### ' docs/usage/execution-flow.md && sed -n '/^### 7\./,/^### 8\./p' docs/usage/execution-flow.md

Return when this command exits 0:
cd /Users/fabiosiqueira/dev/projetos/kiss-skills && grep -q 'sprint edges' docs/usage/execution-flow.md && grep -q 'before the first poll' docs/usage/execution-flow.md && test "$(grep -c '^### ' docs/usage/execution-flow.md)" -eq 13
````

#### Task 2.2: Record why blockedBy edges, not a dispatch-time filter [sonnet]

**Files:**
- Modify: `docs/design/design-rationale.md`

**Diagnosis:** `design-rationale.md` is where this repo records design cuts under "Why X" headings — it already carries "Why eager dispatch" and "Why re-observation instead of recorded state". Enforcing the boundary through `addBlockedBy` rather than a filter in the dispatch poll is exactly that kind of cut, and without it the next reader of step 6 has no answer to "why do these edges exist if the session ends at the boundary anyway?".

**Verification:** `cd /Users/fabiosiqueira/dev/projetos/kiss-skills && grep -q 'Why blockedBy edges' docs/design/design-rationale.md && grep -qi 'dispatch-time filter' docs/design/design-rationale.md`

**Prompt for subagent (Agent tool):**
````
Project: /Users/fabiosiqueira/dev/projetos/kiss-skills
File to modify: docs/design/design-rationale.md (this is the ONLY file you may touch)

This repo is the `fsa-tools` Claude Code plugin — markdown skills, no build, no tests. All content is written in English. Read the whole file first. It is explanatory prose under "Why X" headings — no bullet lists. Match that voice and length: its sections run three to six short paragraphs.

CONTEXT — the decision being recorded

`fsa-tools:execute-plans` declares that sprint boundaries are "binding, not advisory", but nothing in the mechanism enforced that: step 6 (`TaskCreate`) built `addBlockedBy` from intra-cluster and inter-cluster dependencies only, and step 7's eager poll would ship a sprint N+1 cluster whose inter-cluster dependency is `none` in the first wave.

Two fixes were available:

1. **Sprint edges at `TaskCreate`** (chosen): every task of sprint N+1 gets `addBlockedBy` covering every task of sprint N. Sprint ordering becomes an ordinary dependency edge, indistinguishable from the ones the plan already declares.
2. **A filter in the dispatch poll** (rejected): step 7 narrows the available set to the current sprint before dispatching.

Why option 1 won:
- It reuses the dependency machinery that already exists rather than introducing a second, parallel ordering concept.
- The dispatch loop stays byte-identical. `dispatcher.md` needs no change, and its eager-dispatch rule ("if Phase 0 has 4 available tasks, all four ship together") stays unconditionally true — the availability set simply reflects reality.
- `parser.md` rule 7 states that an absent sprint slice "yields a single implicit sprint... the dispatch loop then needs no special case". Option 2 would have made that sentence false. Option 1 keeps it true: a one-sprint plan gains no edges and behaves exactly as before.
- One mechanism can be reasoned about at one place. A filter plus edges would be two things to keep in sync.

The honest cost: the edges are partly belt-and-braces. When a boundary is reached the session ends anyway (step 9), so in the normal path the edges never do visible work. They matter in the paths that are not normal — a resumed session where several sprints have pending tasks, or any future change that lets execution continue past a boundary. A rule enforced only by the code path that also stops execution is a rule that breaks the first time that path changes.

TASK
One edit. Append a new section immediately before the final `## KISS decisions` section, titled exactly:

## Why blockedBy edges instead of a dispatch-time filter

Write three to five short paragraphs from the context above, in this file's voice. Your prose must contain the phrase `dispatch-time filter` (the heading already supplies it, but use it in the body too where it reads naturally). Cover the two options, why the edges won, and the honest cost. Do not use bullet lists — this file is prose under headings.

CONSTRAINTS
- Do not modify any other file.
- Do not reword, reflow, or reorder any existing section.
- Do not touch `## KISS decisions` or any earlier "Why" section.
- Do not commit or stage anything.
- Write in English.

RETURN
A one-paragraph summary of what was inserted and where, plus the output of:
grep -n '^## ' docs/design/design-rationale.md

Return when this command exits 0:
cd /Users/fabiosiqueira/dev/projetos/kiss-skills && grep -q 'Why blockedBy edges' docs/design/design-rationale.md && grep -qi 'dispatch-time filter' docs/design/design-rationale.md
````

## Launch order (DAG resolved)

### Phase 0 — parallel

- Cluster 1 / Task 1.1
- Cluster 2 / Task 2.1
- Cluster 2 / Task 2.2

**Fan-out Phase 0: 3 parallel tasks**

No Phase 1. The plan completes in a single wave.

**Estimate: 3 tasks, fits one sprint.**

## Out of scope (deliberate)

- **Amending `docs/specs/2026-08-06-cross-session-execution-design.md`.** The gap originates there — S2 states the boundary rule in prose and never connects it to `TaskCreate`. Amending an already-executed spec mixes historical record with correction; the rationale belongs in `design-rationale.md`, which is built for it.
- **`CHANGELOG.md` and the version bump.** The operator's close-out flow owns them.
- **Runtime proof that enforcement works.** The DoD asserts prose, not behaviour — the running skill comes from the plugin cache, not from this repo. First real validation is the next sliced plan executed after the plugin is reinstalled.
- **Any change to `dispatcher.md` or `parser.md`.** Their staying unchanged is the point of the chosen approach, and the DoD asserts it.
