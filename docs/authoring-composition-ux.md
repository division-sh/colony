# Authoring & composition UX: type-directed flows, no hand-YAML

> Status: thinking doc for iteration. Companion to [vision-extensible-runtime.md](./vision-extensible-runtime.md) (*why*) and [roadmap.md](./roadmap.md) (*sequence*).
>
> **v3** — reworked after a deep review. The one-line reframe: *structure is Swarm's home turf; the end-game is won in the **time** dimension* — iteration, deadlines, day-30, failure, waiting, regret. v2 rendered structure (contracts, types, capabilities) well and time almost not at all — and omitting time from the authoring UX **hides the moat** (durable replay is exactly the time dimension). v3 adds: a fourth operation (**diagnose**), the **policy sheet** (routing generalized to typed rows — carries time/joins/loops/schedules), **stages**, the **session/hub** as a system, `describe --prove`, failure frames, entity-first, ceremony-by-effect-class.

---

## 0. The core promise + the two insights that shape it
> **Swarm turns business intent into a verified, inspectable, replayable contract through guided type-directed assembly.** And: **generate the contract; don't hide it.**

Two insights govern v3:
1. **The end-game is the time dimension.** wire/route/bind answer "how do pieces connect." The hard, valuable part is *"what happens while nothing is happening?"* — deadlines, joins, loops, schedules, the entity's life. That's durable-runtime territory: Swarm's strongest, competitors' weakest. If it's not in the authoring UX, the moat is invisible.
2. **On a fail-closed platform, the *failure frame* is the product.** `verify ✗` will be frequent by design. The wizard's error frame (doctor-format, refuse-and-offer) is the first impression, not an edge case.

## 1. Four assembly operations (not two)
Composition is *typed choices*, never graph-drawing. Four operations over a typed substrate:

| Operation | User question | Generated artifact | Lives in |
|---|---|---|---|
| **Wire** | "Which flows/events connect?" | pin bindings | contract |
| **Policy sheet** (routing++) | "What happens, and *when*?" | typed ordered rows (below) | contract |
| **Bind** | "What may this touch, under whose authority?" | policy/budget/capability (design-time) + accounts/secrets (deploy-time) | contract **+** environment |
| **Diagnose** | "It ran and did the wrong thing — now what?" | trace→explain→edit→**fork**→regression | *reads* contract + trace |

**Diagnose is the crown jewel and it's cheap** — it surfaces *shipped* primitives (replay, **fork**, trace, spend_ledger), not new mechanism. Every trace row carries the **stable ID** of the contract element that produced it, so `swarm explain trace:8` answers *"→ general-draft because route row 6 (else) matched; rows 1–5 didn't because …"* and offers to edit with the failing case pre-loaded as a route-test fixture. Then **fork** the same fixture against the edited draft and diff outcomes. (Precision: **replay** = redelivery; **fork** = counterfactual re-execution — the iteration primitive here is *fork*, not replay.)

**Two non-obvious rules:**
- **Every interactive step is sugar over a non-interactive flag surface** (`swarm compose route add ticket.classified --when 'sentiment==angry' --to escalate --priority 2 --yes --preview`). Payoffs: scriptability/CI; the round-trip guarantee gets real (interactive layer uses the *same* stable mutation API as hand edits); and the Stage-6 assembly agent becomes a **consumer** of the flag surface, not a new mechanism. *Agent-consumable from day one; the agent itself is still last.*
- **One authoring namespace, not six top-level verbs.** `swarm compose …` (or `create` enters a hub where route/bind/wire/diagnose are actions) — a clean authoring-vs-operating boundary; don't collide with the operator CLI (`agent`, `test`).

## 2. The policy sheet — one mental model for the time dimension
Stress-tested against real workflows (AP invoice, onboarding, fulfillment, underwriting, moderation, dunning), v2's wire+route surface covers ~40–50% — the *reactive-pipeline half*. Everything with a **deadline, join, loop, or calendar** falls out (i.e. most of the economy's paperwork). The fix is **one move, not five subsystems**: generalize the ordered route table into a **policy sheet** — rows gain *types*, all ordered-first-match, all statically verifiable:

```
1. needs_human == true                   → escalate            (condition)
2. after 48h without approval.responded  → escalate-to-manager (temporal)
3. for each invoice.line_items           → check-inventory     (collection / fan-out)
4. when all checks complete              → reserve-stock       (join)
5. when 5d elapse, checks missing        → partial-hold        (join-timeout)
6. loop → draft (max 3, then escalate)                         (loop, capped)
7. every day 09:00 where stage==overdue  → dunning-step        (schedule)
```

One grammar carries conditions, time, collections, joins, loops, schedules; `verify` checks **exhaustiveness, overlap, cap-on-every-loop, join-completeness**. Add one verb — **`stages`** (author the entity lifecycle explicitly) — and **compensation** becomes a stage-indexed sheet on a cancel event. With the policy sheet + stages, **~95% of real workflows compose without YAML**; the residual 5% is legitimately code-shaped (compute packs), which the #1663 ladder assigns to code on purpose.

**The guardrail that keeps this from being a bad DSL-in-a-table:** the row-type vocabulary is a **closed, statically-verifiable set** — each type has its own `verify` check (a temporal row proves the timer is bounded; a loop row proves a cap + human escape; a join row proves completeness across all outcomes). The moment a workflow needs something outside the closed set, it **graduates to a compute pack** — the exact "enumerable → declarative, else code" line. Policy sheet ≠ expression language; it's a fixed set of analyzable row types.

**Scaling caveat:** 30-row sheets and 8-stage lifecycles outgrow wizard screens — **`describe --graph` must exist by M5** or complex flows become write-only.

*(The reviewer's full mortgage-underwriting worked example — entity-first, marked inference, stages, checklist accumulation, parallel join with no silent hole, capped loop with escape, tiered dual-approval with distinct humans, a real `verify ✗`, the hub, the diagnose loop, and `describe --prove` receipts — is the canonical demonstration of the time dimension; it's the strongest single artifact and a better "complex flow" showcase than support-triage.)*

## 3. The session is a system, not a sequence of screens
- **Draft contract written immediately** — `create` emits real, gitable, verify-able files up front (`wrote contracts/support-triage/ (5 files)` shown at create-time grounds the magic in inspectable reality — exactly what wins the skeptical engineer).
- **Interruption always leaves a resumable state** (`swarm compose --resume`). The composer honors the same durability doctrine as the runtime it configures.
- **A hub, not a corridor.** Real authoring bounces (mid-route you realize the classifier needs a 4th enum). A living draft overview with per-element state:
  ```
  support-triage (draft · 5 files · stage-machine ✓)
    classifier ✓   routes ⚠ 1 overlap   bindings ✗ 2 unbound   mock run: never
  Next: resolve the route overlap  →  swarm compose route ...
  ```
  Every sub-editor returns to the hub; `--resume` drops you at the hub; the hub renders "what's left" as a checklist.
- **Failure frames are first-class.** Show `verify ✗` in the loop, doctor-format, with refuse-and-offer where the fix is mechanical:
  ```
  verify ✗  [BLOCKER] route/not_exhaustive: sentiment=='neutral' matches no row
    Add "else → general-draft"?  [Y add / e edit / n leave broken]
  ```
  Prompt grammar: `[Y / e edit / x explain / n]` — marked default, a one-line preview of what `Y` writes, and `explain` **re-offers** the choice (never dead-ends).
- **`swarm undo`** (auto-checkpoint per apply) — regret is a first-class path for non-git-fluent users.
- **Lead with the no-NL path:** empty `swarm compose` opens the template gallery (Colony's index) → parameterize → NL refines. *The empty invocation should be the best invocation.*

## 4. Walkthrough highlights (support-triage — the reactive wedge)
Only the beats v3 changes; the rest is v2.

**Beat 0 — ask the one question only the user can answer (entity identity).**
```
Before I propose a shape — one question I can't infer:
  What's the thing being processed? (each gets its own state, history, budget)
  1. a ticket   ← best guess from "support"    2. an email    3. a customer
> 1   ✔ entity: ticket
```
*Entity identity determines isolation, budget granularity, the replay unit — the one decision with no safe default, and the one a business user is perfectly qualified to make. It does not belong in a parenthesis.*

**Beat 1 — proposal marks what the system *chose*, and shows the files.**
```
  ⚡ trigger  communication.email.received  ← inferred from "email" (alts: outlook, sendgrid-inbound)
  ...
  Wrote contracts/support-triage/ (5 files — yours, gitable)
Wrong from the first frame?  swarm compose --refine "no, internal IT tickets, no CRM"
```
*Inferred choices are visually distinct from asked-for ones; a fundamentally-wrong proposal gets a cheap conversational re-propose (`--refine`), not piecewise editing of the wrong shape.*

**Beat 6 — two consents, ceremony scaled to blast radius.** Split the one `Approve? [y]` into **contract bind** ("goes into your PR") and **environment bind** ("yours, local, never committed"), and scale ceremony by **effect class**: `read_only` auto-approves with a notice; `idempotent_write` a standard prompt; `non_idempotent_write` (send, charge) full ceremony — capability surface shown, type-the-flow-name to grant. *Approval fatigue turns review into theater; ceremony ∝ blast radius keeps the grant that matters meaningful.*

**Beat 7 — mock run, then the diagnose loop.**
```
$ swarm test --mock fixtures/standard_small.json
✓ complete — but it went to underwriter-queue. Expected auto-recommend?
$ swarm explain trace:8         # trace-row-shaped, not command-shaped
  → matched row 4 (else); row 2 requires tier==low, this is 'standard'. Edit row 2? [e]
> e   2. tier in [low, standard] AND amount<250k → auto   → verify ✓ (undo: swarm compose undo)
$ swarm run fork last --against draft    # counterfactual, NOT replay
  ✓ same fixture, edited table → auto-recommend · diff: 1 route, 2 fewer events
$ swarm test --save standard_small_autorecommends    # locked as regression ✓
```
*This closes wizard → run → trace → wizard. It's where the replay moat cashes out — and it's mostly surfacing shipped primitives.*

**Beat 8 — the receipts (`describe --prove`): the moat as UX.**
```
$ swarm describe --prove
  CANNOT send without approval
    └ enforced: send activity gated on approval.approved · verified at boot · violation unrepresentable
  CANNOT call undeclared tools
    └ enforced: sandbox injects only declared tools ✓
```
*Every AI tool promises "CANNOT X" as prose. Only Swarm can tie each CANNOT to its enforcement mechanism. Claims → receipts. The highest-leverage trust moment, nearly free.*

## 5. Composition (Part 2/3 — condensed from v2)
Publishing derives typed pins (#1468) → the flow becomes a pack. `swarm wire` proposes only type-compatible connections; cross-ontology gaps insert a **mapper pack** (pure-compute + tests + lossiness warnings) with the field mapping shown. `swarm plan --pr` output is designed to be a **PR body** (capability diff + policy sheet + mapper tests) — team review rides existing git muscle memory. **`swarm describe` shows provenance** for a stranger's pack (publisher, signature, what changed since you approved). Publish enforces semver/compatibility; upgrade shows a **capability diff** requiring re-approval.

**Ontology governance (say it, or the problem returns one level up):** one Colony-curated ontology per domain; same semver/compat rules as events; cross-major requires a mapper pack. Without this, authors mint near-duplicate ontologies and wiring-incompatibility comes back wearing a nicer hat.

## 6. Doctrines
- **Dumb router, helpfully.** Refuse-and-offer, generalized beyond routing: *"'angry' isn't a signal yet — I'll add a classifier that emits `sentiment`, then route"*; *"send without approval → I'll add the approval gate."* Agents interpret · routers/sheets dispatch · activities perform I/O · compute transforms · contracts own the effect surface.
- **`verify` is structural, not behavioral.** Generated flows ship **eval/test scaffolds** (agent evals, route tests, scenario tests) *and* **typed synthetic fixtures generated from the event schemas** — otherwise the 60-second mock magic only works for the demo template, not the novel flow the user just built.
- **`explain` is trace-row-shaped**, not just command-shaped (see Beat 7).

## 7. Day-30 (the doc must render *time*, not just minute one)
- **The spend loop.** Budgets are a bind dimension; add the *"why did cost double this week?"* journey — `swarm spend` drill-down flow → agent → entity (from `spend_ledger`), same `explain` treatment. Cost anxiety is the #1 recurring emotion of operating LLM systems; the end-game owns it, not just caps it.
- **`swarm rename`** — contract-wide refactor over stable IDs (wizard-named things get renamed; without this, names calcify = wizard debt).
- **Credential rotation / re-bind** as a first-class journey (day-30 is an expired OAuth token + a red incident, not create-time).
- **Async texture** — evals run minutes of LLM calls; the hub shows evals/scenario runs as background jobs so the loop doesn't block on its slowest member.

## 8. What it rests on + roadmap placement
Leverage points (over-invest): **typed pins (#1468)** · **the policy sheet / switch-lookup-threshold helpers (#1668)** · **ontology packs**. Sequencing (see [roadmap.md](./roadmap.md)):
- **M4 (wedge) gets the cheap, high-ROI *quality*** — the diagnose loop, entity-first, `describe --prove`, ceremony-by-effect-class, failure frames, the session/hub. These mostly *surface shipped primitives*. Plus **#1712 (CLI output audit) is a hard M4/Stage-1 prerequisite** alongside #1655/#1657: mock-mode-needs-Docker, trace-hides-business-events, and raw-Go-OAuth-errors each shatter the 60-second wedge.
- **The policy sheet (time/join/loop/schedule) + `stages` is the M4→M5 coverage bridge** — reactive-pipeline (~50%) → real business processes (~95%). This is the larger build; don't let it block the wedge.
- **`describe --graph` by M5** (complex sheets outgrow screens).

## 9. Honest hard parts
1. Assembly agent unsolved → guided composer first; NL is the improving layer; `verify` guarantees valid-or-rejected.
2. Mappers are core infra (enum translation, lossiness, tests) — first-class packs.
3. Semantic ≠ structural — the capability-surface review is where a human confirms intent types can't guarantee.
4. Policy sheet must stay a **closed, verifiable row-type set** — else it's the middle-language tarpit (§2).
5. **Progressive vocabulary is release-blocking:** Beat 1 currently forces ~16 platform concepts. Rule: *no screen introduces more than two new nouns*; default register is business language ("works with any email provider," not "consumes an ontology event"; "safe to retry," not "idempotent"), platform terms one level down in `describe`/`explain`. Plus a **friction budget** (time-to-first-mock-run ≤ X min, zero credentials, ≤ N choices — measured, release-blocking) and a **glyph/NO_COLOR fallback** from day one.
6. The real trap: "no hand-YAML" must not become "no one understands what got generated." The diff, the receipts (`--prove`), `explain`, and the tests are what keep the easy UX from being complexity hidden in worse lighting.
