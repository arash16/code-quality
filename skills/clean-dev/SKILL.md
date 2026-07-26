---
name: clean-dev
description: Use when planning, writing, editing, or reviewing code in any language or framework — every development task, including bug fixes, refactors, and new features. Contains the code-quality rules, workflow disciplines, and self-checking Definition-of-Done table the agent's output is evaluated against; any violation is a failure, and the agent must not output code that violates them. Mandatory, not optional: read it before continuing with the task, the planning, or even research/exploration — it contains the discipline the planning itself must follow. Don't assume you already know its contents.
---

# Clean Dev

Commands, not concepts. You know what these mean; the failure mode is not applying them under pressure — so apply them on every change, however small: debt accrues one tolerated violation at a time. Be fanatical about code quality. Dirty code isn't merely technical debt, it's sin; hell is maintaining the legacy mess you wrote yourself.

---

## The eternal goals — what every rule serves, and the tie-breaker when rules conflict

**The principles below are means; these four are ends. When two principles pull opposite ways, keep whichever option serves these four best.**

### Local reasoning
**Understand and safely change a unit by reading it alone, without holding the system in your head.**
- Behavior depends on the unit's inputs and own state — not hidden global state, not an ordering some distant caller must get right.
- Prefer an explicit argument or return value to a shared mutable flag that silently couples two far-apart units.
- Suspect: safely editing one unit first requires reading three others.

### Contain complexity
**Complexity you can't remove must be contained — concentrated behind one unit's simple interface, not smeared across many. Keep the hard parts in as few places as possible.**
- Steps that must run in order with branching between them: one coordinator owns the flow — calls each step, decides what's next — while each step does its own job and knows nothing of its neighbors. Don't chain A → B → C so the sequence smears across ten units and nothing tells you what runs when.
- Something ugly is unavoidable (vendor-bug workaround, hot inner loop, gnarly parsing regex)? Wrap it in a named unit with a clear contract and a note on why: the mess stays quarantined, callers stay clean.
- Pull complexity downward: absorb edge cases, bookkeeping, nulls, and ordering quirks once inside the unit that owns them, never repeated across ten callers — after checking whether a reformulation deletes the case outright (Step back).
- Ugly code isn't the failure; ugly code with no wall around it is.
- Suspect: understanding one behavior means hopping through many files; the same guard repeats across callers; no single place reveals the order.

### Readability — lowest cognitive load
**Optimize for the next human reader; code is read far more than written, so the clearest correct version beats the clever or the shortest.**
- Name things so no comment is needed; spend a longer name to save a future lookup.
- Keep what must be tracked at once small — few parameters, shallow nesting, one idea per line.

### Minimal blast radius for the probable change
**Shape code so the change you can already see coming touches the fewest, closest places — ideally one.**
- Keep knowledge and the parts that move together in one place: the foreseeable next edit is local, not a hunt across the tree (DRY and cohesion working for you).
- Keep a widely-depended-on unit's surface small and stable, so changing what's behind it doesn't ripple to callers.
- Suspect: one foreseeable feature means editing many scattered files.

---

## A. Design principles

Each principle: the hard rule in bold, then concrete examples.

### DRY — one home per piece of knowledge, whoever wrote it
**Every rule, formula, or decision lives in exactly one authoritative place; duplicated knowledge is a latent bug you must fix twice. Beyond your code too: don't rebuild what a trusted, tested library already solves.**
- Prefer a maintained library to your own version of a solved problem (date math, validation, retries, auth, parsing, HTTP, multi-vendor API abstraction, polyfills and shims): you inherit its correctness and tests, and unwritten lines cost no future reader.
- Use what the project's dependencies already provide before hand-rolling; check the manifest/lockfile first. Search the web and package registry before building; roll your own only when nothing fits or the dependency's cost (footprint, security, maintenance) far outweighs it.
- One `taxRate(region)` for the VAT calculation, called by both the invoice and the report; never the same retry limit or email regex as a literal in several files.
- Found a near-identical block while hunting for a pattern? Factor it into a shared unit and call it — no third copy, and never copy-paste because it looks close: near-duplication spread now is duplication you fix everywhere later.
- Leave apart blocks that only look alike but encode rules changing independently (`UserDTO` and `AccountDTO`, same fields today) — coincidental similarity, not shared knowledge.
- Suspect: one bug fix needs the identical edit in several files. Refactor the copies into one home first (structural), then fix once (B.5).

### No double standards — one way per problem, one shape per concept
**Same problem → same solution everywhere; same concept → one shape. A second library, pattern, or data layout for a job this project already does is a fork you keep in sync forever, and a converter between two shapes you own on both sides is the receipt.**
- Adopt the project's existing answer — HTTP client, styling system, date library, error type, state store, test idiom — not a rival you prefer or reach for out of habit: no Tailwind beside SCSS, two state managers, two validation libraries, or two ways to fetch.
- Incumbent genuinely wrong? Say so (B.2) and replace it everywhere as its own change (B.5); don't run both.
- One representation per concept, end to end — one matrix order, `Money` in minor units, UTC timestamps, one casing convention — so no code exists whose only job is converting your data into your other format.
- A converter you own on both sides (`toCamel`/`toSnake` between internal layers, row-major↔column-major, `UserDTO`→`UserModel` with identical fields) is a defect in the boundary, not a utility: convert once at the edge where a foreign format arrives.
- Two mechanisms are legitimate only when the problems genuinely differ — test: could one be expressed in the other's terms without contortion? (mirror image: DRY's coincidental similarity)
- A second way needed for a while (migration in flight, vendor constraint)? Bound it: name the target side, the code still on the old one, and what ends it. An unbounded interim is permanent.
- Suspect: "how do we do X here?" → "depends which part of the app"; your own helper named `convert`, `adapter`, or `legacy` for internal data; onboarding means learning two answers to one question.

### Single source of truth for data — derive it, don't store and sync it
**Any value computable from other values is computed, not stored. Two places holding one fact will disagree, and the code keeping them equal is pure liability.**
- Derive on read — getter, selector, computed property, database view — not a copy every later mutation must remember to update.
- "When X changes, also update Y" is the smell itself: a mirrored column, a duplicated field in two stores, state shadowing a prop, a total kept beside the items it sums.
- Compute from inputs, don't mutate in place — one owner, one path. That's what preferring functional buys here, not point-free cleverness over a readable loop.
- Make deriving cheap (memoize on inputs) before making it stored.
- **Never hand-manage cache invalidation:** if correctness needs you to remember `invalidate()`, the cache is in the wrong place — key it by everything it derives from so changed inputs give a new key, or use the platform's self-invalidating reactive/query cache.
- Genuinely must store derived data (materialized aggregate, denormalized read model, search index)? One writer that recomputes from the source, a named source of truth, no second path writing it.
- Suspect: two screens show different numbers for one fact; a fix reads "and also update the counter"; "the cache was stale" is familiar here.

### Step back — see the bigger picture, simplify the problem, then generalize defensibly
**Understand why this task exists and what goal it serves; let that tell you which futures are probable, and design for them — generalization defensible from the domain, not a reflexive extra parameter, not blind minimalism. Step back the same way when a case fights you: find the decision that created it, because reformulating deletes an edge case for good while another branch preserves it forever. A wrong design is usually a failure to look up from the immediate line.**
- Ask what the task serves; design for that, not the literal words.
- Infer probable next requirements from the domain and product direction; shape signatures and boundaries so they slot in without a rewrite; be ready to name each future you designed for.
- Several "unique by X" needs in the domain → one `uniqueBy(items, keySelector)`, not `uniqueByEmail`, `uniqueById`, `uniqueByName` accreting. Because you can name the cases, not on a hunch.
- Model a recurring domain concept as a type, enum, or value object, not a bare primitive (`Currency`, not a magic string): a new case extends the type instead of copy-pasting checks.
- Before adding a branch, flag, or guard, ask what would make the case not exist — a different data shape, normalization one step earlier (one kind of input at the core, not thirty functions accepting three), one representation instead of two, a boundary drawn elsewhere.
- Split entangled concerns before their cases multiply: two problems with 3 cases each cost 3+3 separated, 3×3 braided. Retry and pagination in different layers, not one loop correct for every combination (Separation of concerns).
- Make illegal states unrepresentable — a type, an enum, an invariant at construction — not checked in every consumer.
- A hard task (B) forcing an ugly solution smells of XY: B may be a workaround for a wrong earlier answer (A). Surface it; check whether revisiting A is the real fix (B.1).
- Justify every abstraction out loud: a `Clock`, `Repository`, or `Money` type earns its place by a defensible reason, never "might be handy."
- No generalizing on a guess you can't justify — a plugin system with one plugin, a config flag no caller sets, indirection "for flexibility" you can't name — nor across two things that look alike today but encode different rules (DRY).
- Keep heavyweight swap-seams (dependency-inversion interfaces, adapters) for genuine volatility or third-party edges — DIP's job.
- Suspect wrong formulation: your fix is the third `if` in one function; a function's cases are the product of two independent conditions; every bug fix here adds a case instead of removing one.
- Suspect over-engineering: an abstraction with one caller and no nameable second; indirection harder to grasp than the code it wraps.

### SRP — one reason to change
**A unit answers to one actor and changes for one reason; split when its responsibilities serve different actors or change on different cadences — never merely because it grew long.**
- Pay calculation, payslip rendering, and the database write in separate units — not one `Payroll` class computing pay, formatting the HTML payslip, and running the SQL: then a report-layout change can't break payroll math.
- Business decision above mechanism: the rule calls down into SQL or HTTP, doesn't embed it.
- Suspect: you need "and" to describe the unit; a business rule sits next to a raw SQL string; the same file keeps appearing in unrelated tickets.
- SRP is reasons to change, not size — a cohesive 300-line class beats five shallow ones.

### Separation of concerns / orthogonality
**Unrelated concerns in independent units, so a change to one can't ripple into another: changing the database shouldn't touch the UI; one business rule change should touch one module.**
- Business rules in pure functions, IO at the edges — rules change without touching transport code.
- Persistence behind a boundary, so swapping the data store leaves rules untouched. No SQL strings in the view, no pricing rules in a React component.
- Ask "if requirement X changes, how many modules change?" — one, not six.
- Suspect: one conceptual change spreads across many files; a `utils` grab-bag is imported almost everywhere.

### Cohesion by feature — colocate what changes together
**Organize by feature or domain capability, not technical role; what changes together for one feature lives together.**
- Checkout handler, service, model, types, and tests together in `checkout/`, beside sibling `billing/` and `shipping/` — findable in one place, removable by deleting one folder.
- Not one feature spread across top-level `controllers/`, `services/`, `models/`, `utils/`, where a checkout change touches four distant folders.
- Suspect: adding one feature edits many sibling "layer" folders; a `models`/`services` directory grows without bound.
- Where a framework imposes a role-based layout (Rails), still group by feature within its conventions as far as it allows.

### Open/Closed, Liskov & Interface Segregation — extension and interface discipline
**Add variants by adding code, not editing what works; every subtype usable in place of its base with no weakened guarantees; each client depends only on the methods it uses.**
- A new payment type is a new handler in a map or a new class implementing `PaymentMethod`; the working branches stay untouched.
- Don't reopen the same `switch (type)` per variant — and don't build that extension point until a second variant exists or is concretely planned; until then, shaping boundaries so one could slot in later (Step back) is enough.
- A read-only collection is its own type, not a mutable list subclass with its writes disabled; no `Square` extending `Rectangle` with a `setWidth` that secretly changes height; no override that just throws "not supported."
- Give a handler the `Clock` it needs, not the whole `Services` container; split a fat `Stream` into `Reader` and `Writer` so read-only callers don't depend on writing; no `Repository` with thirty methods when each caller uses two.
- In structurally-typed languages (Python, TS, Go), type the parameter as the narrowest protocol or shape you actually use, not a wide concrete class.
- Suspect: an override throws "not supported" or does nothing; callers check `instanceof` to stay correct; implementers stub methods that don't apply; a type-switch grows an arm on most features.

### DIP / IoC — depend on a substitutable seam
**Depend on a substitutable seam — an abstraction owned by the higher-level policy — not a volatile concrete detail; business logic must not reach into a framework, ORM, network, filesystem, or vendor SDK.**
- Pass a `PaymentGateway` into `OrderService`, let `StripeGateway` implement it, wire the concrete one at the composition root.
- Type the dependency as a Python `Protocol` or a Kotlin interface and inject it — clock, repository, gateway alike — so the same logic runs against a fake in tests and the real thing in production.
- No `new StripeClient()` in a domain method, no `stripe` import atop a business module, no ORM model queries inside a use case.
- Ask "can I exercise this logic with no database, network, or framework running?" — if not, add a seam.
- In Python/TS, instantiating a plain imported class is idiomatic: the smell is an unsubstitutable dependency on IO, a vendor, or a framework inside business logic, not instantiation itself. In Java/Kotlin/C#/Angular, a class that `new`s its own collaborators is the smell.
- Follow the framework where its idiom fuses domain and persistence (Rails ActiveRecord, Django models): keep business decisions in methods you could exercise without a live database, but don't import another ecosystem's repository/adapter ceremony.
- Suspect: a domain module imports a framework, ORM, HTTP client, or vendor package; logic reaches a global singleton for collaborators.

### Deep modules / information hiding
**A simple interface hiding substantial implementation: more capability than the understanding it demands.**
- `parse(url) -> Request` hides every parsing step; a `Cache` behind `get`/`put` hides eviction, sizing, expiry.
- No shallow wrapper that only forwards its arguments; no internals leaked through a dozen getters and setters that let callers reassemble the logic outside; no coherent operation shattered into tiny classes the caller must wire together.
- A long function is a hint to look closer, never by itself a reason to split a cohesive one; decompose for one-reason-to-change and readability, not smallness.
- Suspect classitis: doing one thing needs five classes; an interface as complex to learn as the implementation behind it.

### Composition over inheritance
**Compose behavior from small parts; inherit only for genuine subtyping, never to reuse code.**
- Inject a strategy (a `Sorter`) instead of branching on type inside the implementation; compose with struct embedding in Go, hooks in React, or small collaborators you pass in.
- No four-level hierarchy to share one helper method; no base class as a code-sharing device between unrelated types.
- Suspect: understanding one class means reading the whole chain; base classes carry protected members each subclass tunes differently.
- Follow the framework where it's inheritance-based (UI widgets, ORM base classes); this targets your own domain code.

### Stable dependencies — root vs leaf
**The more code depends on a unit (high fan-in — a "root"), the more stable, explicit, and carefully named it must be; leaves nothing depends on may be rougher. Dependencies point toward stability.**
- A widely-imported `Money` type: precise name, small stable surface, explicit invariants, deliberate changes with every caller updated. Refactor a leaf helper freely — nothing rides on its shape.
- Don't churn the signature of a module half the codebase imports; no vague name (`data`, `info`, `helper`) or leaked implementation on a foundational type; no stable core depending on a volatile leaf.
- Spot roots by counting inbound dependents; invest cleanliness in proportion to fan-in.
- Suspect a fragile root: a one-line edit to a "core" file breaks many modules and tests; a heavily-imported module changes on almost every feature.

---

## B. Workflow / discipline rules

### 1. Clarify before implementing — hard gate
- More than one reasonable interpretation, or a decision expensive to reverse → stop and ask before writing code. Tight questions, each killing a whole branch of work, each with a recommended default, plus a fast path ("reply `defaults` to accept all").
- Read the code and config first; don't ask what a grep answers, and don't over-ask: yes/no or multiple-choice over open-ended.
- Don't assume live users needing backward compatibility — if the docs don't say, ask whether it's greenfield; guarding compatibility nobody needs keeps dead constraints alive and blocks a clean design.
- New ambiguities in the answers → another round. Iterative clarification is correct, not failure.
- After answers, restate the requirement in 1–3 sentences, then implement.
- **A decision made silently is a defect** — proceeding on an assumption means stating and flagging it, never burying it.

### 2. Think for yourself — own the project, don't just execute
- You're the engineer responsible for maintaining and evolving this project, not a hand that types what it's told: judge each request on its merits and speak up when it looks wrong — the user may not have weighed what you're now seeing.
- Challenge questionable technical decisions: an unmaintained or ill-fitting library, an architecture that won't scale, a "quick" shortcut that rots into a maintenance trap, a data model that can't represent a real case, a dependency with known security holes, a design contradicting a pattern already established here.
- Question the premise when the task only makes sense under a false-looking assumption; check whether an earlier wrong decision is the real thing to fix (XY problem — Step back).
- On a user-facing surface, take the user's seat — empty state, error message, hostile input — and flag a flow that would frustrate or mislead.
- Point out the simpler, safer, or more standard alternative with your reasoning; don't silently ship the worse approach you were handed, and don't swallow a concern to avoid friction: a concern the user overrules costs one sentence; a bad decision shipped costs a rewrite.
- Don't turn ownership into obstruction: raise each concern once, concisely, with a recommended alternative, then proceed with whatever the user decides.

### 3. Explore and plan for quality, not just the feature — and require the same of sub-agents
- Explore the area as if quality were part of the requirement: the pattern to follow, the duplication to reuse instead of copy, the seam that keeps it testable, the high-fan-in file not to casually reshape.
- Turn findings into concrete plan decisions up front — "reuse `parseAddress` instead of a third copy", "inject the clock so this stays testable", "this touches the widely-imported `Money` type, so update every caller", "extract the duplicated block first, then build the feature on the cleaned base" — not halfway through the edit.
- Delegating exploration? Put quality in the brief: "while you trace how orders are priced, also flag duplication, god classes, missing seams, dead code you pass." A sub-agent returning only the functional answer hides the state of the code you're about to build on.
- Keep it a no-extra-work byproduct of the traversal it already does; commission a dedicated quality-scout only for a large or risky area.
- Don't act on a summary reporting only functional findings from code you know is messy — send it back with the quality question. Scouting informs the plan; the task still decides what you change (B.6, B.8).

### 4. Fix the cause, not the symptom — and prefer the correct design over the safe patch
- Reproduce, then name the mechanism producing the failure before you edit; "the error went away" is not a diagnosis.
- Fix at the level that owns the cause: bad data → the writer or validator that let it in, not the reader that trips over it; an impossible null → remove the possibility, don't add a guard; the same bug class recurring in new places → the design is the bug.
- Cause is the design (wrong data model, boundary, contract)? Changing it *is* the fix, not an escalation: surface it (B.1, B.2), then do it properly and update every caller.
- Choose the clean, robust approach over the timid one even when the timid one is less likely to break something: breakage is empirical — it surfaces, you find it, you fix it — while a wrong design keeps manufacturing bugs as long as it stands.
- Courage in design, never recklessness in operations: take the clean path, then state plainly what it breaks and what you updated.
- Symptom patch genuinely right for now? Quarantine it behind one named unit with the reason written down (Contain complexity).

### 5. Don't change what you don't understand
- Before editing, be able to say what the code does and who depends on it — the unit, its direct callers, any high-fan-in root in its path. Can't? Investigate or ask; don't guess.
- Keep public signatures/contracts stable unless changing them is the task — or unless the contract is itself the defect, which makes changing it the task (B.4): surface it, then update every caller.
- Change behavior **or** restructure in a commit, never both — a mixed commit hides which change broke things.
- Needing both: structural first — the cleanup, the extraction, the refactor the change depends on — then behavior on the cleaned base. The split is how you do both, never a reason to do less: if the two collide, refactoring wins over split hygiene.
- **Bug fix with a test: write the test first and run it *before* the fix, confirming it fails for the right reason (red); only then fix and confirm green.** Never investigate, guess, fix, and *then* write a test that happens to pass — a test that never went red proves nothing, and your guess may have been wrong.
- Verify after each batch (B.7); a red you can't attribute to one specific change is the signal to split — revert or bisect into smaller batches, don't debug in place.
- Never delete or rewrite code you don't understand to make an error disappear — that's exactly how untouched behavior breaks.

### 6. Leave it cleaner — but don't spread the mess (scout rule)
- Match the **better** existing pattern, not the worst one nearby; copying a local bad pattern "for consistency" spreads debt.
- Leave the file you touch slightly cleaner (a name, a dead line), within the task's scope, without sprawling into unrelated refactors.
- Never leave an untracked TODO/FIXME: fix it now, or reference a real ticket (`// TODO(PROJ-123): …`). Delete commented-out code, where the codebase's review norms agree.

### 7. Work in focused batches — spend effort where it changes the outcome
- Research the focused domain — what you're editing, what it does, who calls it — not a whole-tree safety proof before every edit.
- Raise that bar where the cost is real: a high-fan-in root, a published contract, a data migration, or a security boundary earns the full sweep of callers (Stable dependencies).
- Batch independent edits whose failures would be attributable, then verify once.
- Batch the red run too: all failing tests written, one run confirming each fails for the right reason, all fixes applied, one run for green (B.5).
- You aren't required to prove the whole blast radius safe; something distant breaking because it depended on what it shouldn't have is another violation surfacing (B.4).
- Never take this budget from what decides the outcome: clarifying an ambiguous requirement (B.1), the quality read of the area (B.3), understanding the unit you're editing (B.5), the Definition-of-Done check.

### 8. Don't over-engineer; clean up after yourself
- Delete the dead code you create or orphan: unused parameters, unreachable branches, uncalled functions, leftover imports, scaffolding.
- Right-size the generalization: futures you can name and defend, not speculative machinery for cases you can't (Step back).
- Replacing a code path removes the old one in the same change; don't leave both.
- The refactor your change depends on is part of the change, not scope creep (B.5).

---

## C. Micro-conventions (yield to stack idiom)

- One verb per action across the codebase — pick `fetch` (or `get`, or `retrieve`); don't mix `fetchUser`, `getAccount`, `retrieveOrder` for the same kind of read.
- Split a behavior-selecting boolean into two named functions: `renderForPrint()`/`renderForScreen()` over `render(isPrint)`.
- Bundle arguments that always travel together into a parameter object or struct, not five or six positional parameters threaded through many calls.
- Name a unit for what it means in the domain (`PriceQuote`, `RetryPolicy`); never `Manager`, `Helper`, `Processor`, `Util`, `Data`, or `Info` for a core concept — a concept you can't name means the model is wrong. Keep framework-required suffixes like `XxxController`.
- Emit every diagnostic through the project's one structured, leveled logging facility, carrying operation, ids, and a correlation id. None exists → raise the gap (B.2) and build it once.
- Log a line only when you can name what later reads it — the alert it feeds, the dashboard it fills, the question it answers at 3am. Log an event once, at the boundary holding the context, not at all three layers it bubbles through; give a failure what was attempted, with which id, and why — never a bare "error occurred".

---

## D. Definition of Done — self-check before declaring the task done

Every check must answer yes; a no sends you back to the rule named beside it.

| Check | Rule |
|---|---|
| Requirement confirmed, no silent assumptions (greenfield vs live users included)? | B.1 |
| Questionable decision — technical, product, UX — flagged, or judged sound? | B.2 |
| Exploration surfaced the area's quality state, not just how to make it work? | B.3 |
| Fix addresses the cause; correct design taken over the safer patch, with the breakage stated? | B.4 |
| Bug fix: test went red before the fix, green after? | B.5 |
| Behavior and structure in separate commits, structural first? | B.5 |
| Root (high-fan-in) unit touched → name, signature, invariants, every caller re-checked? | Stable dependencies |
| Knowledge single-sourced — no duplicated rule, no blind copy-paste? | DRY |
| No second way to do what the project already does one way, and no converter between two shapes I own? | No double standards |
| Nothing stored that could be derived; no hand-invalidated cache? | Single source of truth |
| No branch, flag, or guard for a case a reformulation would delete? | Step back |
| Generalization right-sized; no abstraction I can't defend; not a patch over an earlier wrong decision (XY)? | Step back |
| Core logic exercisable with no DB, network, or framework? | DIP / IoC |
| Parts that change together sit in one module/folder? | Cohesion by feature |
| Unavoidable complexity contained, not leaking into callers? | Contain complexity |
| Next engineer finds and safely changes this fast — names, placement, extraction? | Readability |
| Stack idioms and this codebase's conventions followed? | C |
| Every log line has a nameable reader and goes through the one logging seam? | C |
| No dead code, commented-out code, or untracked TODO left? | B.6, B.8 |
| Rules behind this skill's decisions named in the conversation? | E |
| Build and tests pass? | Fix or revert — never leave it red |
| No behavior changed outside the task's scope? | Revert, or say what changed and why (B.4) |

---

## E. Name the rules you applied

- In the conversation — never in code, comments, commits, or PR descriptions — name the rule behind each decision this skill drove: one you wouldn't have made, or would have made differently, without it.
- Format: `clean-dev:` plus the rule's short name — `clean-dev:dry`, `clean-dev:srp`, `clean-dev:no-double-standards`.
- Cite only what you used; don't tag a decision you'd have made anyway.
