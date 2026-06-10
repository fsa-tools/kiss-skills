# Verification and Retry Protocol

What to do after each subagent returns.

## Verification

For every returned task:

1. Run the task's verification command: `<task.verification>`.
2. If exit 0: verification passed → go to step 3.
3. If the task is tagged `+reviewer`: invoke `fsa-tools:review` (pass the original prompt, the implementer summary, the diff, the verification command). Wait for the verdict.
4. If review is `APPROVED` (or the task has no `+reviewer`): `TaskUpdate(task.id, status=completed)`.
5. If verification failed (exit ≠ 0) or review is `REJECTED`: follow the retry protocol below.

## Retry protocol

### Verification failure (exit ≠ 0)

1. Collect: the verification command's output, `git diff HEAD~1 HEAD`, the implementer's summary.
2. Decide by failure type:
   - **Simple failure** (missing file, typo, obvious assertion miss): retry with the same model and additional context.
     Retry prompt: original + `FEEDBACK: The command <cmd> returned:\n<output>\nFix exactly this error.`
   - **Complex failure** (logic bug, wrong design): upgrade the model (haiku → sonnet, sonnet → opus, opus → fable). The opus → fable step is the last rung before operator escalation — only pay the ~2x cost after opus has demonstrably failed.
   - **After 2 retries**: escalate to the operator with the task id, original prompt, errors, and a history of what was tried.

### Review rejection

1. Collect the reviewer's issue list.
2. Re-dispatch the implementer with the original prompt plus a section `REVIEWER ISSUES:\n<list>`.
3. If it fails again after one retry: escalate to the operator.

## Escalation

Structured format:

```
Task <id> — <name> — failed after N attempts

Original prompt:
<prompt>

Errors observed:
<errors>

What was tried:
<retry history>

Action needed: [fix manually | adjust the prompt | re-dispatch manually]
```
