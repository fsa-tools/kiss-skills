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

Wait for each answer before asking the next. Stop when you have enough to propose approaches.

## Approach proposal

Before producing the shortlist, propose 2–3 approaches with trade-offs. Lead with your recommendation and explain why.

Examples of meaningful axes: sequential-safe vs. maximally parallel, fine-grained tasks vs. coarse clusters, thin path vs. full review coverage.

Present approaches conversationally. Wait for the operator to choose.

## Phase 1 — Shortlist

### Goal

Produce a high-level decomposition (clusters and tasks with a DAG sketch). Stop at a checkpoint so the operator can confirm scope before deep work begins.

### Steps

1. Compose a shortlist in the format defined in `phase1-shortlist.md`, reflecting the chosen approach.
2. Run the DAG audit (`dag-audit.md`) on every declared dependency.
3. Present the shortlist.
4. **Checkpoint:** "Confirm shortlist? If yes, I move to deep investigation (file reads, full prompts). If you want to change X/Y/Z, I adjust first."
5. Wait for confirmation.

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
