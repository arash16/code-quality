---
name: resilience
description: Use when hardening a system for production, sizing capacity, adding fault tolerance to integration points, or planning a release or deployment — timeouts, circuit breakers, bulkheads, retries, bounded queries, capacity math, zero-downtime deploys, and safe migrations. Applies to new services, new features, and maintenance of existing systems alike.
---

# Resilience

Production is hostile: the integration points between services are the number-one source of outages. Design for failure explicitly. This skill is standalone — reach for it from `architect-new`, from new-feature work, or during maintenance.

## Ecosystem precedence

Use the platform's resilience primitives — the load balancer's health checks, the framework's retry/circuit-breaker library, the cloud's autoscaler, the database's migration tool — rather than hand-rolling. Don't reinvent what the platform already provides; these rules tell you *what* to configure, not to build it from scratch.

## Safety fence (read first)

Chaos experiments, failure injection, and load tests are **design-time planning and explicitly-authorized test-environment activities only**. Never execute a destructive experiment, kill production traffic, or run load against production autonomously. Propose them; let a human run them where authorized.

## Capacity sizing — do the math before adding scale machinery

- **Traffic:** `QPS = DAU × actions-per-user-per-day ÷ 86,400`; **peak = 2–5× average**.
- **Storage:** `records/day × record size × retention`.
- **Scale in order, adding each only once its specific bottleneck is measured or estimated:** vertical (bigger instance) → read-aside cache → read replicas → **shard last**. Introduce a mechanism (queue, cache, shard, new tier) only when its bottleneck actually appears — every "not yet" is a recorded decision, not an omission. Adding them all up front just multiplies failure modes.

## Integration points — every outbound call

- **Timeouts:** set a **connect and a read timeout on every outbound call**, propagated up the call chain. Tune above p99 latency, below the cascade threshold. *A call with no timeout is a defect.*
- **Circuit breaker:** `closed → open` (after N failures in a window, e.g. 5 in 60s) `→ half-open` (after a cooldown, e.g. 30s) → closed on success. Treat a tripped breaker as **expected output, not an incident** — page when it *stays* open, not when it opens.
- **Bulkheads:** isolate a resource pool per dependency (a separate connection pool for payments vs. search) so one slow dependency can't exhaust everything. Size from measured concurrency (p99 active connections + ~20% headroom), not library defaults.
- **Retries:** only for **transient** failures; exponential backoff **+ jitter**; cap attempts; enforce a **fleet-wide retry budget** (e.g. ≤20% extra load) to prevent retry storms. Never retry a non-idempotent operation without an idempotency key.
- **Fail fast:** reject a request you already know will fail (missing config, open breaker, over quota) instead of letting it time out.
- **Bounded results:** every list query has a `LIMIT`; paginate all list endpoints. *An unbounded result set is an out-of-memory crash waiting to happen.*

## Steady state & health

- Auto-purge/rotate cruft — expired sessions, temp files, logs (e.g. purge sessions > 24h, rotate logs at a size cap). Cruft that grows without bound eventually takes the box down.
- **Deep health checks** verify real dependencies (DB, cache, queue, disk headroom), not just process-alive.
- Prefer a clean **restart** over limping in an unknown state where a supervisor can recover the process safely.

## Release & deploy

- **Deploy ≠ release:** ship the code dark, turn it on with a flag. Zero-downtime is non-negotiable — rolling, blue-green, or canary.
- **Migrations are backward-compatible via expand–contract:** add the new column → write both old and new → backfill → switch reads → drop the old. Never a destructive schema change in the same deploy that still needs the old shape.
- **Rollback must be faster than roll-forward** and always available. If you can't roll back quickly, you can't deploy safely.

## Observability

- **Alert on symptoms users feel** — error rate, latency / SLO burn rate, queue depth, saturation — **not on causes** (CPU%) that may be harmless.
- Every request carries a correlation/trace id; logs and errors include operation + state context.

## Definition of Done — pre-launch checklist

| Check | If No → Action |
|---|---|
| Capacity sized from real numbers (QPS, storage, peak)? | Do the math; size before scaling |
| Only the scale mechanisms with a real bottleneck added? | Remove premature caches/shards/queues |
| Every outbound call has connect + read timeouts? | Add them — no exceptions |
| Integration points have breakers, bulkheads, budgeted retries? | Add per the patterns above |
| Every list query bounded / paginated? | Add `LIMIT` + pagination |
| Retries only on transient errors, idempotency ensured? | Guard non-idempotent calls |
| Health checks verify dependencies, not just liveness? | Deepen them |
| Deploy decoupled from release; zero-downtime path? | Add flag/rolling/canary |
| Migrations backward-compatible (expand–contract)? | Restructure the migration |
| Rollback faster than roll-forward and tested? | Make rollback the fast path |
| Alerts fire on user-felt symptoms, requests traceable? | Re-point alerts; add trace ids |
| Any destructive/chaos test aimed at production? | Stop — propose it for an authorized human to run |
