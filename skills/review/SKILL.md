---
name: review
description: Dispatch reviewer subagents (spec compliance, then code quality) to validate a completed task. Invoked by execute-plans when the task is tagged +reviewer; may also be invoked directly for an ad-hoc review.
---

# fsa-tools:review

## Responsibility

Two-stage review of a completed task: spec compliance first, then code quality. Block forward progress on a failed review.

## When invoked

- Automatically by `fsa-tools:execute-plans` when a task is tagged `+reviewer`.
- Directly by the operator for ad-hoc review of a recent change.

## Inputs

- `original_prompt` — the full prompt the implementer subagent received.
- `implementer_summary` — what the implementer returned.
- `git_diff` — output of `git diff HEAD~1 HEAD`, or a per-file diff.
- `verification_command` — the shell command declared on the task.

## Flow

1. **Spec reviewer (always first).** Dispatch a fresh subagent with the prompt in `spec-reviewer-prompt.md`. Receive `APPROVED` or `REJECTED` with an issue list.
2. **Code quality reviewer (only if spec passed).** Dispatch a fresh subagent with the prompt in `code-quality-reviewer-prompt.md`. Receive `APPROVED` or `REJECTED` with an issue list.
3. If spec is rejected: return the issues to `execute-plans`. The execute skill decides — re-dispatch the implementer with feedback, or escalate to the operator.
4. If code quality is rejected: same flow.
5. Both approved: return `APPROVED`. The task is marked completed.

## Order matters

Spec compliance runs before code quality. A patch that does not do what was asked is not worth a style review.
