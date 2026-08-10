# Dispatch Protocol

How the execute skill dispatches available tasks in parallel.

## Principles

- **Eager dispatch.** On every `TaskList` poll, dispatch every available task in a single message with N parallel `Agent` calls.
- **Immediate re-poll.** A completed cluster may unblock others. Re-poll on every `TaskUpdate(completed)`.
- **No artificial ordering.** If Phase 0 has 4 available tasks, all four ship together. No alphabetical, no by-model, no "the small one first" heuristic.

## Loop

```
LOOP:
  available = TaskList(status=pending, no_owner, no_open_blockedBy)

  if available.empty:
    if all_tasks_completed:
      // All tasks done → run the terminal DoD gate (SKILL.md step 11)
      break
    else:
      // tasks are blocked waiting on predecessors
      wait for next TaskUpdate
      continue

  // Eager dispatch — single message, N parallel Agent calls
  SEND ONE MESSAGE WITH:
    for task in available:
      Agent(model=task.model, prompt=task.composed_prompt)

  // Wait for every agent in the batch to return
  for result in results:
    handle via verification-loop.md
    TaskUpdate(task.id, completed)  // or escalate per the verification protocol
    // loop returns to start (re-poll)
```

## Model selection

Use the model declared on the task. If somehow missing (a malformed plan), fall back to `sonnet`.

## Composed prompt

The prompt sent to `Agent` is the plan's `task.prompt` field, prefixed with:

```
[Execution context]
Project: <absolute project path>
Branch: <current branch if a worktree is configured>
Invariant policy:
<lines from the plan's Policy section>

[Task]
<original task.prompt content>
```
