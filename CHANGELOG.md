# Changelog

## [Unreleased]

## [0.4.1] — 2026-08-11

### Fixed

- The sprint boundary stop is scoped to the dispatch loop instead of being stated as a condition on the task list. Step 6 already claimed its resume marking does not trigger the boundary, but step 9 did not say so: read literally, a resumed session that marks a whole finished sprint completed would stop at a boundary it had crossed in an earlier session, and end without dispatching anything.
- The Definition of Done gate reads as a state condition — every task of the final sprint complete, whether the dispatch loop closed it or the resume marking did. It previously required the dispatch loop to report all tasks complete, so a resumed session that closed the final sprint entirely through the resume marking never reached the gate and the plan had no termination.
- The consequence of missing sprint edges is stated precisely. The boundary stop does fire without them — once the last task of sprint N closes — but by then the sprint N+1 work has already landed, so the effect is a boundary enforced too late rather than one that never fires.
- `execute-plans` step 6 says "gains no sprint edges" for a plan with one implicit sprint, matching the wording already used in the sibling docs.

## [0.4.0] — 2026-08-11

### Added

- Cross-session execution. A plan can be sliced into sprints (`- **Sprints:**` in the metadata plus `## Sprint N` sections); `execute-plans` stops at the end of every non-final sprint and the operator opens a fresh session to continue. A plan with no sprint slice yields a single implicit sprint, so nothing changes for it.
- Resume check. On resume the skill re-observes the repository instead of trusting a record: a task is done exactly when its `verification:` command exits 0. The continuity file (`docs/fsa-tools/continuity/<plan-slug>.md`) persists only what the repository cannot re-derive — decisions taken, gotchas discovered, run metrics — never git state, exit codes, or a completion list.

### Fixed

- Sprint boundaries are now binding in the mechanism, not only in the prose. `TaskCreate` builds `addBlockedBy` from a third input — every task of the preceding sprint — so a sprint N+1 cluster declaring no dependency reaching back into sprint N is no longer available in the first wave. Enforcement lives at `TaskCreate`, which leaves the dispatch loop and the parser unchanged.
- The resume check's "dispatch only the tasks whose verification did not exit 0" gained a consumer. Every task is created first (the edges need ids to reference), then the ones already observed passing are marked completed before the first poll. Without it a resumed session re-dispatched work whose verification already exits 0. This marking happens outside the dispatch loop, so it does not trigger the sprint boundary stop.

## [0.3.2] — 2026-07-07

### Fixed

- `execute-plans` no longer instructs the agent to activate `/goal` — a native UI command that only the operator can run and that the agent cannot invoke via the Skill tool. Termination is now agent-owned: once all tasks complete, the skill runs the plan's Definition of Done command via Bash and gates on its exit code (exit 0 → global review; exit ≠ 0 → escalate). Design docs (`principles.md`, `design-rationale.md`, `execution-flow.md`) and the dispatcher loop updated to match.

## [0.3.1] — 2026-06-11

### Added

- `finish-branch` gains a close-out handoff step: after executing any integration option except abandon, the skill states that the round is still open (versioning, changelog, tracking, memory, deploy) and defers to the operator's own close-out flow. Deploy skills must never be suggested directly. (#1)

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
