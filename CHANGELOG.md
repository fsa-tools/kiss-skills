# Changelog

## [Unreleased]

## [0.1.0] — 2026-05-26

### Added

- `fsa-tools:writing-plans` — two-phase plan authoring (shortlist with operator checkpoint, then full plan with subagent prompts and verification commands). Plans are saved to `docs/fsa-tools/plans/`.
- `fsa-tools:execute-plans` — parallel dispatch of subagents per task, DAG-respecting, terminated by `/goal` on the global Definition of Done.
- `fsa-tools:worktree` — dedicated git worktree per plan execution.
- `fsa-tools:review` — opt-in two-stage review (spec compliance, then code quality) triggered by `+reviewer` on a task.
- `fsa-tools:finish-branch` — post-execution branch handling: state report, PR/merge/leave/abandon options, worktree cleanup.
