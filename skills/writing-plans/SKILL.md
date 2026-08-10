---
name: writing-plans
description: Write a parallel execution plan in two phases — quick shortlist with an operator checkpoint, then deep investigation with full subagent prompts. Plans are saved to docs/fsa-tools/plans/. Use before any non-trivial implementation, refactor, or multi-file cleanup.
---

# fsa-tools:writing-plans

## Announce

"Using `fsa-tools:writing-plans` in two-phase mode."

## Pre-flight

1. Verify `docs/fsa-tools/plans/` exists in the current project. Create it if missing.
2. Read context: `pwd`, `git log --oneline -10`, `git status --short`.
3. Read any relevant spec or design doc in `docs/` that matches the topic.

## Exploration (before asking anything)

Understand the terrain before asking questions. This is not optional.

1. Read the files most likely affected by the task (grep for key symbols, read entry points).
2. Identify: what already exists, what is missing, what is unclear.
3. Form a working hypothesis of what the plan will decompose into.

## Scope check

Before proceeding, assess scale:

- If the request describes multiple independent subsystems or phases, flag it. Don't refine details of a project that needs decomposition first. Help the operator split into sub-plans; brainstorm the first sub-plan through the normal flow. Each sub-plan gets its own plan → execution cycle.
- If the scope is appropriate, continue.

## Clarifying questions (one at a time)

Ask questions one at a time — never a list. Each question as multiple-choice with a recommended option.

Focus on: what problem is being solved, hard constraints, success criteria, preferences that affect decomposition (worktree, model selection, review opt-in).

Wait for each answer before asking the next. Stop when you have enough to decompose.

## Phase 1 — Shortlist

### Goal

Produce a high-level decomposition (clusters and tasks with a DAG sketch). Stop at a checkpoint so the operator can confirm scope before deep work begins.

### Steps

1. Compose a shortlist in the format defined in `phase1-shortlist.md`. The skill decides the decomposition itself — parallelism-first per `docs/design/principles.md` P1 — and records a one-line rationale for the shape chosen. Do not ask the operator to pick an approach.
2. Run the DAG audit (`dag-audit.md`) on every declared dependency.
3. Size the shortlist (see `## Sizing` below) and carry the estimate into the checkpoint.
4. Present the shortlist.
5. **Checkpoint:** "Confirm shortlist? If yes, I move to deep investigation (file reads, full prompts). If you want to change X/Y/Z, I adjust first."
6. Wait for confirmation.

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

## Phase 2 — Full plan (after the operator confirms)

### Goal

For each task on the confirmed shortlist: read the relevant files, write a self-contained subagent prompt, and define an executable verification command.

### Steps

1. For each task:
   a. Read target files (Read tool) to confirm the diagnosis.
   b. Write the subagent prompt against the checklist in `phase1-shortlist.md` ("Prompt checklist").
   c. Check the prompt against the anti-patterns in `dag-audit.md`.
   d. Define a verification shell command that returns exit 0 when the task is done.
2. Compose the final plan using the template in `phase2-plan-template.md`.
3. Self-review inline:
   - Scan for placeholders (`TBD`, `TODO`, `implement later`, `similar to Task N`).
   - Check type and name consistency across tasks.
   - Confirm every declared dependency has a justification in the "Dependency justification" section.
4. Save to `docs/fsa-tools/plans/YYYY-MM-DD-<topic>.md`.
5. **Final checkpoint:** "Plan saved at `<path>`. Review and tell me if anything needs to change. When ready, open a fresh session and run `/fsa-tools:execute-plans`."

## Terminal state

Do not invoke `fsa-tools:execute-plans`. The operator does that in a fresh session, so execution starts with a clean context window.

## Internal references

- Shortlist format: `phase1-shortlist.md`
- Final plan template: `phase2-plan-template.md`
- DAG audit and anti-patterns: `dag-audit.md`
