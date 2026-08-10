# Continuity Protocol

The continuity file is the only state `fsa-tools:execute-plans` persists across sessions. It holds what the repository cannot re-derive, and nothing else.

## Location

<target-project>/docs/fsa-tools/continuity/<plan-slug>.md

`plan-slug` is derived exactly as in `skills/worktree/SKILL.md`: the plan filename with the date prefix and the `.md` suffix removed. One file per plan.

An absent file means a first run. The absence is not an error.

## Evidence boundary

**Re-derivable — never persisted, always re-observed on resume.** Branch, `HEAD`, dirty paths, per-task `verification:` exit codes, the global Definition of Done exit code, and task completion itself. The plan already carries every command needed to observe these; a recorded copy can only go stale.

Task completion deserves emphasis because it is the field most designs record. It is re-derivable by construction: a task is done exactly when its `verification:` command exits 0. A separate completion list would be a second source of truth that can disagree with the first — the failure mode this boundary exists to prevent.

**Non-re-derivable — persisted.** Decisions taken during execution, gotchas discovered, and observed run metrics. Nothing in the repository holds these, and they are exactly what a fresh session pays to rediscover.

## Format

# Continuity — <plan-slug>

## Decisions
- <decision taken during execution>
  Rationale: <why>
  Task: <N.M>

## Gotchas
- <thing that cost time and is not visible in the diff>
  Task: <N.M>

## Sprint progress (hint — not authoritative)
- Sprint 1: reported complete YYYY-MM-DD
- Sprint 2: not started

## Observed runs
- YYYY-MM-DD — tasks=N waves=N diff_lines=N duration=Nm

## Rules

1. `Decisions`, `Gotchas`, and `Observed runs` are append-only. A resumed session adds; it never rewrites history.
2. `Sprint progress` is a hint that orders the resume scan, never a source of truth. Authority on whether a task is done is its `verification:` exit code, re-run at resume time. If the hint and the exit codes disagree, the exit codes win and the hint is rewritten to match.
3. Never record git state, exit codes, file hashes, or a per-task completion list.
4. Write the file at the end of every run — sprint boundary stop, Definition of Done success, or escalation. On escalation this is what makes the failure resumable instead of lost.
5. The file lives in the target project's working tree and is versioned in git. There is no state store outside the working directory.
