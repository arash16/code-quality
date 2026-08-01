---
name: architect-new
description: Use when architecting a new system, service, or significant feature before writing much code — choosing module boundaries, a domain model, dependency direction, and data/consistency decisions. Makes only the expensive-to-reverse decisions deliberately and defers the rest, defaults to a modular monolith, forces assumptions into the open, and keeps business rules independent of frameworks. A counterpart to clean-dev.
---

# Architect New

Design the decisions that are **expensive to reverse**; defer everything cheap. This skill produces a deliberate boundary + domain + data design — not code. Decide *with the user* at each fork.

## Ecosystem precedence

Honor the platform's architecture conventions (framework project layout, cloud primitives, team standards). These rules shape *your domain design within* those conventions — they don't override the platform.

## Operating rules

- **Requirements before solutions.** List functional and non-functional requirements and get explicit agreement before designing. A design built on unstated assumptions is built on sand.
- **A decision made silently is a defect.** At every fork, present options with a recommendation, decide with the user, and record the choice plus the rejected alternatives (one line each).
- **Expensive deliberately, cheap deferred.** Convert an expensive decision into a cheap one by putting a boundary in front of it. *Expensive to reverse:* data model/storage, public contracts, service/process boundaries, sync-vs-async, a technology that leaks everywhere. *Cheap:* internal names, a class's shape behind a stable interface, a library you've wrapped.
- **Default to a modular monolith.** Split into services only when a specific force demands it — independent scaling, team autonomy, separate release cadence, or fault isolation. A microservice sharing a database with another is a **distributed monolith**, strictly worse than either.

## The order — hardest-to-reverse first

Each step decides with the user and feeds the next. Stop at the depth the project needs — don't design a platform for an MVP.

1. **Boundaries — where.** Identify domain capabilities (checkout, billing, inventory) and draw module boundaries around them. Put boundaries at points of likely **change/volatility** and at **linguistic seams** (the same word meaning different things in different areas). Start with modules in one codebase — a boundary is a module long before it's ever a service.
2. **Dependency direction — which way.** Dependencies point inward: framework/IO/UI → application/use-cases → domain. The domain **defines** interfaces (e.g. `OrderRepository`); infrastructure **implements** them. No framework/ORM/vendor type crosses into the domain — translate to plain domain objects at the edge. Test: *can the core rules run with no DB, web server, or framework?*
3. **Domain model — what lives inside.**
   - **Ubiquitous language:** name concepts as the business names them. If a concept is hard to name, the model is probably wrong.
   - **Entity vs Value Object:** Entity = identity persists though attributes change; Value Object = defined only by its attributes (most things should be Value Objects — immutable, compared by value).
   - **Aggregates:** keep them small (one root + the minimal cluster that must stay consistent together). Enforce invariants **inside** the aggregate, not in services. Reference other aggregates **by ID**, not object reference. Consistency is immediate inside an aggregate, eventual between them.
   - **Make invalid states unrepresentable** where the language allows (constructors/factories/types that reject bad input).
   - **Anti-Corruption Layer** at every external integration: wrap foreign models into your own at the boundary; never let a foreign model leak into the core.
4. **Data & consistency.** Choose storage from access patterns and required guarantees, not familiarity — and be able to justify it over the alternatives. Know the datastore's **default isolation level** (most default to read-committed/snapshot, *not* serializable) and guard read-then-write paths against **write skew** (`SELECT … FOR UPDATE` or equivalent). Prefer single-partition operations; use sagas + compensating actions over distributed transactions. Separate the **system-of-record** from derived/denormalized data so derived stores can be rebuilt from source.

> Scaling math, timeouts, circuit breakers, and deploy strategy → the `resilience` skill. Don't scale for load you don't have; wrap volatile choices behind boundaries so you can revisit them cheaply.

## Smells

- A core concept is named `Manager`, `Helper`, `Processor`, `Util`, `Data`, or `Info` — the model is unclear, so find the real domain name.
- Entities are bags of getters/setters while all behavior lives in services (an anemic domain) — push the behavior onto the model that owns the data.
- A foreign model (an ORM row, an API DTO, a framework type) is used as a domain object deep inside the core instead of being translated at the edge.
- A "microservice" cannot be deployed or tested without another service's database.
- Boundaries are drawn by technical layer (controllers/services/repos) rather than by business capability.
- Sharding, queues, or event-sourcing are designed in before any measured need for them.

## Definition of Done

| Check | If No → Action |
|---|---|
| Functional + non-functional requirements listed and agreed? | Gather and confirm before designing |
| Every fork decided with the user, alternatives recorded? | Surface the open decisions now |
| Can the core rules run with no DB/framework/network? | Fix the dependency direction |
| Invariants enforced inside aggregates, not services? | Move them in |
| Aggregates small, referencing others by ID? | Split; replace object refs with IDs |
| No foreign model leaking into the core (ACL present)? | Add the anti-corruption boundary |
| Storage justified by access patterns, isolation level known? | Re-justify; check write-skew paths |
| Defaulted to a modular monolith unless a force demands services? | Collapse premature services |
| Only expensive-to-reverse decisions made now, rest deferred behind boundaries? | Defer the cheap ones |
