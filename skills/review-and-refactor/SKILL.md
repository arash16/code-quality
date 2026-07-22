---
name: review-and-refactor
description: Use when asked to review, audit, or assess an existing codebase or module and produce an ordered plan to pay down technical debt or refactor safely. Scans for a catalog of issue types (dead code, duplicated knowledge, god classes, missing IoC, low cohesion, shallow modules, untested change points, fragile integrations), prioritizes by dependency structure and churn, and sequences the work safely — dead-code removal and a test safety net before any behavior-preserving change, never a big-bang rewrite.
---

# Review and Refactor

Produce an **ordered, prioritized** plan and execute it **safely**. Never rewrite blindly. Present findings and a sequenced plan first; change code only once a safety net exists — the sole exception is dead-code removal, which is proven safe by definition.

## Ecosystem precedence

Judge code against **its own stack's idioms and this codebase's established conventions**, not a foreign standard. A pattern that is idiomatic for the framework is not debt. Apply the *intent* of each principle in `clean-dev`, scoped to the language.

## Operating rules (non-negotiable)

- **Safety before change:** no unit is modified (beyond dead-code removal) until the safety net covers it.
- **Structural and behavioral changes never share a commit.**
- **On a test going red mid-refactor: revert and retry smaller — do not debug in place.**
- **Bugs found while characterizing are pinned as-is and ticketed, never silently fixed** — callers may depend on the quirk.
- **Never big-bang rewrite** a working system; migrate incrementally behind a boundary.
- Produce the plan and get the user's agreement on scope + ordering before executing large moves.

---

## Phase 0 — Assess: build the issue inventory

Scan the target for the issue types below. **Each is independently scannable — for a large codebase, fan out one pass (or one sub-agent) per issue type, then merge into a single inventory.** For each finding record: location, issue type, evidence, blast radius (who depends on it), rough effort.

**Issue catalog** (each maps to a principle in `clean-dev`):

1. **Dead code** — unreferenced functions/classes/exports, unreachable branches, unused params/imports, permanently-off flag paths.
2. **Duplicated knowledge** — the same business rule/formula/constant in several places (true DRY violations, *not* coincidental look-alikes).
3. **God classes / SRP violations** — many reasons to change in one unit; mixed abstraction levels (rules beside SQL/HTTP).
4. **Missing IoC / leaked dependencies** — business logic importing framework/ORM/vendor/IO directly; unsubstitutable seams; globals/singletons reached from inside logic.
5. **Feature envy / shotgun surgery / divergent change** — one change touching many files; a method using another object's data more than its own.
6. **Low cohesion** — one feature scattered across role-folders; related code far apart.
7. **Shallow modules / classitis / pass-through** — tiny wrappers, interfaces as complex as their impl, orchestration pushed onto callers.
8. **Safety-net gaps** — change-prone code with no tests.
9. **Long functions / deep nesting / high complexity.**
10. **Primitive obsession** — bare strings/ints/maps where a domain type belongs; stringly-typed states.
11. **Speculative generality** — abstractions with one implementation, unused config/params, plugin points with one plugin.
12. **Fragile integration points** — outbound calls without timeouts/limits, unbounded queries, no retry/breaker → hand to the `resilience` skill.
13. **Inconsistent patterns** — several ways to do the same thing (error handling, data access) across the codebase.

## Phase 0 — Prioritize (rank, don't dump)

- **Map dependency structure.** Mark each unit **root** (high fan-in — many dependents) vs **leaf** (fan-out only) by counting inbound imports/usages. Root nodes with issues rank highest — instability there radiates system-wide.
- **Three-axis targeting** for where to start: (a) code you're about to change anyway, (b) highest git-churn (`git log` hotspots), (c) core domain. A unit scoring on all three is the first target.
- **Sequence prerequisites leaves→root (Mikado):** attempt the desired change; when it forces a prerequisite, note it, revert, do the prerequisite first. Build the prerequisite graph; solve from the leaves (no-dependency tasks) inward toward the root.
- **Output:** a prioritized, ordered list — each item with issue type, location, blast radius, effort, and its place in the sequence. **No forced UPPERCASE docs** — present the plan in your response (or the format the user asks for).

---

## The ordered method (execute in this order)

State which step you're in. Steps 3+ touch a unit only where the step-2 safety net covers it.

1. **Dead code elimination — first.** Remove code proven unreferenced. It shrinks the surface everything else must handle. Be conservative with public API and late-bound callers (reflection, DI containers, string dispatch, framework hooks, serialized names) — verify no external/dynamic caller before deleting; when unsure, deprecate + ticket instead of delete.
2. **Safety net — GATE.** For each change point: find test points where effects *surface* (prefer a **pinch point** — a narrow spot whose few tests cover wide upstream behavior); break dependencies at the cheapest seam (table below); write **characterization tests** — assert a value you know is wrong, read the failure, pin the observed value; cover only the branches the change will touch.
3. **Behavior-preserving refactors.** Named transformations, one at a time; tests green before and after each; commit each; on red, revert. Structure only — no behavior change.
4. **Legibility** — names, small clarifications, error context — only where you're already touching (see `clean-dev`).
5. **Deep-module consolidation** — merge shallow classes that travel together and share state; delete pass-through methods; cure classitis. Bounded by SRP — don't over-merge.
6. **Dependency boundaries** — introduce seams so business logic stops importing the framework/ORM/vendor; start with the most-changed module (see `clean-dev` DIP).
7. **Debt-prevention habits** — replace untracked TODOs with ticketed ones; converge inconsistent patterns onto one; set a debt budget so it doesn't re-accumulate.
8. **Harden integration points** — timeouts, bounded queries, retries/breakers. Defer to the `resilience` skill for specifics.
9. **Bounded contexts / large-scale structure** — carve modules along domain boundaries via incremental migration. Never big-bang.

---

## Decision table — break a dependency at the cheapest seam

| Blocker | Technique |
|---|---|
| Constructor does real work / creates its collaborators | Parameterize Constructor (with a production default) |
| Heavy concrete collaborator | Extract Interface / accept a narrower protocol — safest, only loosens a type |
| Clock / random / global read inside the body | Parameterize Method / inject the value |
| Buried `new` of a dependency | Extract-and-Override Factory Method |
| Static / singleton call | Introduce Instance Delegator / injectable accessor |
| Framework type you can't construct | Adapt Parameter — narrow to an interface you own |
| Monster method with many locals | Break Out Method Object |

Prefer, in order: an existing seam → module/import mocking (`jest.mock`, `unittest.mock.patch`) → create a constructor seam → heavier surgery as a last resort. Seams that improve the design (Extract Interface, Parameterize Constructor) are **permanent**; seams that only enable a test (static setters, module patches) are **scaffolding** — ticket their removal.

## Decision table — large-scale moves

| Move | Use when | Shape |
|---|---|---|
| Branch by Abstraction | replacing a widely-used component | wrap in an abstraction → migrate callers → build new behind it → switch wiring → delete old |
| Parallel Change (expand–migrate–contract) | changing a signature or data shape | add new beside old → migrate callers → remove old |
| Strangler Fig | replacing a subsystem/service | intercept at a routing layer → build new behind it → redirect incrementally → retire old |
| Feature toggle | risky rollout | ship dark → enable incrementally → instant rollback |

---

## Definition of Done

| Check | If No → Action |
|---|---|
| Prioritized plan agreed with the user before large moves? | Present it and confirm scope |
| Dead code removed first, and only if provably unreferenced? | Verify references / deprecate instead |
| Every modified unit covered by the safety net? | Add characterization tests first |
| Did any commit mix structure and behavior? | Split them |
| Discovered bugs pinned + ticketed, not silently fixed? | Restore the quirk; file a ticket |
| Tests green across each refactor step (revert on red)? | Revert and retry smaller |
| Any big-bang rewrite of working code? | Convert to an incremental migration |
| Findings ranked by blast radius (root vs leaf) and churn? | Re-rank before executing |
