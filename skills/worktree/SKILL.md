---
name: worktree
description: Create a dedicated git worktree for executing an fsa-tools plan. The naming is derived from the plan filename. Invoked by execute-plans during setup; may also be invoked directly by the operator.
---

# fsa-tools:worktree

## Responsibility

Create an isolated worktree for plan execution so the operator's primary working tree stays unblocked.

## Flow

1. Receive the plan slug and the worktree header value (`recommended` | `required` | `none` | absent).
2. Branch on the value:
   - `required` — create the worktree without asking.
   - `recommended` or absent — ask the operator: "Create worktree for this execution? [Y/n]" (default Y).
   - `none` — skip setup, use the current workspace.
3. Worktree path: `../.worktrees/<repo-name>-<plan-slug>`, where `plan-slug` is the plan filename with the date prefix and the `.md` suffix removed.
4. Branch name: `fsa/<plan-slug>`.
5. Execute: `git worktree add ../.worktrees/<repo-name>-<plan-slug> -b fsa/<plan-slug>`. Return `(absolute path, branch name)` to the caller.
6. If the worktree already exists: abort with `Worktree already exists at <path>. Options: 'git worktree remove <path>' to remove, or reuse explicitly.`

## Return

To `fsa-tools:execute-plans`: `(absolute worktree path, branch name)`, or a `none` signal if step 2 skipped setup.
