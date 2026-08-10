# Phase 1 — Shortlist Format

The shortlist is the artifact produced at the end of Phase 1 and presented at the operator checkpoint. It is intentionally light: cluster names, task names, models, dependency hints, and a one-line Definition of Done per task. Full prompts and per-task verification commands are deferred to Phase 2.

## Template

```
Cluster 1 — <Theme>
    Task 1.1: <Action> [<model>]
        - Description: <1 line>
        - DoD: <shell command that returns 0 when done>
    Task 1.2: <Action> [<model>] +reviewer
        - Description: <1 line>
        - Dependency: 1.1 (intra-cluster)
        - DoD: <shell command>

Cluster 2 [depends on Cluster 1] — <Theme>
    Task 2.1: <Action> [<model>]
        - Description: <1 line>
        - DoD: <shell command>

Estimate: N tasks, fits one sprint.
```

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

## Task grammar

`Task X.Y: <Name> [<model>] [+reviewer]`

- `[<model>]` is required: `haiku` | `sonnet` | `opus` | `fable`.
- `fable` is exceptional: it requires a one-line justification in the task body (like `blockedBy`) and at most one `fable` task per plan. If a plan "needs" several, the decomposition is wrong — split the task.
- `+reviewer` is optional. Add only when review is wanted; the default is no reviewer (thin path).
- `Dependency:` appears only when there is an intra-cluster dependency.
- `Cluster N [depends on Cluster M]` appears only when there is an inter-cluster dependency.
- No `effort` field. Model is the only tunable.
- No per-task worktree flag. Worktree is a plan-level header.

## Model selection

| Model | When to use |
|-------|-------------|
| `haiku` | Trivial tasks: write a config file, fixture, boilerplate, simple cleanup. |
| `sonnet` | Moderate tasks: clear feature implementation, localized refactor, documentation. |
| `opus` | Complex tasks: obscure root-cause debugging, interface design, business-critical logic. |
| `fable` | Exceptional only (~2x opus cost). A wrong design that is expensive to redo: cross-cutting architecture, critical financial logic, or debugging that already failed on opus. Requires justification; max one per plan. Default for the high tier remains `opus`. |

## Prompt checklist (Phase 2)

Every subagent prompt must satisfy:

- **Focused** — one problem, one file (or a small set), one clear action.
- **Self-contained** — all needed context inline. The subagent should not have to read other files just to understand the prompt.
- **Specific output** — the prompt declares what the subagent returns (summary, files touched, test state).
- **Constraints** — the prompt declares what NOT to modify.
- **Verification** — a shell command and the condition: "Return when `<command>` exits 0."
