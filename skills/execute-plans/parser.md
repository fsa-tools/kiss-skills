# Parser Protocol

How `fsa-tools:execute-plans` extracts structured data from the markdown plan.

## Fields to extract

| Field | Location in the plan | Type |
|-------|----------------------|------|
| `dod_global` | The shell block under `## Definition of Done (global)` | string (shell command) |
| `policy` | Bullet list under `## Policy (invariant)` | `string[]` |
| `worktree_header` | Line `- **Worktree:**` in the Metadata section | `"recommended"` \| `"required"` \| `"none"` \| `null` |
| `sprints` | Line `- **Sprints:**` in the Metadata section, plus `## Sprint N` sections | `Sprint[]` |
| `clusters` | Sections `### Cluster N` | `Cluster[]` |

## Sprint shape

```
Sprint {
  id: number          // N in "## Sprint N"
  name: string        // text after "— "; empty for the implicit sprint
  clusters: number[]  // ids of the clusters under this sprint heading
}
```

## Cluster shape

```
Cluster {
  id: number          // N in "### Cluster N"
  name: string        // text after "— "
  blockedBy: number[] // clusters declared in "Inter-cluster dependency:"
  tasks: Task[]
}
```

## Task shape

```
Task {
  id: string           // "N.M" — cluster.task
  name: string         // text in "#### Task N.M: <Name>"
  model: string        // text between brackets: haiku | sonnet | opus | fable
  reviewer: boolean    // true if "+reviewer" is present in the title
  blockedBy: string[]  // task ids declared in "Intra-cluster dependency:"
  prompt: string       // contents of the code block under "Prompt for subagent"
  verification: string // shell command from "Verification:"
}
```

## Parsing rules

1. **Intra-cluster dependencies** live in the task body under `Intra-cluster dependency:`. Absent → no intra-cluster dependency.
2. **Inter-cluster dependencies** live in the cluster header under `Inter-cluster dependency:`. The value `none` means no inter-cluster dependency.
3. **Task id** for `blockedBy`: parse `Task N.M` → id `N.M`.
4. **Worktree header**: if absent, default to `recommended`.
5. **Verification** appears immediately before `Prompt for subagent:` in the task body.
6. **Sprint membership.** A cluster belongs to the nearest preceding `## Sprint N` heading.
7. **Absent sprint slice.** No `- **Sprints:**` header and no `## Sprint N` headings → yield a single implicit sprint with id 1 containing every cluster in order. The dispatch loop then needs no special case: every plan has at least one sprint, and a plan with one sprint has no non-final sprint, so no boundary stop ever fires.
8. **Disagreement between the header and the sections.** If `- **Sprints:** N` does not match the number of `## Sprint N` headings, escalate to the operator. Do not guess which one is right.
