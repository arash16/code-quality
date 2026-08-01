---
name: review-and-refactor
description: Use when asked to review, audit, or assess existing code and produce an ordered, safe plan to pay down technical debt or refactor. A counterpart to clean-dev.
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
- **Cap concurrent workers (≈3–5), and total workers (≈6–10):** The total number of workers fanned out should be strictly sane relative to the LOC and complexity of the target. For example, a 10k LOC target deserves no more than 3-5 workers. If the repository is much larger than can be reasonably audited by a handful of workers, then ask the user to narrow the scope to a smaller slice of the codebase.

### The catalog — `clean-dev` violations, grouped so a worker can take one family

| Family | Flag violations of (`clean-dev` anchor · compressed smell) |
|---|---|
| **Duplication & placement** | DRY (same rule/formula/constant in N places); One way per problem (rival libraries or patterns for one job); Minimal blast radius (one foreseeable change → many files) |
| **Responsibility** | SRP (god class; rule beside SQL/HTTP); Separation of concerns / orthogonality; Cohesion by feature (one feature spread across role-folders); feature envy / shotgun surgery |
| **Dependencies & extension** | DIP/IoC (framework/ORM/vendor reached from logic; unsubstitutable seam; global/singleton for collaborators); Open-Closed · Liskov · ISP (type-switch grown per variant; `instanceof` to stay correct; fat interface; override that throws "not supported"); Composition over inheritance (deep hierarchy for reuse); Stable dependencies (fragile or vaguely-named high-fan-in root) |
| **Module shape & legibility** | Deep modules / info hiding (shallow pass-through, classitis, getter/setter leakage); Local reasoning (editing 1 unit needs reading 3); Contain complexity (leaked/smeared complexity, long fn, deep nesting); Step back (edge-case accretion: branch stacks and flag combinations a reformulation would delete); Readability & Micro-conventions (unclear names, `Manager`/`Helper`, mixed verbs, boolean-mode param, inconsistent patterns); primitive obsession (Step back, under-modeled: stringly-typed state) |
| **Waste & risk** | Dead code (unreferenced, unreachable, permanently-off flag); Speculative generality (Step back, over-modeled: one-impl abstraction, unused config/param); Single source of truth (stored derived values, fields kept in sync by hand, caches needing manual invalidation); Symptom patches (defensive check hiding a broken invariant, recurring bug class with an unfixed cause); Undiagnosable code (no logging seam, scattered prints, un-actionable or duplicated log lines, secrets in logs); Safety-net gaps (change-prone code, no tests); Fragile integration (no timeout/limit/retry/breaker → hand to `resilience`) |

## Prioritize (rank, don't dump)

- **Dead Code, and Big Wins first** — if you can delete it, do it. If you can replace a big chunk with a library, do it. If you can refactor a big chunk to be simpler, do it. These are the first targets, before any further changes that could be simplified by them.
- **Map dependency structure.** Mark each unit **root** (high fan-in — many dependents) vs **leaf** (fan-out only) by counting inbound imports/usages. Root nodes with issues rank highest — instability there radiates system-wide.
- **Three-axis targeting** for where to start: (a) code you're about to change anyway, (b) highest git-churn (`git log` hotspots), (c) core domain. A unit scoring on all three is the first target.
- **Sequence prerequisites leaves→root (Mikado):** attempt the desired change; when it forces a prerequisite, note it, revert, do the prerequisite first. Build the prerequisite graph; solve from the leaves (no-dependency tasks) inward toward the root.
- **Output:** a prioritized, ordered list — each item with issue type, location, blast radius, effort, and its place in the sequence. **No forced UPPERCASE docs** — present the plan in your response (or the format the user asks for).
