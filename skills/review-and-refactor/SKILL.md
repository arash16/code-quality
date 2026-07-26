---
name: review-and-refactor
description: Use when asked to review, audit, or assess existing code and produce an ordered, safe plan to pay down technical debt or refactor. The counterpart to clean-dev.
---

# Review and Refactor

**Bugs found while characterizing are listed** as issues, planned to be fixed along the way (Red → Fix → Green).

## Phase 1 — Recon (one cheap pass, no full-code review, no fan-out)

Delegate *little* by first learning the shape of the target. In a single pass gather:
- **Size** — files / LOC; does it fit one context?
- **Organization** — by-feature, by-layer, or flat? (decides the partition axis below)
- **Roots vs leaves** — which units carry high fan-in (many inbound imports/usages)? Instability there radiates system-wide.
- **Churn** — `git log` hotspots.
- **Core domain** — where the business value concentrates.
- **Scope** — code the user is about to change anyway.

## Phase 2 — Partition, then delegate (size the fan-out to the recon)

Intelligently pick some partition axis from recon and give each worker the catalog over a bounded slice:

- **Fits one context →** no sub-agents; scan it yourself in one pass.
- **By-feature tree →** one worker per feature/domain folder, each hunting the full catalog in its slice.
- **By-layer / flat / cross-cutting →** one worker per module cluster, or batch the catalog into **issue families** (below) and give one family per worker. A handful covers everything.
- **Focus first on the intersection of root × churn × core domain** — don't audit the whole tree when one slice carries the risk.
- **Cap concurrent workers (≈3–5), and total workers (≈6–10):** The total number of workers fanned out should be strictly sane relative to the LOC and complexity of the target. For example, a 10k LOC target deserves no more than 3-5 workers.

### The catalog — `clean-dev` violations, grouped so a worker can take one family

| Family | Flag violations of (`clean-dev` anchor · compressed smell) |
|---|---|
| **Duplication & placement** | DRY (same rule/formula/constant in N places); Minimal blast radius (one foreseeable change → many files) |
| **Responsibility** | SRP (god class; rule beside SQL/HTTP); Separation of concerns / orthogonality; Cohesion by feature (one feature spread across role-folders); feature envy / shotgun surgery |
| **Dependencies & extension** | DIP/IoC (framework/ORM/vendor reached from logic; unsubstitutable seam; global/singleton for collaborators); Open-Closed · Liskov · ISP (type-switch grown per variant; `instanceof` to stay correct; fat interface; override that throws "not supported"); Composition over inheritance (deep hierarchy for reuse); Stable dependencies (fragile or vaguely-named high-fan-in root) |
| **Module shape & legibility** | Deep modules / info hiding (shallow pass-through, classitis, getter/setter leakage); Local reasoning (editing 1 unit needs reading 3); Contain complexity (leaked/smeared complexity, long fn, deep nesting); Readability & Micro-conventions (unclear names, `Manager`/`Helper`, mixed verbs, boolean-mode param, inconsistent patterns); primitive obsession (Step back, under-modeled: stringly-typed state) |
| **Waste & risk** | Dead code (unreferenced, unreachable, permanently-off flag); Speculative generality (Step back, over-modeled: one-impl abstraction, unused config/param); Safety-net gaps (change-prone code, no tests); Fragile integration (no timeout/limit/retry/breaker → hand to `resilience`) |

## Prioritize (rank, don't dump)

- **Map dependency structure.** Mark each unit **root** (high fan-in — many dependents) vs **leaf** (fan-out only) by counting inbound imports/usages. Root nodes with issues rank highest — instability there radiates system-wide.
- **Three-axis targeting** for where to start: (a) code you're about to change anyway, (b) highest git-churn (`git log` hotspots), (c) core domain. A unit scoring on all three is the first target.
- **Sequence prerequisites leaves→root (Mikado):** attempt the desired change; when it forces a prerequisite, note it, revert, do the prerequisite first. Build the prerequisite graph; solve from the leaves (no-dependency tasks) inward toward the root.
- **Output:** a prioritized, ordered list — each item with issue type, location, blast radius, effort, and its place in the sequence. **No forced UPPERCASE docs** — present the plan in your response (or the format the user asks for).

## The ordered method (state which step you're in; steps 3+ touch only units the step-2 net covers)

1. **Dead code first.**
2. **Safety net — GATE.**
3. **Behavior-preserving refactors**.
4. **Then, only where you're already touching:** legibility (names, error context); deep-module consolidation (merge shallow classes that share state, delete pass-throughs — bounded by SRP, don't over-merge); dependency seams (stop logic importing the framework/ORM — start with the most-changed module).
5. **Debt-prevention:** ticket every stray TODO; converge inconsistent patterns onto one.
6. **Bounded contexts** — carve modules along domain lines via incremental migration (table below).
