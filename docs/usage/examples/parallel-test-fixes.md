# Example — Parallel Test Fixes in a Monorepo

A worked example of an `fsa-tools` plan. The scenario is generic: a TypeScript monorepo has 26 failing tests in one of its packages. The fixes group into thematic clusters that can ship in parallel.

This file shows what a real plan looks like end-to-end. Use it as a structural reference when writing your own.

---

# Plan — Fix 26 failing tests in packages/api-service

## Metadata

- **Generated:** 2026-05-26
- **Worktree:** recommended

## Context

Monorepo at `/repo`. Subproject `packages/api-service` (TypeScript + Vitest). The failing tests cluster by failure mode — fixture mismatches, type mismatches, and an integration boundary issue. No shared root cause; each cluster can be fixed independently.

## Baseline (current state)

```bash
cd packages/api-service && npx vitest run 2>&1 | tail -5
```

Output excerpt: `Tests: 26 failed | 47 passed (73 total)`.

## Objective

Bring `packages/api-service` test suite to 73 passing, zero failing.

## Definition of Done (global)

```bash
cd packages/api-service && npx vitest run --reporter=json 2>/dev/null | jq -e '.numFailedTests == 0 and .numTotalTests == 73'
```

**Expected output:** exit 0.

## Policy (invariant)

- Do not modify business logic in `packages/api-service/src/`. Only fix test scaffolding, types, and integration glue.
- Do not add new implementation modules. Only edit existing files or add test fixtures.
- Do not bump dependency versions in `package.json`.

## Dependency justification

- **Cluster 3 blockedBy Cluster 2.** Cluster 3 (integration fixes) edits `packages/api-service/tests/integration/checkout.test.ts`, which imports the `PositionState` type and the `Registry` shape that Cluster 2 normalizes. Without Cluster 2's normalization, Cluster 3 would re-introduce the type mismatch.

## Clusters

### Cluster 1 — Fixture configuration

**Inter-cluster dependency:** none

#### Task 1.1: Add missing fields to pool fixture [haiku]

**Files:**
- Modify: `packages/api-service/tests/fixtures/pool.ts`

**Diagnosis:** A recent schema change added `feeTier` and `tickSpacing` to the `Pool` type. The fixtures in this file were not updated. 8 tests fail with "Property 'feeTier' is missing".

**Verification:** `cd packages/api-service && npx vitest run tests/fixtures/pool 2>&1 | grep -q "0 failed"`

**Prompt for subagent (Agent tool):**
```
Project: /repo
File to edit: packages/api-service/tests/fixtures/pool.ts

The Pool type now requires two new fields: feeTier (number, default 3000) and tickSpacing (number, default 60). Every fixture in this file needs both fields added. Do not change the existing field values; only append the two new fields.

Constraints:
- Edit only packages/api-service/tests/fixtures/pool.ts.
- Do not modify the Pool type or any test files.

Verification: cd packages/api-service && npx vitest run tests/fixtures/pool 2>&1 | grep -q "0 failed"
Return when: the verification command exits 0.
```

#### Task 1.2: Add missing fields to position fixture [haiku]

**Files:**
- Modify: `packages/api-service/tests/fixtures/position.ts`

**Diagnosis:** The same schema change touched the `Position` type — added `openedAt` (number, default `Date.now()`). 4 tests fail.

**Verification:** `cd packages/api-service && npx vitest run tests/fixtures/position 2>&1 | grep -q "0 failed"`

**Prompt for subagent (Agent tool):**
```
Project: /repo
File to edit: packages/api-service/tests/fixtures/position.ts

The Position type now requires openedAt (number, default Date.now()). Add it to every fixture in this file.

Constraints:
- Edit only packages/api-service/tests/fixtures/position.ts.
- Do not change existing field values.

Verification: cd packages/api-service && npx vitest run tests/fixtures/position 2>&1 | grep -q "0 failed"
Return when: the verification command exits 0.
```

### Cluster 2 — Type-mismatch normalization

**Inter-cluster dependency:** none

#### Task 2.1: Normalize PositionState representation [sonnet] +reviewer

**Files:**
- Modify: `packages/api-service/src/types/position.ts`
- Modify: `packages/api-service/tests/unit/state.test.ts`

**Diagnosis:** `PositionState` is declared as a string union in one place and as a TypeScript `enum` in another. Tests assert against `PositionState.OPEN` but receive the string literal `"open"`. Normalize to the string union — it is the public API surface.

**Verification:** `cd packages/api-service && npx vitest run tests/unit/state 2>&1 | grep -q "0 failed"`

**Prompt for subagent (Agent tool):**
```
Project: /repo
Files:
- packages/api-service/src/types/position.ts — replace `enum PositionState { OPEN, CLOSED, PENDING }` with the string union `type PositionState = "open" | "closed" | "pending"`. Update internal callers within src/types/ that referenced the enum members.
- packages/api-service/tests/unit/state.test.ts — replace `PositionState.OPEN` with the literal `"open"`. Apply equivalent rewrites for CLOSED and PENDING.

Constraints:
- Do not change the test assertions, only how the enum is referenced.
- Do not touch src/ files outside types/position.ts.

Verification: cd packages/api-service && npx vitest run tests/unit/state 2>&1 | grep -q "0 failed"
Return when: the verification command exits 0.
```

#### Task 2.2: Correct registry value type [sonnet]

**Files:**
- Modify: `packages/api-service/src/registry.ts`
- Modify: `packages/api-service/tests/unit/registry.test.ts`

**Diagnosis:** Registry is typed as `Map<string, string>` but values are inserted as numbers. Tests assert `typeof value === "number"`.

**Verification:** `cd packages/api-service && npx vitest run tests/unit/registry 2>&1 | grep -q "0 failed"`

**Prompt for subagent (Agent tool):**
```
Project: /repo
Files:
- packages/api-service/src/registry.ts — change the registry type from `Map<string, string>` to `Map<string, number>`. Adjust insertion sites accordingly.
- packages/api-service/tests/unit/registry.test.ts — no changes expected; the test should pass once the type is correct.

Constraints:
- Do not introduce a new helper module.
- Do not change the registry's public API shape.

Verification: cd packages/api-service && npx vitest run tests/unit/registry 2>&1 | grep -q "0 failed"
Return when: the verification command exits 0.
```

### Cluster 3 — Integration fix

**Inter-cluster dependency:** depends on Cluster 2

#### Task 3.1: Rewire checkout integration test against normalized types [opus]

**Files:**
- Modify: `packages/api-service/tests/integration/checkout.test.ts`

**Diagnosis:** The checkout integration test imports `PositionState` (now a string union after Cluster 2) and reads from the registry (now `Map<string, number>`). Several setup helpers still mock the old enum and the wrong value type. Needs careful rewiring without changing the assertion intent.

**Verification:** `cd packages/api-service && npx vitest run tests/integration/checkout 2>&1 | grep -q "0 failed"`

**Prompt for subagent (Agent tool):**
```
Project: /repo
File to edit: packages/api-service/tests/integration/checkout.test.ts

Cluster 2 normalized PositionState to a string union and the registry value type to number. This integration test uses the old enum and treats registry values as strings. Audit every reference to these two types and rewire:
- Replace PositionState.OPEN/.CLOSED/.PENDING with the literals.
- Treat registry.get(k) as a number; remove any string coercion.

Do NOT change the assertion intent. Only adjust how inputs are constructed and how registry values are read.

Verification: cd packages/api-service && npx vitest run tests/integration/checkout 2>&1 | grep -q "0 failed"
Return when: the verification command exits 0.
```

## Launch order (DAG resolved)

### Phase 0 — parallel

- Cluster 1 / Task 1.1 [haiku]
- Cluster 1 / Task 1.2 [haiku]
- Cluster 2 / Task 2.1 [sonnet] +reviewer
- Cluster 2 / Task 2.2 [sonnet]

**Fan-out Phase 0: 4 parallel tasks**

### Phase 1 — after Cluster 2 completes

- Cluster 3 / Task 3.1 [opus]

**Fan-out Phase 1: 1 task**
