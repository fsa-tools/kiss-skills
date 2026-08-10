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
6. If the worktree already exists, check its branch with `git -C ../.worktrees/<repo-name>-<plan-slug> branch --show-current`:
   - Branch is `fsa/<plan-slug>` → **reuse it**. Return its absolute path and branch name to the caller exactly as step 5 would. This is a resume, not a collision: worktree naming is deterministic, so a second session on the same plan is expected to land here.
   - Branch differs → abort with `Worktree already exists at <path> on branch <branch>. Options: 'git worktree remove <path>' to remove, or reuse explicitly.`

## Return

To `fsa-tools:execute-plans`: `(absolute worktree path, branch name)`, or a `none` signal if step 2 skipped setup.
On reuse, the return value is identical to a fresh creation — the caller does not distinguish the two cases.
