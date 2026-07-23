---
name: clean-dev
description: Use when planning, writing, editing, or reviewing code in any language or framework. Includes every special rule evaluated and expected from the agent regarding code quality, workflow disciplines to follow, and a self-checking Definition-of-Done table. These should be followed for any kind of development task, including bug fixes, refactors, and new features. Agents' final output is evaluated against this skill, and any violation is a failure. The agent should not output code that violates these rules. Never continue with user's task or planing or ever research/exploration before reading this skill. It contains the very discipline you need to follow to do the planing.
---

# Clean Dev

These are commands, not concepts. You already know what these principles mean; the failure mode is not applying them under task pressure — so follow them on every change, however small, because debt accumulates one tolerated violation at a time. Be fanatically, religiously devoted to code quality: dirty code is not merely technical debt, it is sin. Hell is maintaining the legacy mess you wrote yourself; salvation is a green, maintainable codebase you can live in.

## Ecosystem precedence — read first

- The project's language, framework, and idiom best practices **override any rule here**. Follow the stack.
- A principle's **intent** is universal; its **mechanism and smells are often language-specific**. Where a rule is scoped ("where applicable", a named stack), apply the intent — use the mechanism only if it fits.
- **Never import a convention from another ecosystem.** If a rule here would fight an idiomatic best practice of the current stack, the stack wins — note the conflict in one line and move on.
- Match the conventions already established in this codebase (naming, layout, error handling) over your personal defaults.

---

## A. Design principles – The Ten Commandments

Each principle opens with the hard rule (in bold), followed by concrete examples to follow.

### DRY — one home per piece of knowledge; don't reinvent the wheel
**Every business rule, formula, or decision lives in exactly one authoritative place; duplicating that knowledge means fixing it in two places later — a latent bug. This extends beyond your code: don't rebuild what a trusted, tested library already solves.**
- Prefer a well-maintained external library over your own version of a solved problem (date math, validation, retries, auth, parsing, HTTP, abstraction on multi-vendor APIs, polyfills and shims, etc) — you inherit its correctness and tests, and every line you don't write is a line no future session has to read and understand.
- Use what the project's existing dependencies already provide before hand-rolling the same capability; check the manifest/lockfile and reach for their helpers first.
- For any task or sub-problem, search the web and the package registry for a ready, maintained solution before building it, and write your own only when nothing fits or the dependency's cost (footprint, security, maintenance) far outweighs it.
- Keep the VAT calculation in one `taxRate(region)` function that the invoice and the report both call.
- When you search the codebase for a pattern to follow and find a near-identical block, factor it into a shared, reusable unit and call it, rather than adding a third copy.
- Don't copy-paste a pattern you found just because it looks close — near-duplication you spread now is duplication you must fix in every copy later.
- Don't encode the same retry limit or email regex as a literal in several files.
- Leave apart two blocks that merely look alike but encode different rules that will change independently (a `UserDTO` and an `AccountDTO` with the same fields today) — that is coincidental similarity, not shared knowledge.
- Suspect a DRY violation when one bug fix requires the identical edit in several files. Here the fix includes refactoring out the shared code.

### Step back — see the bigger picture, then generalize defensibly
**Before implementing, step back and understand why this task exists and what larger goal it serves; let that understanding tell you which futures are probable, and design for them. The aim is intelligent generalization you could defend in review from the domain — not a reflexive extra parameter, and not blind minimalism. Under-thinking the whole is the common failure: a wrong design is usually a failure to look up from the immediate line.**
- Ask what the task is really in service of and design for that goal, not merely for the literal words of the request.
- Infer the most probable next requirements from the domain and product direction, shape signatures and boundaries so those cases slot in without a rewrite, and be ready to name each future you designed for.
- When the domain plainly has several "unique by X" needs, write one `uniqueBy(items, keySelector)` rather than letting `uniqueByEmail`, `uniqueById`, and `uniqueByName` accrete over time — do it because you can name those cases, not on a hunch.
- Model a recurring domain concept as a type, enum, or value object rather than a bare primitive (a `Currency` type, not a magic string), so a new case extends the type instead of copy-pasting checks.
- When a hard task (B) forces an ugly solution, suspect an XY problem — B may be a workaround for a wrong earlier answer (A); surface it and check whether revisiting A is the real fix (see B.1 Clarify before implementing).
- Justify every abstraction out loud: a `Clock`, a `Repository`, or a `Money` type earns its place by a reason you could state and defend, never "might be handy."
- Don't generalize on a guess you cannot justify — a plugin system with one plugin, a config flag no caller sets, or indirection added "for flexibility" you cannot name is over-engineering, not foresight.
- Don't generalize across two things that merely look alike today but encode different rules (see DRY) — coincidental similarity is not a reuse case.
- Keep heavyweight swap-seams (dependency-inversion interfaces, adapters) for genuine volatility or third-party edges — that is DIP's job.
- Suspect over-engineering when an abstraction has exactly one caller with no plausible second you can name, or when the indirection is harder to comprehend than the code it wraps.

### SRP — one reason to change
**A unit answers to one actor and changes for one reason; split it when its responsibilities serve different actors or change on different cadences, never merely because it grew long.**
- Separate the pay calculation, the payslip rendering, and the database write into their own units, so changing the report layout can never break the payroll math.
- Keep the business decision above the mechanism, with the rule calling down into the SQL or HTTP rather than embedding it.
- Don't let one `Payroll` class compute pay, format the HTML payslip, and run the SQL — that is three reasons to change living in one place.
- Suspect a broken responsibility when you need the word "and" to describe the unit, when a business rule sits on the line next to a raw SQL string, or when the same file keeps appearing in unrelated tickets.
- Remember SRP is about reasons to change, not size — a cohesive 300-line class beats five shallow ones.

### Separation of concerns / orthogonality
**Keep unrelated concerns in independent units so a change to one doesn't ripple into another; changing the database should not touch the UI, and changing one business rule should touch one module.**
- Keep business rules in pure functions and push IO to the edges, so the rules can change without touching transport code.
- Route persistence through a boundary so swapping the data store leaves the rules untouched.
- Don't put SQL strings in the view or pricing rules inside a React component.
- Don't let a single rule change force edits across six modules.
- Ask "if I change requirement X, how many modules change?" — the answer should be one.
- Suspect tangled concerns when one conceptual change spreads across many files, or when a `utils` grab-bag is imported almost everywhere.

### Cohesion by feature — colocate what changes together
**Organize code by feature or domain capability, not by technical role; everything that changes together for one feature lives together.**
- Put the checkout handler, service, model, types, and tests together in a `checkout/` module, beside sibling `billing/` and `shipping/` modules.
- Structure the tree so you could remove a feature by deleting a single folder.
- Don't spread one feature across top-level `controllers/`, `services/`, `models/`, and `utils/`, so a checkout change has to touch four distant folders.
- Ask "can I find everything about feature X in one place, and could I delete it by deleting one folder?"
- Suspect low cohesion when adding one feature edits many sibling "layer" folders, or when a `models`/`services` directory grows without bound.
- Where a framework imposes a role-based layout (Rails), still group by feature within its conventions as far as it allows.

### Open/Closed, Liskov & Interface Segregation — extension and interface discipline
**Add new variants by adding code rather than editing what already works, keep every subtype usable in place of its base with no weakened guarantees, and let each client depend only on the methods it actually uses.**
- Register a new payment type as a new handler in a map, or a new class implementing `PaymentMethod`, leaving the branches that already work untouched.
- Don't reopen and edit the same `switch (type)` on every new variant, and don't build that extension point until a second variant actually exists.
- Model a read-only collection as its own type instead of subclassing a mutable list and disabling its write methods.
- Don't make `Square` extend `Rectangle` with a `setWidth` that secretly changes the height, and don't override a method just to throw "not supported."
- Give a handler only the `Clock` it needs instead of the whole `Services` container, and split a fat `Stream` into `Reader` and `Writer` so read-only callers don't depend on writing.
- Don't define one `Repository` with thirty methods when each caller uses two of them.
- Suspect a violation when a subclass override throws "not supported" or does nothing, when caller code checks `instanceof` to stay correct, when implementers stub methods that don't apply, or when a type-switch grows a new arm on most features.
- In structurally-typed languages (Python, TS, Go), depend on the narrowest protocol or shape you actually use as the parameter type, not a wide concrete class.

### DIP / IoC — depend on a substitutable seam
**Depend on a seam you can substitute — an abstraction owned by the higher-level policy — not on a volatile concrete detail; business logic must not reach directly into a framework, ORM, network, filesystem, or vendor SDK.**
- Pass a `PaymentGateway` into `OrderService` and let `StripeGateway` implement it, wiring the concrete one at the composition root.
- Type the dependency as a `Protocol` in Python or an interface in Kotlin, and pass a fake implementation in tests.
- Inject the clock or repository so the same logic runs against a fake in a test and the real thing in production.
- Don't call `new StripeClient()` inside a domain method, import `stripe` at the top of a business module, or query the ORM's models directly inside a use case.
- Ask "can I exercise this logic with no database, network, or framework running?" — if not, add a seam.
- Suspect leaked control when a domain module imports a framework, ORM, HTTP client, or vendor package, or when logic reaches a global singleton for its collaborators.
- Note that in Python/TS, directly instantiating a plain imported class is idiomatic — the smell is an unsubstitutable dependency on IO, a vendor, or a framework inside business logic, not instantiation itself; in Java/Kotlin/C#/Angular, a class that `new`s its own collaborators is the smell.

### Deep modules / information hiding
**Prefer modules with a simple interface hiding substantial implementation; a module should give the caller more capability than the understanding it demands.**
- Expose `parse(url) -> Request` that hides all the parsing steps, so callers never see the internals.
- Put a `Cache` behind `get` and `put` and hide eviction, sizing, and expiry inside it.
- Don't create shallow wrappers that only forward their arguments to another method and add nothing.
- Don't leak a class's internals through a dozen getters and setters that let callers reassemble the logic outside it.
- Don't shatter one coherent operation into several tiny classes the caller has to wire together.
- Treat a long function as only a hint to look closer, never by itself a reason to split a cohesive one; decompose for one-reason-to-change and readability, not to make units small.
- Suspect classitis when doing one thing requires understanding five classes, or when an interface is as complex to learn as the implementation behind it.

### Composition over inheritance
**Compose behavior from small parts; inherit only for genuine subtyping, never merely to reuse code.**
- Inject a strategy (a `Sorter`) instead of branching on type inside the implementation.
- Compose capabilities with struct embedding in Go, hooks in React, or small collaborators you pass in.
- Don't build a four-level class hierarchy just to share one helper method.
- Don't reach for a base class as a way to reuse code between otherwise unrelated types.
- Suspect an inheritance problem when understanding one class means reading the whole chain, or when base classes carry protected members tuned differently by each subclass.
- Follow the framework where it is inheritance-based (UI widgets, ORM base classes); this rule targets your own domain code.

### Stable dependencies — root vs leaf
**The more code depends on a unit (high fan-in — a "root"), the more stable, explicit, and carefully named it must be; leaf units nothing depends on may be simpler and rougher, and dependencies should point toward stability.**
- Give a widely-imported `Money` type a precise name, a small stable surface, and explicit invariants, and change it deliberately with every caller updated.
- Refactor a leaf helper freely, since nothing else rides on its shape.
- Don't casually churn the signature of a module half the codebase imports.
- Don't let a foundational type carry a vague name (`data`, `info`, `helper`) or leak its implementation, and don't make a stable core depend on a volatile leaf.
- Spot roots by counting inbound dependents, and invest cleanliness in proportion to fan-in.
- Suspect a fragile root when a one-line edit to a "core" file breaks many modules and tests, or when a heavily-imported module changes on almost every feature.

---

## B. Workflow / discipline rules

### 1. Clarify before implementing — hard gate
- If more than one reasonable interpretation of the request exists, or a decision is expensive to reverse, stop and ask before writing code; ask tight questions that each eliminate a whole branch of work, give a recommended default, and offer a fast-path ("reply `defaults` to accept all").
- Read the code and config first, and don't ask what a quick grep would answer.
- Do not assume the project has live users and needs backward compatibility — if the docs don't say so, ask whether it's greenfield; guarding compatibility on a greenfield project keeps dead constraints alive and blocks a clean design.
- If the user's answers raise new ambiguities, ask another round — iterative clarification is correct, not a failure.
- After answers, restate the requirement in 1–3 sentences, then implement.
- **A decision made silently is a defect** — if you must proceed on an assumption, state it explicitly and flag it, never bury it.
- Don't over-ask: prefer yes/no or multiple-choice over open-ended, and skip trivia a reading would settle.

### 2. Don't change what you don't understand
- Before editing existing code, be able to say what it does and who depends on it; if you can't, investigate or ask rather than guess.
- Keep public signatures/contracts stable unless changing them is the task; if you must change one, find and update every caller.
- Change behavior **or** restructure in a commit, never both — a mixed commit hides which change broke things.
- **When fixing a bug with a test, write the test first and run it *before* the fix to confirm it fails for the right reason (red); only then apply the fix and confirm it passes (green).** Never investigate, guess the cause, apply a fix, and *then* write a test that happens to pass — a test that never went red proves nothing, and your guess may have been wrong.
- Run the build/tests after each change; a green→red means revert and retry smaller, not debug in place.
- Never delete or rewrite code you don't understand just to make an error disappear — that is exactly how untouched behavior breaks.

### 3. Leave it cleaner — but don't spread the mess (scout rule)
- Match the **better** existing pattern, not the worst one nearby; copying a local bad pattern "to stay consistent" spreads debt.
- Make the file you touch slightly cleaner than you found it (a name, a dead line), within the task's scope, without sprawling into unrelated refactors.
- Never leave an untracked TODO/FIXME: either fix it now, or leave a TODO that references a real ticket (`// TODO(PROJ-123): …`).
- Delete commented-out code rather than leaving it, where the codebase's review norms agree.

### 4. Don't over-engineer; clean up after yourself
- Delete the dead code you create or orphan: unused parameters, unreachable branches, functions no longer called, leftover imports, and scaffolding.
- Right-size the generalization: design for the futures you can name and defend, but don't build speculative machinery for cases you can't (see Step back — see the bigger picture).
- When you replace a code path, remove the old one in the same change instead of leaving both.

---

## C. Micro-conventions (yield to stack idiom)

- Use one verb for one action across the codebase — pick `fetch` (or `get`, or `retrieve`) and don't mix `fetchUser`, `getAccount`, and `retrieveOrder` for the same kind of read.
- Split a behavior-selecting boolean into two named functions, preferring `renderForPrint()` and `renderForScreen()` over `render(isPrint)` when the flag picks a mode.
- Bundle arguments that always travel together into a parameter object or struct instead of threading five or six positional parameters through many calls.
- Name a unit for what it means in the domain (`PriceQuote`, `RetryPolicy`), and don't name a core concept `Manager`, `Helper`, `Processor`, `Util`, `Data`, or `Info` — a concept you can't name is a signal the model is wrong; keep framework-required suffixes like `XxxController` where expected.

---

## D. Definition of Done — self-check before declaring the task done

| Check | If No → Action |
|---|---|
| Requirement confirmed, no silent assumptions (incl. greenfield vs live users)? | State assumptions or ask now |
| Every change follows the stack's idioms and this codebase's conventions? | Align to the stack |
| Core logic I added/changed is testable with no DB/network/framework? | Add a seam or justify |
| For a bug fix: did the test fail (red) before the fix and pass after? | Run it pre-fix to prove it reproduces |
| Knowledge single-sourced — no duplicated business rule, no blind copy-paste? | Extract to one home and reuse |
| Have I named the most probable next change or feature, and would it slot in without a rewrite? | Reshape the design so the likely future fits |
| To make that probable change, how many places must be touched — is it one? | Consolidate until the knowledge has one home (DRY) |
| Do the parts that change together sit close (one module/folder), or scattered across the tree? | Colocate them by feature (Cohesion) |
| Is this only a fix for the immediate task, or am I confident it's a good move for the bigger picture? | Step back; justify it against the real goal, or redesign |
| Can I defend every abstraction I introduced from the domain, out loud? | Drop the ones I can't justify |
| Does the task look like a workaround for an earlier wrong decision (XY problem)? | Surface it; check whether revisiting A is the real fix |
| Would the next engineer find and safely change this behavior quickly? | Improve naming, placement, and cohesion until yes |
| Did I inline conceptual steps that a named helper would make readable? | Extract for readability |
| Any dead code, commented-out code, or untracked TODO left? | Delete / ticket it |
| Behavior and structure changed in the same commit? | Split them |
| Did I touch a high-fan-in (root) unit? | Re-check its name, signature stability, invariants, and all callers |
| Build and tests pass? | Fix or revert — don't leave it red |
| Any behavior outside the task's scope changed? | Revert, or justify and ask for approval |

---

## Coverage of the recurring failure modes

1. Silent wrong assumptions → **B.1 Clarify before implementing**
2. Creeping tech debt / broken windows → **B.3 Scout rule**
3. Overengineering + dead code → **Step back — see the bigger picture** (its over-engineering cautions) and **B.4**
4. Unintended side effects → **B.2 Don't change what you don't understand**
5. Root vs leaf stability → **Stable dependencies**
6. Cohesion by feature → **Cohesion by feature**
7. IoC / god classes → **DIP / IoC**
