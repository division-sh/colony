# Vision alignment: open issues → milestones (+ proposed epics for gaps)

> Status: for review. A **bounded** pass mapping the 90 open `division-sh/swarm` issues onto the [roadmap](./roadmap.md) milestones, surfacing four kinds of finding — **consolidate**, **reconcile** (same thing designed twice), **resequence** (priority the vision changes), **gap** (vision needs it, no issue exists). Discipline: reframes concentrate on the **wedge-critical band (M2–M5)**; M6/M7-era and pure runtime-hardening issues are left as-is (they'll change after the wedge, or they're orthogonal). Every reframe is *framing/sequencing guidance for the gate*, not a mandate.

## Headline findings
1. **The linchpin (M2) has no implementation issue.** Typed flow interfaces / pins is only **Discussion #1468** — the single highest-leverage thing on the critical path is undesigned as an issue. → **Epic E1.**
2. **The entire M4 authoring surface is a gap** — the guided composer, mock connectors, the diagnose loop, `describe --prove` have *no issues*; they live only in the design docs. → **Epics E2–E4.**
3. **#1668 and the policy sheet are the same grammar designed in two places** — reconcile before either ships.
4. **The #1663 ladder *is* the pack-composition primitive set** — reframe it as such so activities/compute/helpers/orchestration line up with the pack model.
5. **A cluster of onboarding/config/CLI issues are actually one thing: the M4 wedge floor** — sequence them together, gated by #1712.

---

## M0 — Foundation *(runtime moat; mostly shipped — remaining = hardening)*
**Maps here:** #1666 (activities — apply the strict activity-semantics gate), #564/#695 (replay/resume/fork semantics — the durable-replay moat; **note: fork is the M4 diagnose primitive**), #169 (watchdogs/observability — foundation *and* the M4 trace surface), #1239/#1386/#1218/#1233/#1196/#1658/#1702 (SQLite↔Postgres parity + test-health), #995 (raw Postgres boot error, no migration — *this is a teaching-error/first-run failure, tag it M4-adjacent*), #164/#170/#172/#161/#453/#456/#165 (runtime architecture hardening), #1445/#1260 (bug/lint), #746 (bash isolation — kernel security), #126/#159/#174/#94–#107 (audits/maintenance — orthogonal).

**Reframes:**
- **#165 (typed expression model, replace fallback interpretation) → resequence into M2.** "A typed model with explicit variable scopes" *is* the substrate type-directed routing/wiring needs. It's mis-filed as generic runtime hygiene; it's a linchpin dependency.
- **#995 → M0 gate item with an M4 lens.** "Boot fails with a raw Postgres error, no migration path" is exactly the raw-Go-dump failure class the wedge cannot afford (#1712 kin).
- **Gate:** enforce the strict activity checklist from the roadmap (journaled · effect-classed · idempotent · replay-reuses-result · fork explicit) before any of it is claimed done — M4 depends on activities being *boring*.

## M1 — Reference agentic flow *(hand-author support-triage / twitter)*
**Maps here:** #1661 (colony batch-classify + validate twitter-prospecting — the flagship-flow validation; blocked on the rebuild), #1526/#1532 (friction-sweep remnants surfaced by hand-authoring), #1462 (the friction-sweep parent — mostly closed).
**Reframe:** #1661 should be sequenced *after* the M1 hand-rebuild (its validation target doesn't exist yet — flagged earlier). Use it as the M1 acceptance, not a standalone.
**Gap:** none — M1 is hand-authored contracts, doable today.

## M2 — Typed composition substrate *(THE LINCHPIN)*
**Maps here:** #1668 (declarative helpers → **the policy-sheet row types**), #1463/#1464 (authoring-surface reduction / friction addendum — the "generate the contract" ease work), #1685/#1703 (agent defaults/profiles → *defaults-first, not a profiles system*), #165 (typed expressions — resequenced in), #1031 (contract-authored list predicates — typed operations), #1693 (handler.branch reconcile — routing/branch, folds into the policy sheet), #1442 (proof/routing/authoring ergonomics), #374 (typed turn-usable tool surface — typed tools for wiring).

**Reframes / reconciliations:**
- **#1668 ⟷ the policy sheet — RECONCILE.** Don't ship `switch`/`lookup`/`threshold` as three handler keywords; introduce **one ordered, typed-row construct** whose first three rows are the selection types, each with its own `verify` check, extensible by *closed* row types (temporal/join/loop/schedule land in M5), lowering to the existing rules/compute owners. Keeps the handler grammar == the wedge's routing UX.
- **#1693 (handler.branch) folds into the same construct** — it's a fourth spelling of branch selection; don't let it survive as a separate interpreter.
- **#1463/#1464/#1685/#1703 are the "type-directed authoring" cluster** — sequence them as one theme (infer the derivable, defaults-first, mapping sugar), not scattered friction fixes.

> **Gap → Epic E1 — Typed flow interfaces (pins).** Promote **Discussion #1468** to an implementation epic. This is the composition substrate: typed input/output pins (event schema + ontology ref + direction + cardinality + effect class), `verify`-checked pin compatibility, the basis for *wiring = type-directed graph search*. **Nothing in M3–M7 works without it; it is the single highest-priority create.**

## M3a — Minimal pack envelope
**Maps here:** #1661 (Colony validation), Colony scaffold + #1659/#1660 (shipped).
> **Gap → Epic E5 — Unified pack envelope + Colony as multi-type registry.** One pack shape (manifest + optional compute + optional activities + capability declaration) across trigger/connector/compute/flow/agent/ontology packs; Colony generalized from flow-only to multi-type. Enough for the composer to have real ingredients (M4).

## M3b — Provider pack engine
**Maps here:** #1651 (provider trigger adapters — parent), #1704 (Stripe adapter), #1458 (native OAuth + top-10–20 integrations — the flagship-pack + binding scoping).
**Reconcile:** **#1651/#1704 ⟷ E5.** The provider-trigger *manifest engine* must produce packs in the unified envelope (not a bespoke Go adapter registry) — and the acceptance is *re-expressing GitHub+Slack+Stripe as manifests*, else classified unsupported. #1704's refinement comment already points here; make it the gate.
**Resequence:** must **not block M4** — see roadmap. If the manifest engine slips, ship Stripe as a Go adapter behind the envelope and prove the wedge first.

## M4 — Guided composer wedge *(THE MARKET PROOF)*
**Maps here — the "wedge floor" cluster (sequence together):** #1578 (first-run umbrella), #1574 (embed spec/drop --platform-spec), #1576 (IPC transport), #1441 (profile-driven runs), #1600/#1640 (env-var/workspace-env audits), #1138/#1213 (**host backend — Docker-optional; this is the mock-mode-without-Docker prerequisite**), #1252 (mock LLM), #1649 (CLI surface / the `compose` namespace), #1445 (task-mode audit), #1562/#1564 (agent liveness/list — diagnose surface), #169/#375 (trace — diagnose surface), #1712 (**CLI output/error UX — the hard Stage-1 floor**).

**Reframes / resequences:**
- **#1712 → the three P0s are wedge blockers, above "CLI polish":** mock-needs-Docker (dies at `runtime_context`), trace-hides-business-events (behind `platform.runtime_log` noise), OAuth/connection = raw Go dumps. These each shatter the 60-second demo.
- **#1138/#1213 (host backend) → elevate as the mock-mode prerequisite.** Mock mode is only magical if it boots with *no infra*; today it needs Docker.
- **#1578 becomes the M4 acceptance owner**, consuming #1574/#1576/#1441/#1600/#1640/#1712 as children — not a parallel onboarding track.
- **#1252 covers mock *LLM* only — mock *connectors* is missing** → **Epic E3.**
- **#1649 → make it the `compose` authoring namespace**, not just grouped help (align with the "one authoring namespace" rule).

> **Gap → Epic E2 — Guided composer (type-directed authoring loop).** `swarm compose` (create/route/bind/diagnose), draft-contract-immediately + hub + `--resume`, failure frames, entity-first question, **two-consent binding**, ceremony-scaled-to-effect-class, progressive vocabulary, one namespace. Generates the contract; `verify` is the net. The flag surface is the substrate the M6 agent later consumes.
> **Gap → Epic E3 — Mock connectors + generated fixtures.** Tool-side of the scenario runner (#1655): connectors that return fixtures; generate typed synthetic scenario fixtures from event schemas so novel flows (not just the template) run in mock mode.
> **Gap → Epic E4 — The diagnose loop + provable capability surface.** Trace-to-contract linkage (every trace row carries the contract-element stable ID) → `swarm explain <trace-row>` → edit → **fork** to iterate → `swarm test --save` regression; and **`describe --prove`** (each CANNOT tied to its enforcement — the moat as UX). Mostly *surfaces shipped primitives* (replay/fork/trace/spend_ledger) — cheapest high-ROI work in the plan.

## M5 — Time dimension + cross-author composition
**Maps here:** #1669 (compute-module / pure function — **the compute-pack tier**), #1670 (function build tooling/CLI), #1671 (orchestration functions — durable journal over activities), #1672 (ladder trace/test/conformance closure), #983 (LLM provider interface — backend packs), #1458 (rich binding/OAuth), #1448 (large-emit → tool-result fan-out — connects to compute/activities), #1449 (per-role concurrency cap — R8).
**Reframe — #1663 ladder *is* the pack-primitive set:** activities (#1666) = the I/O primitive; compute-module (#1669) = compute packs; declarative helpers (#1668) = the policy sheet; orchestration (#1671) = the durable-chain source syntax. Frame the ladder tracker as *"the composition primitives packs are built from,"* so each rung lands in the pack model rather than as a standalone runtime feature.

> **Gap → Epic E6 — Policy sheet: time/join/loop/schedule + stages + compensation.** Extend E1's/#1668's ordered-row construct with the temporal/collection/join/join-timeout/loop(capped)/schedule row types + a `stages` verb (author the entity lifecycle) + compensation as a stage-indexed cancel sheet. Each row type ships its own `verify` check (bounded timer, loop cap+escape, join completeness). **This is the ~50%→~95% coverage bridge** (evidence-gated per the #1663 discipline). Plus `describe --graph`.
> **Gap → Epic E7 — Ontology packs + governance + mapper packs.** Shared event vocabularies (one Colony-curated ontology per domain, semver, cross-major needs a mapper), mapper packs (pure-compute + tests + lossiness warnings). The interop stdlib — without it, packs are islands and cross-author composition (the moat proof) can't happen.
> **Gap → Epic E8 — Day-30 operating surfaces.** `swarm spend` (spend_ledger drill-down flow→agent→entity), `swarm rename` (contract-wide refactor over stable IDs), credential rotation/re-bind, async job texture in the hub.

## M6 — Assembly agent *(deliberately left alone — build last)*
**Maps here:** nothing, correctly. The NL layer is a *consumer of E2's flag surface*, built after the wedge. Do **not** create issues yet; it will change based on M4's evidence. (Precision to hold: E2's flag surface must be agent-consumable from day one; the agent itself is M6.)

## M7 — Marketplace flywheel *(leave the reshaping until the wedge proves out)*
**Maps here:** #814/#823 (open-source readiness + Apache-2.0 license — public-release prerequisites), #977 (agent-id UUID identity — multi-bundle marketplace identity), #1026/#1028 (multi-bundle artifact/context — bundle packaging), #983 (backend packs), #735/#794 (CLI v1/v2 compliance — operator surface).
**Reframe:** #977 (identity) + #1026/#1028 (bundles) are the **marketplace-multi-tenancy** substrate; group them under an M7 umbrella when M7 activates — don't design them now.
> **Gap (M7, do NOT build yet) → future Epic:** publishing UX + semver/compatibility enforcement + capability-diff-on-upgrade + third-party contribution/trust path. Named for completeness; deferred until the wedge.

## Orthogonal — runtime hardening / audits / bugs *(no vision reframe)*
Leave as-is, prioritize on their own merits: #94–#107 (P3 audits), #365 (verify expansion), #499/#628/#630 (prompt-lint/strict-tools), #664/#777 (typed-write/delivery lifecycle), #749 (forkchat), #1031 (list predicates — though it's M2-adjacent), #1368 (event.publish route context), #1432 (record_evidence), #1696/#1699 (test-companion/trace-drain), #1658/#1702 (flakes). *These are real work; they're just not on the wedge critical path and shouldn't be re-scoped by the vision.*

---

## Proposed epics (the gaps, consolidated)
| Epic | Milestone | Fills the gap | Priority |
|---|---|---|---|
| **E1 — Typed flow interfaces (pins)** | M2 | promote Discussion #1468 → issue; the composition substrate | **P0 (linchpin)** |
| **E2 — Guided composer (authoring loop)** | M4 | create/route/bind/diagnose, hub, no hand-YAML | P1 |
| **E3 — Mock connectors + generated fixtures** | M4 | mock mode for *novel* flows, no infra | P1 |
| **E4 — Diagnose loop + `describe --prove`** | M4 | trace→explain→fork; capability receipts | P1 *(cheap — surfaces shipped primitives)* |
| **E5 — Unified pack envelope + multi-type Colony** | M3a | one pack shape; registry generalized | P1 |
| **E6 — Policy sheet: time/join/loop/schedule + stages** | M5 | ~50%→95% coverage; extends E1/#1668 | P2 *(evidence-gated)* |
| **E7 — Ontology + mapper packs + governance** | M5 | cross-author interop stdlib | P2 |
| **E8 — Day-30 operating surfaces** | M5 | spend/rename/rotation/async | P2 |

## The five moves I'd make first
1. **Create E1 (typed pins) and prioritize it** — the linchpin has no issue; everything waits on it.
2. **Reconcile #1668 ⟷ policy sheet** in its gate — one grammar, row-typed, not three keywords.
3. **Make #1578 the M4 wedge-floor owner**, pulling in #1712/#1574/#1576/#1441/#1600/#1640/#1138/#1213; treat #1712's three P0s as wedge blockers.
4. **Reframe #1663 as "the pack-composition primitives"** so activities/compute/helpers/orchestration land in the pack model.
5. **Stand up E2–E4 as the M4 authoring epics** — the wedge is currently an empty milestone (all design, no issues).

*Caveat, same as everywhere: the milestone frame is unvalidated design. Do the reframes on the wedge band; hold M6/M7 and the orthogonal cluster until the wedge (M4) returns evidence.*
