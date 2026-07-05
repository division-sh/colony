# Roadmap: working backwards to guided, type-directed, capability-bounded agentic flows

> Status: living roadmap. Companion to [vision-extensible-runtime.md](./vision-extensible-runtime.md) (*why*) and [authoring-composition-ux.md](./authoring-composition-ux.md) (*the goal UX*). This is *the sequence to get there*, defined by **proofs** (what you can demonstrate), not by "build X."
>
> Tracking: each milestone is a **GitHub Milestone** on `division-sh/swarm` (auto-progress from assigned issues); this doc holds the **gate** (definition of done) for each. Milestone gate not met ⇒ milestone not done, regardless of issue count.

## The spine
> M1 proves the runtime on a real flow · M2 makes composition type-checkable · M4 proves the wedge · M6 adds NL only after the substrate is safe.

Two rules govern the whole sequence:
1. **Type-checkable before the agent** — the assembly agent (M6) drives verified typed operations; it is never the thing holding the system together.
2. **Do not let the full marketplace/provider engine delay the first wedge (M4).** Build *enough* pack structure to prove the no-hand-YAML path, then deepen.

Fastest path to market proof: **M1 + M2 + M4** on a minimal pack envelope. Everything else deepens from there.

---

## M0 — Foundation *(largely shipped — the moat)*
Durable/replayable/deterministic runtime, entities, verify, replay/fork, scenario runner (#1252/#1655), onboarding/context/gateway (#1578 cluster), grouped CLI (#1657), Colony + manifest hygiene (#1659/#1660), durable activities (#1666).

**Gate — activities must be *boring*, not just activity-shaped:**
- [ ] activity intent committed transactionally · executes outside the entity lock/DB txn
- [ ] result journaled · typed success/failure event emitted
- [ ] effect class required · idempotency key supported · retry policy enforced
- [ ] replay reuses recorded result · fork policy explicit
- [ ] trace shows request/attempt/result
- [ ] **one hand-authored activity flow survives crash/replay/fork with no duplicate external effect**

*If any activity semantic is missing, it does not drift into M4 — M4 depends on activities feeling boring.*

## M1 — Reference agentic flow *(do now, parallel with M2)*
Hand-author **support-triage** end-to-end (over twitter-prospecting — it proves *agentic business process*, the larger claim; keep twitter-prospecting as the M3b provider stress-test).

**Gate:** inbound email → classify → typed-signal route → draft (3 paths, one KB-tool-constrained) → human approve → durable send → CRM log → trace → replay.
Becomes: flagship template · composer target · scenario fixtures · Colony seed · docs spine.

## M2 — Typed composition substrate *(THE LINCHPIN)*
Typed pins (#1468) + declarative routing helpers (#1668).

**Gate:** two flows wire by type-compatibility; bad wiring fails `verify`; route tables are exhaustive; overlap/priority is visible (ordered first-match). *Everything M3–M7 depends on this — prioritize above WASM, provider engine, and the agent.*

## M3a — Minimal pack envelope
Just enough pack structure to give the composer real ingredients: manifest, exports/requires, capability declaration, version + platform compatibility, basic Colony registration, first-party install.

**Gate:** first-party packs install, declare capabilities, expose pins/tools/activities, and appear in composer search. *(Do not turn this into a grand pack taxonomy — the first user doesn't care that it's elegant.)*

## M3.5 — Contract mutation engine *(unsexy, therefore critical)*
`plan` / `diff` / `apply`, stable generated IDs, minimal patches, round-trip preservation, verify-after-every-mutation, explainable changes.

**Gate:** guided commands produce **minimal, repeatable** contract diffs; re-running the same command yields no meaningless diff. *This is what makes "generate the contract, don't hide it" real — unstable/unreadable generated YAML = "we hide the YAML badly."*

## M4 — Guided composer wedge *(THE MARKET PROOF — keep it narrow)*
No-hand-YAML path for one **reactive-pipeline** flow (support-triage — ~50% of real workflows; the *time* dimensions come in M5), ~5 first-party packs (email, approval, kb.search, CRM, send), mock connectors first, **thin binding** second. Includes the cheap, high-ROI UX quality that *surfaces shipped primitives* — the **diagnose loop** (trace→explain→edit→**fork**→regression, using replay/fork/trace/spend_ledger), **entity-first** question, **`describe --prove`** (claims→receipts), **ceremony scaled to effect class**, **failure frames** (`verify ✗`, doctor-format, refuse-and-offer), and the **session/hub** model (draft-immediately, resumable). Not a general marketplace composer.

**Hard Stage-1 prerequisite — #1712 (CLI output audit):** three findings each shatter the 60-second wedge and belong alongside #1655/#1657: mock boots require Docker (dies at `runtime_context`), `trace` hides business events behind `platform.runtime_log` noise, OAuth/connection errors are raw Go dumps.

**Gate:** a new user goes guided-intent → running mock flow → inspect capability surface (`--prove`) → diagnose a wrong outcome (explain→edit→fork) → bind real credentials → running. If it ends with "now go configure secrets in a file," the wedge is fake. Plus a **friction budget** (time-to-first-mock-run ≤ target, zero credentials, ≤ N choices — measured, release-blocking) and **progressive vocabulary** (≤2 new platform nouns per screen).

## M3b — Provider pack engine *(parallel with / after M4 — must not block it)*
The provider-trigger manifest engine (#1704): closed trust primitives, raw-body/HMAC/JWKS/timestamp/dedupe, durable ack, setup activities, capability audit.

**Gate:** GitHub + Slack + Stripe re-expressed as manifests (retiring their Go), *or* deliberately classified unsupported pending a new kernel primitive. Twitter-prospecting is the stress-test.

## M5 — The time dimension + cross-author composition and trust
**The M4→M5 coverage bridge is the *policy sheet*:** generalize the ordered route table into typed, ordered, statically-verifiable rows — condition / temporal (deadlines, SLAs, escalations) / collection (fan-out) / join / join-timeout / loop (capped, with escape) / schedule — plus one verb, **`stages`** (author the entity lifecycle), and compensation as a stage-indexed cancel sheet. This takes coverage from reactive-pipeline (~50%) to **real business processes (~95%)**; the residual 5% is code-shaped (compute packs, #1663). Guardrail: the row-type set is **closed and each type has its own `verify` check** (bounded timer, loop cap, join completeness) — else it's a middle-language tarpit. **`describe --graph` lands here** (30-row sheets / 8-stage lifecycles outgrow wizard screens). Plus: ontology packs (governed — one Colony-curated ontology per domain, semver, cross-major needs a mapper), mapper packs (pure-compute + tests + lossiness warnings), upgrade capability-diffs, richer binding (data/budget/retention), and the **day-30** surfaces (`swarm spend`, `rename`, credential rotation, async job texture).

**Gate:** a flow with a *deadline, a join, a capped loop, and a lifecycle* composes without YAML and verifies (exhaustiveness + cap-on-loops + join-completeness); a stranger's flow + a first-party provider + a mapper compose safely with a visible capability surface and end-to-end replay. *(Ontology = meaning, not just field shape.)*

## M6 — Assembly agent *(the magic trick — only after the floor is concrete)*
NL intent → proposed composition, driving the same typed operations a human can (search → shape → wire → route → mapper → bind → capability surface → verify → patch).

**Gate:** wrong guesses **fail closed** through `verify`; the agent produces *plans and patches*, never mystery YAML from vibes.

## M7 — Marketplace flywheel *(quality gate, not "20 packs exist")*
Flagship pack library + publishing UX + semver/compatibility + ontology governance + third-party contribution path.

**Gate:** every flagship pack passes conformance, anti-rot CI, capability audit, docs, mock fixtures, scenario tests. An external author can publish; the importer sees a capability diff; the composed system stays replayable and auditable. *(Otherwise Colony is npm with a badge and a prayer candle.)*

---

## Funding order (if resources are tight)
1. **M1** support-triage hand-authored flagship
2. **M2** typed pins + switch/lookup/threshold  *(the linchpin)*
3. **M3.5** stable contract mutation (plan/diff/apply)
4. **M4** guided composer in mock mode  *(the market truth)*
5. **M3b** provider manifest engine
6. **M5** ontologies / mappers / cross-author trust
7. **M6** assembly agent
8. **M7** marketplace flywheel

**M1 + M2 = the engineering truth. M4 = the market truth. M6 = the magic trick, but only after the floor is concrete.**

## Red flags to watch
1. Calling **M0 "done"** before activities are fully journaled/effect-classed/idempotent.
2. Letting **M3** become a grand abstraction swamp before M4 proves the wedge.
3. Building **M6** before typed operations exist (→ a grammatical-nonsense generator).
4. **Hiding** generated contracts — "no hand-YAML" ≠ "no visible contract."
5. Treating **ontology** as merely schema (it's meaning; mappers need lossiness warnings + tests).
6. Letting **code sneak back into effects** — compute stays pure, external I/O stays in durable activities (no `logic_node`-with-tools regression).
7. **Rendering only structure, never time.** The authoring UX is strongest at structure (contracts, types, capabilities) and weakest at *time* (iteration, deadlines, day-30, failure, waiting, regret) — but time is where the durable-runtime moat lives. Omitting the diagnose loop / policy sheet / day-30 doesn't just cut coverage, it *hides the moat*. Watch that M4/M5 render time, not just shape.
