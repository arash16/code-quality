# code-quality

An opinionated skill pack that makes AI coding agents treat code quality as non-negotiable — clean code, sound architecture, safe refactoring, and production resilience — across any language or framework.

The skills **command**, they don't lecture. An agent already knows what SRP or DRY mean; the failure mode is not applying them under task pressure, so debt accretes one tolerated shortcut at a time. Here every principle is a hard rule with concrete, multi-language examples and the smells that signal it slipping — no book summaries, no forced document artifacts, no rigid cross-skill workflows. Each skill closes with a `Check | If No → Action` Definition-of-Done gate so the agent audits its own work instead of hand-waving.

An agent always sees each skill's one-line description and reads the full skill when it judges it relevant to the task. `clean-dev` is written broadly, so it applies to essentially any coding work; the other three have focused descriptions that surface in their specific situations.

| Skill | Relevant when | What it does |
|---|---|---|
| [`clean-dev`](skills/clean-dev/SKILL.md) | any coding task | Ten design commandments — SOLID, DRY (+ don't reinvent the wheel), defensible generalization from the bigger picture, cohesion-by-feature, deep modules, dependency inversion, root-vs-leaf stability — plus workflow gates: clarify before coding, scout rule, and changing code without breaking untouched behavior |
| [`review-and-refactor`](skills/review-and-refactor/SKILL.md) | reviewing or refactoring existing code | Scan a codebase against an explicit issue catalog, prioritize by dependency structure + churn, and execute an ordered plan (dead code → safety net → refactor → boundaries) |
| [`architect-new`](skills/architect-new/SKILL.md) | designing a new system or feature | Design boundaries → dependency direction → domain model → data/consistency; only expensive-to-reverse decisions, modular monolith by default |
| [`resilience`](skills/resilience/SKILL.md) | hardening, sizing, or releasing to production | Capacity sizing, integration-point fault tolerance (timeouts, breakers, bulkheads, retries), and safe zero-downtime releases |

## First rule of all four: ecosystem precedence

The project's language / framework / idiom best practices **override** any rule in these skills. A principle's *intent* is universal; its *mechanism and smells* are often language-specific, so context-dependent items are scoped ("where applicable"). When a rule would fight the stack, the stack wins.

## Install

Works with any Agent Skills-compatible agent (Claude Code, Cursor, and others):

```sh
npx skills add arash16/code-quality
```

That installs all four skills. From then on the agent reads each one when its description matches the work at hand — `clean-dev` on essentially any coding task, the others in their specific situations. You can also point the agent at a skill explicitly when you want it.

## How the skills are written

- **Rules, not slogans.** Each principle is a hard rule followed by concrete, multi-language example sentences — the move to make, the one to avoid, and the smells that betray a violation before it spreads.
- **Opinionated by observed failure mode.** Rules are written against what AI agents actually get wrong. It pushes *defensible generalization* — because agents under-design by default, so a plain "keep it simple" just licenses laziness; it demands *don't reinvent the wheel* (prefer tested libraries over homegrown code); and it enforces a **red-before-green** test on every bug fix (see the bug reproduce before it's fixed).
- **Non-conflicting by design.** The known tensions are resolved once, in the open, so no two rules fight: function size vs. deep modules (line count is only a smell), DRY vs. coincidental duplication (dedupe *knowledge*, not text), defensible generalization vs. over-engineering (design for futures you can name and defend; keep heavyweight swap-seams for real volatility or third-party edges), and clarify vs. over-ask.
- **Self-checking.** Every skill ends with a `Check | If No → Action` Definition-of-Done table — including bigger-picture questions (what's the probable next change? how many places must it touch? is this only for the immediate task?) — that the agent must pass before calling the work done.
