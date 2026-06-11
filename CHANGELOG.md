# Changelog

## [Unreleased]

## [0.3.1] — 2026-06-11

### Added

- `finish-branch` gains a close-out handoff step: after executing any integration option except abandon, the skill states that the round is still open (versioning, changelog, tracking, memory, deploy) and defers to the operator's close-out orchestrator (e.g. `/done`). Deploy skills must never be suggested directly. (#1)

## [0.3.0] — 2026-06-09

### Added

- `fable` as an exceptional fourth model tier in the task grammar (`haiku | sonnet | opus | fable`). Gated: one-line justification required, max one `fable` task per plan, `opus` remains the default for the high tier.
- Escalation ladder in `execute-plans` verification loop gains `opus → fable` as the last rung before operator escalation.

## [0.2.0] — 2026-06-09

_Backfilled entry — released without a changelog entry._

### Changed

- Strengthened `writing-plans` brainstorming and added a global review step to `execute-plans`.
- Task list entries now show the model name.
- Plugin manifest moved to `.claude-plugin/plugin.json`; root `plugin.json` removed.
- Marketplace source fixed to clone the full repo.

## [0.1.0] — 2026-05-26

### Added

- `fsa-tools:writing-plans` — two-phase plan authoring (shortlist with operator checkpoint, then full plan with subagent prompts and verification commands). Plans are saved to `docs/fsa-tools/plans/`.
- `fsa-tools:execute-plans` — parallel dispatch of subagents per task, DAG-respecting, terminated by `/goal` on the global Definition of Done.
- `fsa-tools:worktree` — dedicated git worktree per plan execution.
- `fsa-tools:review` — opt-in two-stage review (spec compliance, then code quality) triggered by `+reviewer` on a task.
- `fsa-tools:finish-branch` — post-execution branch handling: state report, PR/merge/leave/abandon options, worktree cleanup.
