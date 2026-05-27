---
name: writing-plans
description: Write a parallel execution plan in two phases — quick shortlist with an operator checkpoint, then deep investigation with full subagent prompts. Plans are saved to docs/fsa-tools/plans/. Use before any non-trivial implementation, refactor, or multi-file cleanup.
---

# fsa-tools:writing-plans

## Announce

"Using `fsa-tools:writing-plans` in two-phase mode."

## Pre-flight

1. Verify `docs/fsa-tools/plans/` exists in the current project. Create it if missing.
2. Read context: `pwd`, `git log --oneline -5`, `git status --short`.

## Phase 1 — Shortlist

### Goal

Produce a high-level decomposition (clusters and tasks with a DAG sketch). Stop at a checkpoint so the operator can confirm scope before deep work begins.

### Steps

1. Ask 3–5 high-leverage questions about the task. Each one as multiple-choice with a recommended option. Examples of good questions: scope (which subprojects), preferred model for heavy tasks, worktree usage, review opt-in vs. thin path.
2. Wait for answers.
3. Compose a shortlist in the format defined in `phase1-shortlist.md`.
4. Run the DAG audit (`dag-audit.md`) on every declared dependency.
5. Present the shortlist.
6. **Checkpoint:** "Confirm shortlist? If yes, I move to deep investigation (file reads, full prompts). If you want to change X/Y/Z, I adjust first."
7. Wait for confirmation.

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
