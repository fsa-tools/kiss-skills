# DAG Audit — Rules and Anti-patterns

Run during Phase 1, before the operator checkpoint. The purpose is to keep dependencies honest: every `blockedBy` should reflect a real artifact handoff, not a sequential habit.

## Rules

1. **Default is independent.** No `blockedBy` is declared without an explicit reason.
2. **Audit every declared dependency** before the Phase 1 checkpoint. Each one must answer: "what does the blocker produce that the blocked task consumes?"
3. **Decompose aggressively.** A broad task with parallelizable parts becomes N tasks.
4. **Stub-and-integrate over sequential.** When A blocks B because of an interface, propose: A produces the contract (a type or schema) as a tiny task; B works against the stub in parallel; integration becomes the final task.
5. **File overlap forces sequential split.** Two tasks touching the same file must be sequential.
6. **Cluster granularity.** A 1-task cluster is fine. A 6+-task cluster is suspect — split it.
7. **Phase 0 fan-out is the plan-quality metric.** More parallel tasks in the initial batch is better, all else equal.

## Anti-patterns (reject on sight)

- **Too-broad.** "Fix all the tests" → split into N tasks by file or domain.
- **No-context.** Prompt missing the error message, test name, or file path → add the excerpt.
- **No-constraints.** No "do not touch X" clause → add an explicit policy.
- **Vague-output.** "Fix it" → specify what the subagent returns.
- **Artificial dependency.** "Task B depends on Task A because I would run A first" → remove the dependency.
- **Monolithic cluster.** 6+ tasks with no thematic coherence → split into smaller clusters.

## Audit questions (use at the checkpoint)

For each declared dependency, ask:

- "Does Task B actually need an artifact Task A produces, or is this just sequential habit?"
- "If I ran Task B now without Task A, what would specifically break?"
- "Can A produce a stub or interface that lets B run in parallel?"
