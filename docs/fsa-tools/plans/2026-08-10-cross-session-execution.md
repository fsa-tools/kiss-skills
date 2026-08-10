# Plan — Cross-Session Execution

## Metadata

- **Generated:** 2026-08-10
- **Worktree:** none

## Context

Project root: `/Users/fabiosiqueira/dev/projetos/kiss-skills`. This is the `fsa-tools` Claude Code plugin — plain markdown skills, no build step, no runtime, no test suite. The "code" is the skill prose itself, so every verification command here is a `grep`/`test` assertion against a file.

The plan implements `docs/specs/2026-08-06-cross-session-execution-design.md`: make a plan resumable across session boundaries, size plans against a session, and persist only what the repository cannot re-derive.

Three facts that shape the decomposition:

- **`Worktree: none` is deliberate.** Every verification is a `grep` against a path. Under a worktree the orchestrator's Bash cwd and the tree the subagents edited are not reliably the same tree, so greps can pass or fail against an untouched tree. With no build and no tests, a worktree buys nothing here and adds a failure class.
- **No self-modification in flight.** The running skill is loaded from the plugin cache (`~/.claude/plugins/cache/fsa-tools/fsa-tools/0.3.2/`); the edits land in this repo. The consequence for the DoD: it can only assert that the prose says X, never that resume works at runtime. Verification is intentionally textual.
- **R4 from the spec is a non-issue.** `grep -rin 'judge\|panel' skills/ docs/` returns nothing — the judge-panel anchor of `docs/specs/2026-06-06-workflow-native-evolution-design.md` never reached shipped prose. Removing the Approach-proposal step collides with nothing that exists. No task needed.

## Baseline (current state)

```bash
cd /Users/fabiosiqueira/dev/projetos/kiss-skills
grep -c 'Approach proposal' skills/writing-plans/SKILL.md   # 1 — step still present
grep -rc 'Sprint' skills/ docs/usage/ | grep -v ':0'        # no output — no sprint syntax anywhere
test -f skills/execute-plans/continuity.md; echo $?         # 1 — contract does not exist
grep -n 'already exists' skills/worktree/SKILL.md           # line 22 — unconditional abort blocks resume
```

## Objective

Add cross-session resume to the suite: an optional sprint slice in the plan schema, a re-observation-based resume step in `execute-plans`, a continuity artifact holding only non-re-derivable state, and a worktree helper that reuses instead of aborting. Every plan already written stays valid.

## Definition of Done (global)

Single verifiable command:

```bash
cd /Users/fabiosiqueira/dev/projetos/kiss-skills && set -e && \
grep -q '\*\*Sprints:\*\*' docs/usage/plan-schema.md && \
grep -q '## Sprint N — <Theme>' docs/usage/plan-schema.md && \
grep -q 'Sprints are optional' docs/usage/plan-schema.md && \
grep -q '\*\*Sprints:\*\*' skills/writing-plans/phase2-plan-template.md && \
grep -q '## Sprint N — <Theme>' skills/writing-plans/phase2-plan-template.md && \
grep -q 'Sprints are optional' skills/writing-plans/phase2-plan-template.md && \
test -f skills/execute-plans/continuity.md && \
grep -q 'docs/fsa-tools/continuity/<plan-slug>.md' skills/execute-plans/continuity.md && \
grep -q 'hint — not authoritative' skills/execute-plans/continuity.md && \
grep -q 'append-only' skills/execute-plans/continuity.md && \
! grep -q 'Approach proposal' skills/writing-plans/SKILL.md && \
! grep -q 'chosen approach' skills/writing-plans/SKILL.md && \
grep -q '## Sizing' skills/writing-plans/SKILL.md && \
grep -q '12 tasks per sprint' skills/writing-plans/SKILL.md && \
grep -q 'Estimate: N tasks' skills/writing-plans/phase1-shortlist.md && \
grep -q 'Resume check (cross-session)' skills/execute-plans/SKILL.md && \
grep -q 'Sprint boundary stop' skills/execute-plans/SKILL.md && \
grep -q 'continuity.md' skills/execute-plans/SKILL.md && \
grep -qi 're-observ' skills/execute-plans/SKILL.md && \
test "$(grep -c '^### ' skills/execute-plans/SKILL.md)" -eq 12 && \
grep -q 'sprints' skills/execute-plans/parser.md && \
grep -q 'Sprint {' skills/execute-plans/parser.md && \
grep -q 'implicit sprint' skills/execute-plans/parser.md && \
grep -q 'resume, not a collision' skills/worktree/SKILL.md && \
grep -qi 'reuse' skills/worktree/SKILL.md && \
grep -q 'Resume check' docs/usage/execution-flow.md && \
grep -q 'Sprint boundary' docs/usage/execution-flow.md && \
grep -q 'continuity' docs/design/principles.md && \
grep -qi 're-observ' docs/design/design-rationale.md && \
grep -qi 'cross-session' README.md && \
echo "DOD_PASS"
```

**Expected output:** `DOD_PASS`

## Policy (invariant)

- **English only.** Every file committed to this repo ships in the plugin — `SKILL.md`, helper `.md`, docs, README. No pt-BR in any edited file.
- **Metadata idiom beats the spec's shorthand.** The header is `- **Sprints:** N`, matching the existing `- **Worktree:** recommended`. Never the bare `sprints: N` written in the spec prose.
- **Surgical edits only.** Change exactly the lines the task names. No reflow of untouched paragraphs, no rewording of neighbouring prose, no section reordering, no "while I'm here" improvements.
- **Stay inside the declared Files list.** A task that believes it needs another file stops and reports instead of editing it.
- **Do not commit, do not stage.** The operator's close-out flow owns commits.
- **Never touch `CHANGELOG.md` or `.claude-plugin/plugin.json`.** Versioning and changelog belong to the close-out flow, deliberately out of this plan's scope.
- **Helper docs are referenced from `SKILL.md` by relative path, never inlined.**

## Dependency justification

None. No task declares `blockedBy`.

The one dependency that looks real — the parser (Task 3.2) needing the schema (Task 1.1) — is false. The parser consumes the *syntax*, and the syntax is fully specified in `docs/specs/2026-08-06-cross-session-execution-design.md` and reproduced verbatim inside every prompt that needs it. This is stub-and-integrate with the spec in the stub's role: Task 3.2 can complete before Task 1.1 exists without breaking anything.

File-overlap audit: ten tasks, ten disjoint file sets. Task 5.2 is the only one touching more than one file, and none of its three files appears in another task.

## Clusters

### Cluster 1 — Sprint and continuity contract

**Inter-cluster dependency:** none

#### Task 1.1: Add sprint syntax to the plan schema [sonnet]

**Files:**
- Modify: `docs/usage/plan-schema.md`

**Diagnosis:** The schema documents `- **Worktree:**` as the only optional metadata header and lists nine schema rules. Sprints need a tenth rule plus the section shape, added so that a plan without sprints parses exactly as today.

**Verification:** `cd /Users/fabiosiqueira/dev/projetos/kiss-skills && grep -q '\*\*Sprints:\*\*' docs/usage/plan-schema.md && grep -q '## Sprint N — <Theme>' docs/usage/plan-schema.md && grep -q 'Sprints are optional' docs/usage/plan-schema.md`

**Prompt for subagent (Agent tool):**
````
Project: /Users/fabiosiqueira/dev/projetos/kiss-skills
File to modify: docs/usage/plan-schema.md (this is the ONLY file you may touch)

This repo is the `fsa-tools` Claude Code plugin. It has no build step and no tests — the content is markdown prose. All content is written in English.

TASK
Add the optional sprint slice to the plan schema. Three edits, nothing else.

EDIT 1 — the Metadata block inside the template (around line 12).
It currently reads:

- **Generated:** YYYY-MM-DD
- **Worktree:** recommended | required | none

Add one line below it, exactly:

- **Sprints:** N

EDIT 2 — leave the `## Clusters` section of the template EXACTLY as it is. The template shows the common case, which has no sprints. Do NOT change the `### Cluster N`, `#### Task N.M`, or any other heading level — the parser keys on them and they must stay byte-identical.

EDIT 3 — the "Schema rules" numbered list. Append rule 10, verbatim:

10. **Sprints are optional and backwards compatible.** A plan that fits one session declares no sprints and is parsed exactly as before. When the plan is sliced, the header `- **Sprints:** N` is present and the `## Clusters` heading is replaced by one `## Sprint N — <Theme>` heading per sprint; each `### Cluster N` belongs to the nearest preceding sprint heading. A sprint boundary falls at a cluster end, never inside one, and two sprints never touch the same file. The Definition of Done stays single and global — sprints do not get their own.

Then add, immediately after the numbered rules list, this subsection verbatim (the fenced block keeps the sliced shape out of the main template, which stays the unsliced common case):

## Sprint header

`- **Sprints:** N` is present only when the plan is sliced across sessions. Absent means a single implicit sprint containing every cluster — the common case, which gains no syntax.

A sliced plan replaces the `## Clusters` heading with one heading per sprint:

## Sprint 1 — <Theme>

### Cluster 1 — <Theme>

### Cluster 2 — <Theme>

## Sprint 2 — <Theme>

### Cluster 3 — <Theme>

Cluster and task headings are unchanged. A cluster belongs to the nearest preceding sprint heading.

CONSTRAINTS
- Do not modify any other file.
- Do not reword, reflow, or reorder existing prose. Only insert the content above.
- Do not renumber or edit schema rules 1 through 9.
- Do not commit or stage anything.
- Write in English.

RETURN
A one-paragraph summary naming the three insert points, plus the output of:
grep -n 'Sprint' docs/usage/plan-schema.md

Return when this command exits 0:
cd /Users/fabiosiqueira/dev/projetos/kiss-skills && grep -q '\*\*Sprints:\*\*' docs/usage/plan-schema.md && grep -q '## Sprint N — <Theme>' docs/usage/plan-schema.md && grep -q 'Sprints are optional' docs/usage/plan-schema.md
````

#### Task 1.2: Mirror sprint syntax into the Phase 2 plan template [sonnet]

**Files:**
- Modify: `skills/writing-plans/phase2-plan-template.md`

**Diagnosis:** `phase2-plan-template.md` and `docs/usage/plan-schema.md` are near-duplicates — same template, same nine schema rules — and are the two artifacts `writing-plans` and `execute-plans` read. If sprint syntax lands in one and not the other, the skill that writes plans and the doc that defines them disagree. The insert content is identical by construction.

**Verification:** `cd /Users/fabiosiqueira/dev/projetos/kiss-skills && grep -q '\*\*Sprints:\*\*' skills/writing-plans/phase2-plan-template.md && grep -q '## Sprint N — <Theme>' skills/writing-plans/phase2-plan-template.md && grep -q 'Sprints are optional' skills/writing-plans/phase2-plan-template.md`

**Prompt for subagent (Agent tool):**
````
Project: /Users/fabiosiqueira/dev/projetos/kiss-skills
File to modify: skills/writing-plans/phase2-plan-template.md (this is the ONLY file you may touch)

This repo is the `fsa-tools` Claude Code plugin. It has no build step and no tests — the content is markdown prose. All content is written in English.

CONTEXT
This file is the canonical template `fsa-tools:writing-plans` follows when composing a plan. It is a deliberate near-duplicate of docs/usage/plan-schema.md — another task is adding the identical content there. The two must not drift, so transcribe the blocks below exactly as given rather than paraphrasing them.

TASK
Add the optional sprint slice. Three edits, nothing else.

EDIT 1 — the Metadata block inside the template (around line 12).
It currently reads:

- **Generated:** YYYY-MM-DD
- **Worktree:** recommended | required | none

Add one line below it, exactly:

- **Sprints:** N

EDIT 2 — leave the `## Clusters` section of the template EXACTLY as it is. The template shows the common case, which has no sprints. Do NOT change the `### Cluster N`, `#### Task N.M`, or any other heading level — the parser keys on them and they must stay byte-identical.

EDIT 3 — the "Schema rules" numbered list. Append rule 10, verbatim:

10. **Sprints are optional and backwards compatible.** A plan that fits one session declares no sprints and is parsed exactly as before. When the plan is sliced, the header `- **Sprints:** N` is present and the `## Clusters` heading is replaced by one `## Sprint N — <Theme>` heading per sprint; each `### Cluster N` belongs to the nearest preceding sprint heading. A sprint boundary falls at a cluster end, never inside one, and two sprints never touch the same file. The Definition of Done stays single and global — sprints do not get their own.

Then add, immediately after the numbered rules list and before the `## Worktree header values` section, this subsection verbatim:

## Sprint header

`- **Sprints:** N` is present only when the plan is sliced across sessions. Absent means a single implicit sprint containing every cluster — the common case, which gains no syntax.

A sliced plan replaces the `## Clusters` heading with one heading per sprint:

## Sprint 1 — <Theme>

### Cluster 1 — <Theme>

### Cluster 2 — <Theme>

## Sprint 2 — <Theme>

### Cluster 3 — <Theme>

Cluster and task headings are unchanged. A cluster belongs to the nearest preceding sprint heading.

CONSTRAINTS
- Do not modify any other file.
- Do not reword, reflow, or reorder existing prose. Only insert the content above.
- Do not renumber or edit schema rules 1 through 9.
- Do not touch the "Worktree header values" section.
- Do not commit or stage anything.
- Write in English.

RETURN
A one-paragraph summary naming the three insert points, plus the output of:
grep -n 'Sprint' skills/writing-plans/phase2-plan-template.md

Return when this command exits 0:
cd /Users/fabiosiqueira/dev/projetos/kiss-skills && grep -q '\*\*Sprints:\*\*' skills/writing-plans/phase2-plan-template.md && grep -q '## Sprint N — <Theme>' skills/writing-plans/phase2-plan-template.md && grep -q 'Sprints are optional' skills/writing-plans/phase2-plan-template.md
````

#### Task 1.3: Create the continuity artifact contract [sonnet]

**Files:**
- Create: `skills/execute-plans/continuity.md`

**Diagnosis:** The continuity file is state that `execute-plans` reads and writes at runtime, so its contract belongs beside the other runtime protocols (`parser.md`, `dispatcher.md`, `verification-loop.md`), not under `docs/`. Every existing `SKILL.md` references only siblings in its own skill directory. The file does not exist yet.

**Verification:** `cd /Users/fabiosiqueira/dev/projetos/kiss-skills && test -f skills/execute-plans/continuity.md && grep -q 'docs/fsa-tools/continuity/<plan-slug>.md' skills/execute-plans/continuity.md && grep -q 'hint — not authoritative' skills/execute-plans/continuity.md && grep -q 'append-only' skills/execute-plans/continuity.md`

**Prompt for subagent (Agent tool):**
````
Project: /Users/fabiosiqueira/dev/projetos/kiss-skills
File to create: skills/execute-plans/continuity.md (this is the ONLY file you may touch)

This repo is the `fsa-tools` Claude Code plugin — markdown skills, no build, no tests. All content is written in English. Helper protocol docs live beside the SKILL.md they support; read skills/execute-plans/parser.md first to match its tone, heading depth, and terseness.

TASK
Create skills/execute-plans/continuity.md with exactly this content:

# Continuity Protocol

The continuity file is the only state `fsa-tools:execute-plans` persists across sessions. It holds what the repository cannot re-derive, and nothing else.

## Location

<target-project>/docs/fsa-tools/continuity/<plan-slug>.md

`plan-slug` is derived exactly as in `skills/worktree/SKILL.md`: the plan filename with the date prefix and the `.md` suffix removed. One file per plan.

An absent file means a first run. The absence is not an error.

## Evidence boundary

**Re-derivable — never persisted, always re-observed on resume.** Branch, `HEAD`, dirty paths, per-task `verification:` exit codes, the global Definition of Done exit code, and task completion itself. The plan already carries every command needed to observe these; a recorded copy can only go stale.

Task completion deserves emphasis because it is the field most designs record. It is re-derivable by construction: a task is done exactly when its `verification:` command exits 0. A separate completion list would be a second source of truth that can disagree with the first — the failure mode this boundary exists to prevent.

**Non-re-derivable — persisted.** Decisions taken during execution, gotchas discovered, and observed run metrics. Nothing in the repository holds these, and they are exactly what a fresh session pays to rediscover.

## Format

# Continuity — <plan-slug>

## Decisions
- <decision taken during execution>
  Rationale: <why>
  Task: <N.M>

## Gotchas
- <thing that cost time and is not visible in the diff>
  Task: <N.M>

## Sprint progress (hint — not authoritative)
- Sprint 1: reported complete YYYY-MM-DD
- Sprint 2: not started

## Observed runs
- YYYY-MM-DD — tasks=N waves=N diff_lines=N duration=Nm

## Rules

1. `Decisions`, `Gotchas`, and `Observed runs` are append-only. A resumed session adds; it never rewrites history.
2. `Sprint progress` is a hint that orders the resume scan, never a source of truth. Authority on whether a task is done is its `verification:` exit code, re-run at resume time. If the hint and the exit codes disagree, the exit codes win and the hint is rewritten to match.
3. Never record git state, exit codes, file hashes, or a per-task completion list.
4. Write the file at the end of every run — sprint boundary stop, Definition of Done success, or escalation. On escalation this is what makes the failure resumable instead of lost.
5. The file lives in the target project's working tree and is versioned in git. There is no state store outside the working directory.

CONSTRAINTS
- Create only this file. Do not modify any other file — another task wires the reference into skills/execute-plans/SKILL.md.
- Reproduce the content above exactly. Do not add sections, examples, or commentary of your own.
- Do not commit or stage anything.
- Write in English.

RETURN
Confirmation that the file was created, plus the output of:
wc -l skills/execute-plans/continuity.md && grep -n '^#' skills/execute-plans/continuity.md

Return when this command exits 0:
cd /Users/fabiosiqueira/dev/projetos/kiss-skills && test -f skills/execute-plans/continuity.md && grep -q 'docs/fsa-tools/continuity/<plan-slug>.md' skills/execute-plans/continuity.md && grep -q 'hint — not authoritative' skills/execute-plans/continuity.md && grep -q 'append-only' skills/execute-plans/continuity.md
````

### Cluster 2 — writing-plans

**Inter-cluster dependency:** none

#### Task 2.1: Remove the Approach proposal, add the sizing step [sonnet]

**Files:**
- Modify: `skills/writing-plans/SKILL.md`

**Diagnosis:** The `Approach proposal` step (lines 41-47) asks the operator to re-choose parallelism-first, which `docs/design/principles.md` P1 already establishes as the default. It resolves to the recommendation every time — ceremony, not a decision. In its place the skill decides and records a one-line rationale, and Phase 1 gains a sizing step that gives the checkpoint a real decision when the plan is too big for one session.

**Verification:** `cd /Users/fabiosiqueira/dev/projetos/kiss-skills && ! grep -q 'Approach proposal' skills/writing-plans/SKILL.md && ! grep -q 'chosen approach' skills/writing-plans/SKILL.md && grep -q '## Sizing' skills/writing-plans/SKILL.md && grep -q '12 tasks per sprint' skills/writing-plans/SKILL.md`

**Prompt for subagent (Agent tool):**
````
Project: /Users/fabiosiqueira/dev/projetos/kiss-skills
File to modify: skills/writing-plans/SKILL.md (this is the ONLY file you may touch)

This repo is the `fsa-tools` Claude Code plugin — markdown skills, no build, no tests. All content is written in English. Read the whole file first; match its imperative, terse voice.

TASK
Four edits, nothing else.

EDIT 1 — DELETE the entire `## Approach proposal` section (currently lines 41-47), including its heading and all three paragraphs:

## Approach proposal

Before producing the shortlist, propose 2–3 approaches with trade-offs. Lead with your recommendation and explain why.

Examples of meaningful axes: sequential-safe vs. maximally parallel, fine-grained tasks vs. coarse clusters, thin path vs. full review coverage.

Present approaches conversationally. Wait for the operator to choose.

Rationale for the deletion (do not add it to the file): docs/design/principles.md P1 already fixes parallelism-first as the default and requires justification for every blockedBy. The skill holds the prior; asking the operator to re-pick it is ceremony that resolves to the recommendation.

EDIT 2 — in the `## Clarifying questions (one at a time)` section, the last line currently reads:

Wait for each answer before asking the next. Stop when you have enough to propose approaches.

Replace only that sentence's ending so it reads:

Wait for each answer before asking the next. Stop when you have enough to decompose.

EDIT 3 — in `## Phase 1 — Shortlist`, under `### Steps`, item 1 currently reads:

1. Compose a shortlist in the format defined in `phase1-shortlist.md`, reflecting the chosen approach.

Replace it with:

1. Compose a shortlist in the format defined in `phase1-shortlist.md`. The skill decides the decomposition itself — parallelism-first per `docs/design/principles.md` P1 — and records a one-line rationale for the shape chosen. Do not ask the operator to pick an approach.

Then, in the same `### Steps` list, insert a new item between the current item 2 (`Run the DAG audit...`) and item 3 (`Present the shortlist.`), renumbering the following items so the list stays 1..6:

3. Size the shortlist (see `## Sizing` below) and carry the estimate into the checkpoint.

EDIT 4 — insert a new `## Sizing` section immediately after the whole `## Phase 1 — Shortlist` section and immediately before `## Phase 2 — Full plan (after the operator confirms)`. Content, verbatim:

## Sizing

Run at the end of Phase 1, before the checkpoint. Count the tasks in the shortlist and compare against the limit. The default limit is **12 tasks per sprint**.

- **Within the limit:** the checkpoint carries one extra line — `Estimate: N tasks, fits one sprint.` No sprint syntax reaches the plan.
- **Over the limit:** the checkpoint additionally proposes sprint boundaries — the task count per sprint and the rationale for each cut — and asks the operator to confirm or adjust the cut.

Sprint boundary rules:

- A boundary falls at a cluster end, never inside one.
- No file overlap between sprints. Two sprints touching the same file make the second sprint's starting state unpredictable.
- The global Definition of Done stays single and unsplit. Sprints do not get their own.

Limit resolution: default 12, overridden by natural language in the invocation ("short sprints", "limit 8"). Never ask for the limit as a standalone question — in the common case the answer changes nothing, and a question whose answer never changes anything becomes the next rubber stamp.

Task count is the sizing unit because it is the only quantity this skill counts deterministically, and orchestrator context grows roughly linearly in it: about one dispatch prompt, one result summary, and one verification output per task. The 12 default is a starting guess, not a measurement.

CONSTRAINTS
- Do not modify any other file.
- Keep the `## Scope check` section exactly as it is.
- Keep the Phase 1 checkpoint (`**Checkpoint:** "Confirm shortlist?..."`) exactly as it is — it stays the cost gate in front of Phase 2.
- Do not reword, reflow, or reorder any prose other than the edits named above.
- Do not commit or stage anything.
- Write in English.

RETURN
A one-paragraph summary of the four edits, plus the output of:
grep -n '^#' skills/writing-plans/SKILL.md

Return when this command exits 0:
cd /Users/fabiosiqueira/dev/projetos/kiss-skills && ! grep -q 'Approach proposal' skills/writing-plans/SKILL.md && ! grep -q 'chosen approach' skills/writing-plans/SKILL.md && grep -q '## Sizing' skills/writing-plans/SKILL.md && grep -q '12 tasks per sprint' skills/writing-plans/SKILL.md
````

#### Task 2.2: Add the sizing line to the shortlist format [haiku]

**Files:**
- Modify: `skills/writing-plans/phase1-shortlist.md`

**Diagnosis:** `phase1-shortlist.md` defines the artifact presented at the Phase 1 checkpoint. The sizing step in `SKILL.md` produces an estimate line and, when over the limit, sprint blocks — neither is expressible in the current template.

**Verification:** `cd /Users/fabiosiqueira/dev/projetos/kiss-skills && grep -q 'Estimate: N tasks' skills/writing-plans/phase1-shortlist.md && grep -q 'Sprint 1' skills/writing-plans/phase1-shortlist.md`

**Prompt for subagent (Agent tool):**
````
Project: /Users/fabiosiqueira/dev/projetos/kiss-skills
File to modify: skills/writing-plans/phase1-shortlist.md (this is the ONLY file you may touch)

This repo is the `fsa-tools` Claude Code plugin — markdown skills, no build, no tests. All content is written in English.

TASK
Two edits, nothing else.

EDIT 1 — in the `## Template` code block, append this line at the very end of the block content (after the last `- DoD: <shell command>` line, separated by one blank line):

Estimate: N tasks, fits one sprint.

EDIT 2 — insert a new section immediately after the `## Template` section and immediately before `## Task grammar`. Content, verbatim:

## Sizing line

Every shortlist ends with one estimate line:

Estimate: N tasks, fits one sprint.

When the task count exceeds the limit (default 12), the shortlist is presented sliced instead, and the estimate names the cut:

Sprint 1 — <Theme> (N tasks)
    Cluster 1 — <Theme>
    Cluster 2 — <Theme>

Sprint 2 — <Theme> (N tasks)
    Cluster 3 — <Theme>

Estimate: N tasks over 2 sprints. Boundary after Cluster 2 — <one-line rationale>.

A boundary falls at a cluster end, never inside one, and two sprints never touch the same file. The Definition of Done stays single and global.

CONSTRAINTS
- Do not modify any other file.
- Do not touch the `## Task grammar`, `## Model selection`, or `## Prompt checklist (Phase 2)` sections.
- Do not reword, reflow, or reorder existing prose. Only insert the content above.
- Do not commit or stage anything.
- Write in English.

RETURN
A short summary of both edits, plus the output of:
grep -n 'Estimate\|Sprint' skills/writing-plans/phase1-shortlist.md

Return when this command exits 0:
cd /Users/fabiosiqueira/dev/projetos/kiss-skills && grep -q 'Estimate: N tasks' skills/writing-plans/phase1-shortlist.md && grep -q 'Sprint 1' skills/writing-plans/phase1-shortlist.md
````

### Cluster 3 — execute-plans

**Inter-cluster dependency:** none

#### Task 3.1: Resume check, sprint boundary stop, continuity write [opus] +reviewer

**Files:**
- Modify: `skills/execute-plans/SKILL.md`

**Diagnosis:** The deepest change. `execute-plans` today runs nine steps and assumes a first run: no resume, no sprint awareness, no persisted state. Three new steps are needed, and the existing ones renumber around them. The resume step must re-observe rather than trust — recorded completion would be a second source of truth that can disagree with the `verification:` exit codes.

**Verification:** `cd /Users/fabiosiqueira/dev/projetos/kiss-skills && grep -q 'Resume check (cross-session)' skills/execute-plans/SKILL.md && grep -q 'Sprint boundary stop' skills/execute-plans/SKILL.md && grep -q 'continuity.md' skills/execute-plans/SKILL.md && grep -qi 're-observ' skills/execute-plans/SKILL.md && test "$(grep -c '^### ' skills/execute-plans/SKILL.md)" -eq 12`

**Prompt for subagent (Agent tool):**
````
Project: /Users/fabiosiqueira/dev/projetos/kiss-skills
File to modify: skills/execute-plans/SKILL.md (this is the ONLY file you may touch)

This repo is the `fsa-tools` Claude Code plugin — markdown skills, no build, no tests. All content is written in English. Read the whole file first, plus skills/execute-plans/parser.md, to match voice and structure.

CONTEXT
The file currently has nine numbered steps under `## Steps (in order)`:
1. Resolve the plan file
2. Parse the plan
3. Worktree setup
4. Hold the Definition of Done
5. TaskCreate per task
6. Dispatch loop
7. Spot-check after each wave
8. Definition of Done gate + global review
9. Final report

Cross-session execution adds three steps. The final numbering must be:
1. Resolve the plan file            (unchanged)
2. Parse the plan                   (one bullet added)
3. Resume check (cross-session)     (NEW)
4. Worktree setup                   (renumbered from 3, content unchanged)
5. Hold the Definition of Done      (renumbered from 4, content unchanged)
6. TaskCreate per task              (renumbered from 5, content unchanged)
7. Dispatch loop                    (renumbered from 6, content unchanged)
8. Spot-check after each wave       (renumbered from 7, content unchanged)
9. Sprint boundary stop             (NEW)
10. Continuity write                (NEW)
11. Definition of Done gate + global review  (renumbered from 8, one sentence added)
12. Final report                    (renumbered from 9, content unchanged)

Note that steps 8 and 9 of the current file contain cross-references to "step 8" — update those to the new numbers.

Another task is creating skills/execute-plans/continuity.md, the protocol this file will reference. Assume it exists; reference it by relative filename, never inline its content. Its contract, which you may rely on: the continuity file lives at `docs/fsa-tools/continuity/<plan-slug>.md` in the target project; its `Decisions`, `Gotchas`, and `Observed runs` sections are append-only; its `Sprint progress` section is a hint that orders the resume scan and is never a source of truth.

TASK
Five edits, nothing else.

EDIT 1 — in step "2. Parse the plan", add one bullet to the extraction list, after the `Worktree header` bullet:

- Sprint slice (`- **Sprints:** N` header and `## Sprint N` sections). Absent → a single implicit sprint containing every cluster.

EDIT 2 — insert a new step 3 between "Parse the plan" and "Worktree setup". Content, verbatim:

### 3. Resume check (cross-session)

A plan can outlive the session that executes it. This step finds out what is already done by looking at the repository, never by trusting a record. Full contract in `continuity.md`.

1. Look for `docs/fsa-tools/continuity/<plan-slug>.md`. Absent → this is a first run; continue to step 4 unchanged. The absence is not an error.
2. Present → read it for decisions, gotchas, and the sprint-progress hint. Then **re-observe rather than trust**: read `HEAD` and `git status --short`, and run the `verification:` command of every task in the plan. Exit 0 means the task is done. That exit code is the only completion signal — there is no stored completion list, by design.
3. The `Sprint progress` section orders the scan so the likely-pending sprint is checked first. It never shortcuts it.
4. Report divergences explicitly and escalate to the operator rather than deciding silently: a task the hint calls complete whose verification now fails, a dirty working tree, an unexpected branch.
5. Dispatch only the tasks whose verification did not exit 0, in DAG order, stopping at the first sprint boundary that follows them.

Re-running every verification is not free on a large plan. It is the price of having one source of truth instead of two, and verification commands are cheap by construction. If it proves slow, the fix is faster verification commands — not a cached completion list.

EDIT 3 — insert a new step 9 between "Spot-check after each wave" and the Definition of Done gate. Content, verbatim:

### 9. Sprint boundary stop

Sprint boundaries are binding, not advisory. A boundary the executor may ignore is decoration.

After the last task of a **non-final** sprint completes:

1. Run the per-wave spot-check (step 8).
2. Write the continuity file (step 10).
3. End the session with `Sprint N/M complete — open a fresh session to continue`.

The global Definition of Done does not run here. It is global and single, so a partial run would be meaningless. `fsa-tools:finish-branch` is not invoked either — it runs only after the final sprint's Definition of Done passes.

A plan with no sprint slice has exactly one sprint, which is also the final one, so this step never fires.

EDIT 4 — insert a new step 10 immediately after step 9. Content, verbatim:

### 10. Continuity write

At the end of every run — sprint boundary stop, Definition of Done success, or escalation — append to `docs/fsa-tools/continuity/<plan-slug>.md` following `continuity.md`: decisions taken during execution, gotchas discovered, the sprint-progress hint, and one observed-run line (`tasks=`, `waves=`, `diff_lines=`, `duration=`).

Persist only what the repository cannot re-derive. Never record git state, exit codes, file hashes, or a per-task completion list — those are re-observed at resume time (step 3).

On escalation this is what makes the failure resumable instead of lost.

EDIT 5 — in the Definition of Done gate step (now step 11), add this sentence at the end of the intro line "Once the dispatch loop reports all tasks complete, before declaring done:":

This gate runs on the final sprint only.

CONSTRAINTS
- Do not modify any other file.
- Renumber steps as specified and fix the internal cross-references, but do not otherwise reword the content of existing steps.
- Add `- Continuity protocol: `continuity.md`` to the `## Internal references` list at the end of the file.
- Do not commit or stage anything.
- Write in English.

RETURN
A summary of the five edits and every cross-reference you renumbered, plus the output of:
grep -n '^###' skills/execute-plans/SKILL.md

Return when this command exits 0 (the last assertion counts `### ` headings — it fails if the renumbering left a duplicate or a gap):
cd /Users/fabiosiqueira/dev/projetos/kiss-skills && grep -q 'Resume check (cross-session)' skills/execute-plans/SKILL.md && grep -q 'Sprint boundary stop' skills/execute-plans/SKILL.md && grep -q 'continuity.md' skills/execute-plans/SKILL.md && grep -qi 're-observ' skills/execute-plans/SKILL.md && test "$(grep -c '^### ' skills/execute-plans/SKILL.md)" -eq 12
````

#### Task 3.2: Add sprint parsing to the parser protocol [sonnet]

**Files:**
- Modify: `skills/execute-plans/parser.md`

**Diagnosis:** The parser extracts four fields and knows nothing about sprints. It must learn one optional header field and one grouping rule, written so that a plan without sprints yields a single implicit sprint and the dispatch loop needs no special case.

**Verification:** `cd /Users/fabiosiqueira/dev/projetos/kiss-skills && grep -q 'sprints' skills/execute-plans/parser.md && grep -q 'Sprint {' skills/execute-plans/parser.md && grep -q 'implicit sprint' skills/execute-plans/parser.md`

**Prompt for subagent (Agent tool):**
````
Project: /Users/fabiosiqueira/dev/projetos/kiss-skills
File to modify: skills/execute-plans/parser.md (this is the ONLY file you may touch)

This repo is the `fsa-tools` Claude Code plugin — markdown skills, no build, no tests. All content is written in English. Read the whole file first; it is terse and table-driven. Match that.

CONTEXT
Plans gain an optional sprint slice. The exact syntax, which you must parse and must not redesign:

- An optional metadata header line: `- **Sprints:** N`, sitting alongside `- **Worktree:** recommended`.
- When present, the `## Clusters` heading is replaced by one `## Sprint N — <Theme>` heading per sprint, and each `### Cluster N` belongs to the nearest preceding sprint heading.
- The `### Cluster N` and `#### Task N.M` headings are unchanged, so all existing parsing rules keep working byte-for-byte.

TASK
Three edits, nothing else.

EDIT 1 — add a row to the "Fields to extract" table, after the `worktree_header` row:

| `sprints` | Line `- **Sprints:**` in the Metadata section, plus `## Sprint N` sections | `Sprint[]` |

EDIT 2 — add a "Sprint shape" section immediately before the existing "## Cluster shape" section:

## Sprint shape

Sprint {
  id: number          // N in "## Sprint N"
  name: string        // text after "— "; empty for the implicit sprint
  clusters: number[]  // ids of the clusters under this sprint heading
}

EDIT 3 — append these parsing rules to the numbered "## Parsing rules" list, continuing the existing numbering:

6. **Sprint membership.** A cluster belongs to the nearest preceding `## Sprint N` heading.
7. **Absent sprint slice.** No `- **Sprints:**` header and no `## Sprint N` headings → yield a single implicit sprint with id 1 containing every cluster in order. The dispatch loop then needs no special case: every plan has at least one sprint, and a plan with one sprint has no non-final sprint, so no boundary stop ever fires.
8. **Disagreement between the header and the sections.** If `- **Sprints:** N` does not match the number of `## Sprint N` headings, escalate to the operator. Do not guess which one is right.

Use the same fenced-code style the file already uses for the Cluster and Task shapes.

CONSTRAINTS
- Do not modify any other file.
- Do not change the existing Cluster shape, Task shape, or rules 1 through 5.
- Do not reword, reflow, or reorder existing prose. Only insert the content above.
- Do not commit or stage anything.
- Write in English.

RETURN
A short summary of the three edits, plus the output of:
grep -n -i 'sprint' skills/execute-plans/parser.md

Return when this command exits 0:
cd /Users/fabiosiqueira/dev/projetos/kiss-skills && grep -q 'sprints' skills/execute-plans/parser.md && grep -q 'Sprint {' skills/execute-plans/parser.md && grep -q 'implicit sprint' skills/execute-plans/parser.md
````

### Cluster 4 — worktree

**Inter-cluster dependency:** none

#### Task 4.1: Reuse the worktree on a matching branch [sonnet]

**Files:**
- Modify: `skills/worktree/SKILL.md`

**Diagnosis:** Step 6 aborts unconditionally when the worktree exists (`skills/worktree/SKILL.md:22`). Since the worktree name is derived deterministically from the plan slug, a second session on the same plan always hits that abort — the helper actively blocks the resume this plan is adding. Existing-and-matching is a resume; existing-with-a-different-branch is still a genuine collision.

**Verification:** `cd /Users/fabiosiqueira/dev/projetos/kiss-skills && grep -q 'resume, not a collision' skills/worktree/SKILL.md && grep -qi 'reuse' skills/worktree/SKILL.md`

**Prompt for subagent (Agent tool):**
````
Project: /Users/fabiosiqueira/dev/projetos/kiss-skills
File to modify: skills/worktree/SKILL.md (this is the ONLY file you may touch)

This repo is the `fsa-tools` Claude Code plugin — markdown skills, no build, no tests. All content is written in English. Read the whole file first; it is a short numbered flow.

CONTEXT
Worktree naming is deterministic: path `../.worktrees/<repo-name>-<plan-slug>`, branch `fsa/<plan-slug>`. Step 6 currently aborts whenever the worktree already exists, which blocks a second session from resuming the same plan. An existing worktree on the matching branch is a resume, not a collision. An existing worktree on a different branch still is one.

TASK
Replace step 6 of the `## Flow` list. It currently reads:

6. If the worktree already exists: abort with `Worktree already exists at <path>. Options: 'git worktree remove <path>' to remove, or reuse explicitly.`

Replace it with, verbatim:

6. If the worktree already exists, check its branch with `git -C ../.worktrees/<repo-name>-<plan-slug> branch --show-current`:
   - Branch is `fsa/<plan-slug>` → **reuse it**. Return its absolute path and branch name to the caller exactly as step 5 would. This is a resume, not a collision: worktree naming is deterministic, so a second session on the same plan is expected to land here.
   - Branch differs → abort with `Worktree already exists at <path> on branch <branch>. Options: 'git worktree remove <path>' to remove, or reuse explicitly.`

Then add one line to the `## Return` section, after the existing sentence:

On reuse, the return value is identical to a fresh creation — the caller does not distinguish the two cases.

CONSTRAINTS
- Do not modify any other file.
- Do not change steps 1 through 5 or the frontmatter.
- Do not reword, reflow, or reorder any other prose.
- Do not commit or stage anything.
- Write in English.

RETURN
A short summary of the change, plus the output of:
cat skills/worktree/SKILL.md

Return when this command exits 0:
cd /Users/fabiosiqueira/dev/projetos/kiss-skills && grep -q 'resume, not a collision' skills/worktree/SKILL.md && grep -qi 'reuse' skills/worktree/SKILL.md
````

### Cluster 5 — Narrative docs

**Inter-cluster dependency:** none

#### Task 5.1: Update the execution flow doc [sonnet]

**Files:**
- Modify: `docs/usage/execution-flow.md`

**Diagnosis:** `execution-flow.md` is the operator-facing narrative of `execute-plans` and mirrors its numbered steps one-for-one. It currently describes nine steps and a first-run-only flow; it must gain the resume check, the sprint boundary stop, and the continuity write, and its end-to-end example must show a boundary stop.

**Verification:** `cd /Users/fabiosiqueira/dev/projetos/kiss-skills && grep -q 'Resume check' docs/usage/execution-flow.md && grep -q 'Sprint boundary' docs/usage/execution-flow.md && grep -q 'continuity' docs/usage/execution-flow.md`

**Prompt for subagent (Agent tool):**
````
Project: /Users/fabiosiqueira/dev/projetos/kiss-skills
File to modify: docs/usage/execution-flow.md (this is the ONLY file you may touch)

This repo is the `fsa-tools` Claude Code plugin — markdown skills, no build, no tests. All content is written in English. Read the whole file first; it is the operator-facing narrative of skills/execute-plans/SKILL.md and mirrors its numbered steps.

CONTEXT
`fsa-tools:execute-plans` is gaining cross-session execution. Another task is editing the SKILL.md itself; this task keeps the narrative doc in sync. The behaviour being documented:

- A plan may declare an optional sprint slice: header `- **Sprints:** N` plus `## Sprint N — <Theme>` sections wrapping clusters. Absent means a single implicit sprint containing every cluster — every plan written so far.
- Before worktree setup, the skill checks `docs/fsa-tools/continuity/<plan-slug>.md`. Absent → first run. Present → re-observe rather than trust: read HEAD and `git status --short`, then run the `verification:` command of every task. Exit 0 is the only completion signal; there is no stored completion list. The `Sprint progress` section of the continuity file only orders the scan.
- Sprint boundaries are binding. After the last task of a non-final sprint, the skill spot-checks, writes the continuity file, and ends the session with `Sprint N/M complete — open a fresh session to continue`. The global Definition of Done does not run at a boundary — it is global and single.
- The continuity file is appended at the end of every run: boundary stop, DoD success, or escalation. It holds decisions, gotchas, the sprint-progress hint, and one observed-run line. It never holds git state, exit codes, or a completion list.

TASK
Three edits.

EDIT 1 — the `## Steps` section currently has nine numbered subsections. Insert three new ones and renumber so the final order is:

1. Resolve the plan file        (unchanged)
2. Announce                     (unchanged)
3. Parse                        (add a bullet for the sprint slice)
4. Resume check                 (NEW — describe the four points above in the file's voice, 4-6 lines)
5. Worktree setup               (renumbered; add one sentence: an existing worktree on branch `fsa/<plan-slug>` is reused, which is what makes a second session possible)
6. Hold the Definition of Done  (renumbered, unchanged)
7. TaskCreate per task          (renumbered, unchanged)
8. Dispatch loop                (renumbered, unchanged)
9. Spot-check after each wave   (renumbered, unchanged)
10. Sprint boundary stop        (NEW — 3-5 lines)
11. Continuity write            (NEW — 3-5 lines)
12. Final report                (renumbered; note the DoD gate runs on the final sprint only)

Write the body of the three new sections yourself, in this file's existing voice: short, declarative, no hedging. Do not copy SKILL.md verbatim — this doc is narrative, SKILL.md is procedure.

The three new headings, however, are NOT yours to phrase. They must read exactly:

### 4. Resume check
### 10. Sprint boundary stop
### 11. Continuity write

Prose in-voice, headings verbatim.

EDIT 2 — the `## End-to-end example` currently shows a 4-task plan finishing in two waves. Keep it, and add a second short example below it titled `### Resuming a sliced plan`: a plan of 2 sprints where session 1 stops at the boundary and session 2 re-observes (verification of sprint 1's tasks exits 0, so nothing is re-dispatched) and finishes. Use the same `t=0 / t=1` trace style as the existing example.

EDIT 3 — leave the `## Spot-check protocol` section unchanged.

CONSTRAINTS
- Do not modify any other file.
- Do not reword existing step content beyond the renumbering and the two sentences named above.
- Do not commit or stage anything.
- Write in English.

RETURN
A summary of the new sections and the renumbering, plus the output of:
grep -n '^#' docs/usage/execution-flow.md

Return when this command exits 0:
cd /Users/fabiosiqueira/dev/projetos/kiss-skills && grep -q 'Resume check' docs/usage/execution-flow.md && grep -q 'Sprint boundary' docs/usage/execution-flow.md && grep -q 'continuity' docs/usage/execution-flow.md
````

#### Task 5.2: Update principles, rationale and README [sonnet]

**Files:**
- Modify: `docs/design/principles.md`
- Modify: `docs/design/design-rationale.md`
- Modify: `README.md`

**Diagnosis:** `principles.md` KISS states "No state outside the working directory and git" — the continuity file is the first persisted state and must be named as the one permitted exception, still inside the working tree. `design-rationale.md` explains the why behind every design choice and has no entry for re-observation over recorded state. `README.md` never mentions cross-session execution. Three files, one narrative, no overlap with any other task.

**Verification:** `cd /Users/fabiosiqueira/dev/projetos/kiss-skills && grep -q 'continuity' docs/design/principles.md && grep -qi 're-observ' docs/design/design-rationale.md && grep -qi 'cross-session' README.md`

**Prompt for subagent (Agent tool):**
````
Project: /Users/fabiosiqueira/dev/projetos/kiss-skills
Files to modify (these three, and no others):
- docs/design/principles.md
- docs/design/design-rationale.md
- README.md

This repo is the `fsa-tools` Claude Code plugin — markdown skills, no build, no tests. All content is written in English. Read all three files first and match each one's voice: principles.md is rule-shaped and terse, design-rationale.md is explanatory prose under "Why X" headings, README.md is short and factual.

CONTEXT — the feature being documented
`fsa-tools` gains cross-session execution:
- A plan may declare an optional sprint slice (header `- **Sprints:** N` plus `## Sprint N — <Theme>` sections). Absent means one implicit sprint — every plan written so far stays valid.
- `execute-plans` stops at every sprint boundary and ends the session. Boundaries are binding, not advisory.
- One continuity file per plan, at `docs/fsa-tools/continuity/<plan-slug>.md` in the target project — markdown, in the working tree, versioned in git.
- The evidence boundary: branch, HEAD, dirty paths, per-task verification exit codes, the DoD exit code, and task completion itself are **re-derivable** and are never persisted — they are re-observed at resume time by re-running each task's `verification:` command. Only decisions, gotchas, and observed run metrics are persisted, because nothing in the repository holds them.
- Rationale for that cut: a recorded completion list is a second source of truth that can disagree with the exit codes. Re-running the verification *is* the validation; a stored exit code is a claim with a shelf-life.

EDIT 1 — docs/design/principles.md
The `## KISS` section ends with the bullet `- No state outside the working directory and git.` Replace that single bullet with:

- No state outside the working directory and git. The one persisted artifact is the continuity file (`docs/fsa-tools/continuity/<plan-slug>.md`) — plain markdown in the target project's working tree, versioned in git. No database, no schema validator, no store outside the repo.
- Persist only the non-re-derivable. Task completion, exit codes, and git state are re-observed at resume time, never recorded. A recorded copy is a second source of truth that can disagree with the first.

Add nothing else to this file.

EDIT 2 — docs/design/design-rationale.md
Append a new section immediately before the final `## KISS decisions` section, titled:

## Why re-observation instead of recorded state

Three to five short paragraphs covering: what a resumed session needs to know; why task completion is re-derivable by construction (a task is done exactly when its `verification:` command exits 0 — that is the field's entire purpose); why a second completion list is the failure mode the boundary exists to prevent; and what the continuity file therefore holds instead (decisions, gotchas, observed run metrics — the things a fresh session would otherwise pay to rediscover). Acknowledge the cost honestly: re-running every verification is not free, and it is the price of one source of truth instead of two.

Write it in this file's voice. Do not add bullet lists — the file is prose under headings.

EDIT 3 — README.md
Add a new `## Cross-session execution` section between the existing `## Skills` and `## Documentation` sections. Four to six lines covering: a plan that does not fit one session declares sprints; `execute-plans` stops at each boundary and the operator opens a fresh session; on resume the skill re-observes the repository instead of trusting a record; the continuity file holds only decisions, gotchas, and run metrics; plans without sprints are unaffected.

Do NOT edit the `## Skills` table, the `## Status` section, or any other part of README.md.

CONSTRAINTS
- Modify only the three files listed. Do not touch CHANGELOG.md or .claude-plugin/plugin.json — versioning belongs to the operator's close-out flow.
- Do not reword, reflow, or reorder any prose other than the edits named above.
- Do not commit or stage anything.
- Write in English.

RETURN
A short summary per file naming what was inserted, plus the output of:
grep -n 'continuity' docs/design/principles.md && grep -n '^## ' docs/design/design-rationale.md && grep -n '^## ' README.md

Return when this command exits 0:
cd /Users/fabiosiqueira/dev/projetos/kiss-skills && grep -q 'continuity' docs/design/principles.md && grep -qi 're-observ' docs/design/design-rationale.md && grep -qi 'cross-session' README.md
````

## Launch order (DAG resolved)

### Phase 0 — parallel

- Cluster 1 / Task 1.1
- Cluster 1 / Task 1.2
- Cluster 1 / Task 1.3
- Cluster 2 / Task 2.1
- Cluster 2 / Task 2.2
- Cluster 3 / Task 3.1
- Cluster 3 / Task 3.2
- Cluster 4 / Task 4.1
- Cluster 5 / Task 5.1
- Cluster 5 / Task 5.2

**Fan-out Phase 0: 10 parallel tasks**

No Phase 1. The plan completes in a single wave.

**Estimate: 10 tasks, fits one sprint.**

## Out of scope (deliberate)

- **`CHANGELOG.md` and the version bump in `.claude-plugin/plugin.json`.** The spec's rollout step 6 lists them; the operator's close-out flow owns them. Two owners for the same file is a real conflict.
- **`README.md:35` says "`/goal` termination".** Stale since 0.3.2, and Task 5.2 touches this file — but fixing it was not requested, so the prompt explicitly forbids editing the Skills table.
- **R4 from the spec.** No task: the judge-panel anchor exists only in `docs/specs/2026-06-06-workflow-native-evolution-design.md`, never in shipped prose.
- **Runtime proof that resume works.** The DoD asserts prose, not behaviour — the running skill comes from the plugin cache, not from this repo. First real validation is the next plan executed after the plugin is reinstalled.
