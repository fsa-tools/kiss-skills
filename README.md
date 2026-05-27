# fsa-tools

A skill suite for parallel execution planning in Claude Code. Decompose work into a DAG of independent clusters, dispatch subagents in parallel, and stop when a verifiable Definition of Done passes.

## Philosophy

Parallelism is the default. Sequential work is the exception, declared only when one task literally cannot start without another's output. Autonomy is the second principle: a plan is structured well enough for subagents to execute without back-and-forth with the operator.

See [design principles](docs/design/principles.md) and [design rationale](docs/design/design-rationale.md).

## Install

```
claude plugin install github.com/fsa-tools/kiss-skills
```

## Quickstart

```
# 1. Generate a plan (current session, project context loaded)
/fsa-tools:writing-plans

# 2. Review the generated file at docs/fsa-tools/plans/<plan>.md

# 3. Execute (fresh Opus session, clean context)
/fsa-tools:execute-plans
```

## Skills

| Skill | Type | Purpose |
|-------|------|---------|
| `fsa-tools:writing-plans` | core | Two-phase plan authoring (clusters + DAG) |
| `fsa-tools:execute-plans` | core | Parallel subagent dispatch, `/goal` termination |
| `fsa-tools:worktree` | helper | Dedicated git worktree per plan |
| `fsa-tools:review` | helper | Spec-compliance and code-quality review (opt-in via `+reviewer`) |
| `fsa-tools:finish-branch` | helper | Post-execution branch handling and PR creation |

## Documentation

- [Plan schema](docs/usage/plan-schema.md)
- [Execution flow](docs/usage/execution-flow.md)
- [Example: parallel test fixes](docs/usage/examples/parallel-test-fixes.md)
- [Design principles](docs/design/principles.md)
- [Design rationale](docs/design/design-rationale.md)

## Status

v0.1 alpha. Claude Code only. The plan schema may have breaking changes before v1.0. Version source of truth: `plugin.json`.

## License

MIT.
