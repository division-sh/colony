# Authoring & composition UX: type-directed flows, no hand-YAML

> Status: thinking doc for iteration. Companion to [vision-extensible-runtime.md](./vision-extensible-runtime.md) — vision covers *why* (microkernel + pack marketplace); this covers *how a human actually builds something* without hand-writing a small YAML religion.
>
> **v2** — reworked after review: **binding** added as a third assembly operation; the core promise reframed away from "NL builds it"; routing given explicit ordered-first-match semantics; classifier confidence / `needs_human`; business flows decoupled from providers via ontology events; round-trip/diff editing; tiered + richer capability surface as the *containment boundary*; mock/dev mode; verify-vs-eval; prompt-injection surfacing; semver/compatibility on publish.

---

## 0. The core promise (not "NL builds it")

The foundation is **not** natural language. NL *accelerates*; it isn't the bedrock. The promise:

> **Swarm turns business intent into a verified, inspectable, replayable contract through guided, type-directed assembly.**

And the one principle that makes it work:

> **Generate the contract; don't hide it.**

You never *have* to write YAML — but a real contract is produced, versioned, diffable, verified, auditable, composable, replayable. This avoids both bad futures: *hand-authored YAML* (too expensive for first contact) and *hidden behind a wizard* (opaque — "the AI made a workflow somehow," which is vibe-coded operations). The wizard generates the contract; the contract stays the source of truth; `verify` proves it; the **capability surface** is the review artifact.

## 1. Three type-directed assembly operations

Composition reduces to three operations over a typed substrate — the user makes *typed choices*, never designs a graph from scratch:

| Operation | User question | Generated artifact | Lives in |
|---|---|---|---|
| **Wiring** | "Which flows/events connect?" | pin bindings | contract |
| **Routing** | "Which typed cases go where?" | `switch`/`threshold`/`lookup` (#1668) | contract |
| **Binding** | "What may this touch, under whose authority?" | policy/capability/budget (design-time) + credentials/accounts (deploy-time) | contract **+** environment |

**Binding splits by lifecycle** (important): *design-time* binding — policy, budgets, capability grants, approval surfaces, retention — is authored into the **contract** and versioned (a pack ships with it). *Deploy-time* binding — real OAuth accounts, secrets, tenant scope — is **per-environment**, in the secret store, *not* the contract. A pack *declares what needs binding*; the importer *binds real accounts*.

**Why binding is the trust primitive, not plumbing:** the bound capability surface **is the containment boundary for untrusted packs.** You import a stranger's flow, and it can only touch the accounts/capabilities you explicitly bound — nothing else. Binding + prompt-injection defense + capability-audit are one principle: *a pack is sandboxed to exactly what you bound it to.*

**The architectural constraints that keep all three tractable:**
- **Agents do judgment** → emit *typed signals*; "is this angry?" is never the router's job.
- **The router is deliberately dumb** → dispatches only on typed fields (doctrine — §5).
- **Compute packs** do pure computation; **activities** do durable external I/O (#1666).
- **Typed pins (#1468)** + **ontology packs** → wiring is graph search; independent authors' flows connect.
- **`verify`** → structural safety net (necessary, not sufficient — §6).

---

## 2. Part 1 — Create a non-trivial flow: `support-triage`

A real business process (not a chatbot): trigger → classify → multi-way route by type/mood/confidence → draft (3 variants, one KB-tool-constrained) → human approval → durable send + durable CRM log.

**Beat 1 — Intent → a plan (not YAML), consuming an *ontology* event (not a provider event).**
```
$ swarm create "triage inbound support email: classify it, route by type/mood,
   draft a reply, get a human to approve, then send and log to the CRM"

Proposed flow: support-triage   (entity: ticket)
  ⚡ trigger   communication.email.received@1   ← gmail@2.1 maps into support-ontology
  🧠 classify  triage-classifier    emits {category, sentiment, urgency, needs_human}
  ⑃ branch    ordered route on classification
  🧠 draft     billing | technical | general
  🙋 approve   human approval        ← approval@1.3
  ⚙ send       communication.send    (activity, idempotent) ← gmail@2.1 OR sendgrid@1.2
  ⚙ log        crm.create_note       (activity, idempotent) ← hubspot@1.0
Packs: gmail@2.1, approval@1.3, hubspot@1.0, support-ontology@1.1
Approve this shape?  [y / edit / explain]
```
*The flow consumes `communication.email.received@1` (ontology), so it's portable — Outlook / SendGrid-inbound / Zendesk map into the same event. The flow isn't accidentally Gmail-shaped.*

**Beat 2 — The classifier emits typed signals *including its own uncertainty*.**
```
$ swarm agent edit triage-classifier
  category   enum: billing | technical | complaint | other
  sentiment  enum: angry | neutral | happy
  urgency    enum: high | low
  needs_human boolean            ← the agent's OWN "I'm unsure, escalate" judgment
  rationale  string              (for the trace/audit, not for the router)
> verify ✓   (output is a declared event: ticket.classified)
```
*`needs_human` (a single boolean the agent self-assesses) is the primary uncertainty escape — cleaner than the router thresholding confidence floats. Add per-field confidence only where a route depends on it. Misclassification is not hypothetical; the router must be able to bail to a human safely.*

**Beat 3 — The router: an *ordered* decision table (dumb, first-match, overlap-aware).**
```
$ swarm route ticket.classified   (ordered: first match wins)
  1. needs_human == true     → escalate-to-human
  2. sentiment == angry      → escalate-to-human
  3. category == complaint   → escalate-to-human
  4. category == billing     → billing-draft
  5. category == technical   → technical-draft
  6. else                    → general-draft
> Overlap: (angry + billing) matches rows 2 and 4 → row 2 wins by priority.
> verify ✓  (exhaustive; overlaps resolved by explicit order)
```
*Rows are **ordered first-match**; `verify` checks exhaustiveness and **warns on overlap unless the priority is explicit** — so the user trusts the router instead of assuming it's haunted. You picked typed fields; the CLI enumerated cases; no CEL written.*

**Beat 4 — Draft agents + tools (capability-declared).** `technical-draft` gets `tools: [kb.search]`; the sandbox injects only that.

**Beat 5 — Human gate + durable activities.** On approve → `communication.send` (activity, idempotency key `ticket.id`) then `crm.create_note`. On reject → loop to draft with the human's note. *Crash after approve, before send → replay resumes without double-sending.*

**Beat 6 — Bind (design-time policy + deploy-time accounts), then review the capability surface.**
```
$ swarm bind support-triage
  DESIGN-TIME (→ contract):
    budget           max 3 LLM turns/ticket · emergency-stop enabled
    send policy      allowed ONLY after approval.approved
    retention        email body retained in entity state: yes · redaction: none
  DEPLOY-TIME (→ environment):
    communication.email  → choose account   [gmail_oauth ▸ connect]
    crm                  → choose secret     [hubspot_api_key]
    backend              → choose            [anthropic]
    approval_queue       → choose team mailbox
Approve bindings?  [y / edit / explain]
```
```
$ swarm describe support-triage          # tiered: one-line summary + drill-down
support-triage (entity: ticket)
  ▸ CAN: read email · classify (LLM, no tools) · draft (technical may kb.search) ·
         send email ONLY after approval (idempotent) · write CRM note
  ▸ CANNOT: send without approval · call undeclared tools · touch anything unbound
  ▸ NEEDS: gmail_oauth, hubspot_api_key, anthropic_api_key
  ▸ REPLAYABLE ✓  DURABLE ✓  EFFECTS-VISIBLE-IN-CONTRACT ✓
  [--data] DATA: reads email subject/body/from + CRM record; writes reply + CRM note
  [--untrusted] INJECTION SURFACE: email.body → classifier + draft prompts;
                mitigation → classifier has no tools, send requires human approval
  [--budget] 3 turns/ticket, low est. cost, emergency-stop on
```
*You approve **what it can do + what it can touch + who authorizes it**, provably, before it runs. The rich dimensions (data / injection / budget / retention) are **drill-downs**, not a 20-line wall — one-line summary by default.*

**Beat 7 — Run it *in mock mode first*, then real bindings.**
```
$ swarm serve --dev --mock-connectors          # mock gmail/crm/kb — feel value with NO OAuth
$ swarm scenario run fixtures/angry_complaint_email.json
   → classify(needs_human) → escalate → approval  (no email sent) ✓
$ swarm trace latest ; swarm run replay latest
# then, when ready:
$ swarm bind gmail --oauth ; swarm bind hubspot --oauth ; swarm serve
```
*Mock connectors = the tool-side of the shipped scenario runner (#1252/#1655) — a tool that returns fixtures. The user feels the architecture in 60 seconds instead of fighting OAuth for twenty minutes. Ships with fixtures + expected traces.*

---

## 3. Part 2 — It becomes composable: the flow gets an interface (with compat rules)
```
$ swarm publish support-triage --to colony
  interface:
    inputs:  [ communication.email.received@1 ]
    outputs: [ ticket.resolved@1, complaint.escalated@1 ]      (ontology-typed)
    requires: backend[anthropic], tools[email, kb, crm]
    ontology: support-ontology@1.1
  compatibility: ticket.resolved@1 unchanged · complaint.escalated@1 unchanged
  verify ✓ · platform_version ">=1.8" · pushed support-triage@1.0.0
```
Compatibility rules (enforced on publish/upgrade): add-optional-field = compatible; add-required / remove / rename / narrow-enum = **breaking**; widen-enum = maybe. Upgrades show a **capability diff** requiring re-approval:
```
$ swarm upgrade support-triage@1.1
  + technical-draft may now call kb.search
  + ticket.resolved adds optional resolution_summary
  no new external writes
Approve upgrade? [y / explain]
```

## 4. Part 3 — Compose into `customer-ops` (typed wiring + mappers as first-class infra)
```
$ swarm add support-triage@1.0 churn-watch@2.3 csat-survey@1.1
$ swarm plan            # preview the composition as a patch before applying
$ swarm wire
  support-triage.ticket.resolved     → csat-survey.on_resolved   ✓ exact (shared ontology)
  support-triage.complaint.escalated → churn-watch.signal        ⚠ cross-ontology
      mapper support→retention@0.4:
        customer_id → subject.customer_id
        sentiment   → severity   (angry→high, neutral→medium, happy→low)
        reason (free text) → risk.reason (enum)   ⚠ LOSSY — needs a classifier mapper
      mapper tests: ✓ angry complaint→high  ✓ missing customer_id rejected
      insert? [y]
> apply → verify ✓
```
*Mappers are not glue — they're **pure-compute packs + schema assertions + tests**, first-class Colony artifacts, with **lossiness surfaced** (free-text→enum needs a classifier, not a rename).*

Composed capability surface (`swarm describe customer-ops`) shows the whole system's effect surface and full data flow on one screen, replayable end-to-end — *including a stranger's pack, confined to what you bound.*

---

## 5. Doctrine: the router is dumb (and helpful about it)
Make it a named principle. The composer **refuses-and-offers** rather than accepting semantic logic as routing:
```
> "route angry emails to a human"
  'angry' isn't a typed signal yet. I'll add a classifier that emits `sentiment`,
  then route: sentiment == angry → human.  OK? [y]
```
The router's dumbness stays *invisible and helpful* — it auto-inserts the classifier, not a lecture. Agents interpret · routers dispatch · activities perform I/O · compute transforms · **contracts own the effect surface.** That separation is why the flow stays inspectable.

## 6. Verify is necessary, not sufficient — generated flows ship with tests/evals
`verify` proves *structure* (schemas, exhaustive routes, declared capabilities, effect classes, pin compatibility). It cannot prove *behavior* (the classifier is accurate, the mapper preserves meaning, the escalation policy matches company practice). So generated flows include **eval/test scaffolds**:
```
$ swarm agent eval triage-classifier   → category 91% · sentiment 96% · fails: case_017
$ swarm route test ticket.classified   → ✓ angry+billing→escalate  ✓ unknown→general
$ swarm scenario test                  → ✓ angry complaint escalates, does NOT send email
```
*The contract gives the skeleton; tests/evals prove the meat isn't cursed.*

## 7. Round-trip editing (no hand-YAML ≠ no visible diff)
Serious users trust the machine *because* they see the diff. The model:
- **contract = source of truth**; the CLI mutates it through a stable model and writes **minimal diffs**.
- **stable IDs** (`route_ticket_classified_by_sentiment`, not `route_1`) so re-running the composer doesn't churn the repo.
- every command supports `--preview` / `swarm plan` / `swarm diff` / `swarm apply`.
- hand-edits are respected (contract is authoritative); the composer re-reads and patches.

## 8. `explain` everywhere — the product is *confidence*, not just workflows
`swarm explain route ticket.classified`, `swarm explain capability send` — show *why* each row/branch/effect is what it is (which policy, which condition, which approval gate). Creating workflows is table stakes; creating *confidence in what got generated* is the product.

---

## 9. What it rests on + staged roadmap
Over-invest in the three leverage points: **(1) typed pins (#1468)** · **(2) switch/lookup/threshold helpers (#1668)** · **(3) ontology packs**. Then:

- **Stage 1 — one flow magical in mock mode.** Guided `create` (not full NL) + support-triage template + mock connectors + generated contract + ordered route table + tiered capability surface + trace/replay. *Assembles shipped pieces* (scenario runner #1655, context model, grouped CLI #1657) + the mock-connector piece + a few packs. **Goal: prove the UX loop.**
- **Stage 2 — typed pins + ontology-backed wiring** (exact pin matching, incompatibility diagnostics, mapper-insertion preview). **Goal: prove composition.**
- **Stage 3 — binding + capability review** (inline OAuth, secret/policy binding, capability screen, upgrade capability diff). **Goal: prove trust.**
- **Stage 4 — Colony flagship packs** (Gmail/Outlook, Slack, HubSpot/Salesforce, approval, KB, SendGrid, Stripe, GitHub, generic webhook/HTTP, support/commerce/customer ontologies). **Goal: give the assembly agent real ingredients.**
- **Stage 5 — assembly agent, last.** A *power assistant over the existing typed operations* (search → propose shape/wiring/routes/mappers/bindings → verify → explain → patch), never "LLM writes files from vibes." When it's wrong, the platform still **fails closed** instead of confidently generating grammatical nonsense.

## 10. Honest hard parts / open questions
1. **Assembly agent is unsolved** — ship guided composer first; NL is the improving layer, with `verify` guaranteeing valid-or-rejected, never broken.
2. **Mappers are core infra** — enum translation, normalization, lossiness, defaults, unit/identity resolution, version compat. First-class packs with tests, not magic glue.
3. **Semantic ≠ structural** — ontologies mitigate; the capability-surface review is where a human confirms intent types can't guarantee (so review is not optional).
4. **Ambiguity needs a chooser** — reduce "design the graph" to "pick one of two"; can't automate away choice.
5. **Prompt injection** — support email is adversarial; the containment story (no-tool classifier, approval-gated send, durable activities, bound capabilities) is the mitigation, and it must be *surfaced* in the capability review.
6. **Credential friction can kill the wedge** — mock mode first; `create` ends *running*, not "go set up secrets."
7. **Pack-library chicken-and-egg** — seed Colony's flagship packs before the `create` demo is worth showing.
8. **The real trap:** "no hand-YAML" must not become "no one understands what got generated." The diff, the capability surface, `explain`, and the tests are what keep the easy UX from becoming complexity hidden in worse lighting.
