---
name: finish-branch
description: Post-execution branch handling. Show branch state, offer integration options (PR, merge, leave, abandon), then execute the chosen option. Invoked by execute-plans on DoD success, or directly by the operator.
---

# fsa-tools:finish-branch

## Responsibility

After plan execution completes, show the operator the branch state and let them decide how to integrate the work.

## Flow

1. Receive the branch name (or detect with `git branch --show-current`) and the worktree path (if any).
2. Show state:
   - `git log main..HEAD --oneline` — commits ahead.
   - `git diff main...HEAD --stat` — files touched.
   - Tests: run the plan's Definition of Done command if known.
3. Present numbered options:
   1. Push and open a PR via `gh pr create` (default if a remote is configured).
   2. Merge directly into main: `git checkout main && git merge --ff-only <branch> && git push`.
   3. Leave as-is (operator will integrate later).
   4. Abandon: `git checkout main && git branch -D <branch> && git worktree remove <path>`.
4. Execute the chosen option.
5. If a worktree was used, run `git worktree remove <path>` at the end — except when the operator chose option 3 (leave as-is).
6. Close-out handoff (every option except abandon): integration is not release close-out. End by stating that the round is still open — versioning, changelog, tracking, memory, deploy — and defer to whatever close-out flow the operator's setup defines. Never suggest invoking a deploy skill directly from here.

## Invoke

Automatically by `fsa-tools:execute-plans` when the global Definition of Done is met. Or directly: `/fsa-tools:finish-branch`.
