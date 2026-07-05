# Authoring & composition UX: type-directed flows, no hand-YAML

> Status: thinking doc for iteration. Companion to [vision-extensible-runtime.md](./vision-extensible-runtime.md) — the vision covers *why* (microkernel + pack marketplace); this covers *how it feels* to create a non-trivial flow and compose it into a larger system. Grounds the "ease of first agent" wedge in a concrete UX.

---

## 0. The wedge and the one principle

To win the agentic-workflow user we must match **ease of first agent** (LangChain is `pip install` + 10 lines) while keeping the moat (durable, replayable, auditable, bounded). The fear is that a YAML-authored platform is just a less-functional n8n. The resolution is one principle:

> **Generate the contract; don't hide it.**

The CLI/assembly-agent *writes* the declarative contract for you and `verify` proves it. You never *have* to write YAML — but the contract is still there, versioned, inspectable, editable, auditable. So you get LangChain's ease **and** the guarantees, because the artifact is a *verified contract*, not opaque code. Every composition command mutates the package and re-verifies; the package stays the source of truth.

## 1. What makes "no hand-YAML" actually possible

Composition reduces to **two type-directed operations**, plus a small set of architectural constraints that keep them tractable.

**The architectural constraints (why the hard parts are elsewhere):**
- **Agents do judgment** → they emit *typed signals* (`sentiment: angry|neutral|happy`). "Is this angry?" is never the router's job.
- **Compute packs do computation** → pure WASM, emits typed results.
- **Activities do external I/O** → durable, capability-declared, idempotent (#1666).
- **The router is deliberately dumb** → it only *dispatches on typed fields*. That constraint is what makes routing specifiable without code.
- **Typed pins (#1468)** → a flow's inputs/outputs are typed, ontology-referenced, directional events.
- **Ontology packs** → shared event vocabularies, so independently-authored flows connect.
- **`verify`** → the safety net; anything invalid fails closed, so the CLI/agent can generate confidently.
- **The capability surface** → the *review artifact* (what a flow can touch), not the YAML.

**The two type-directed operations:**
1. **Wiring** = connect compatible pins → *graph search over typed interfaces* (unique path auto-connects, ambiguity asks, gap inserts a mapper).
2. **Routing** = decide which branch → *a decision table over typed fields* (enum → enumerated cases, number → thresholds; the CLI generates a `switch`/`threshold` #1668, `verify` checks exhaustiveness).

Both are "pick from typed options; the tool generates + verifies." Neither is possible in an arbitrary-code tool — there's no typed field to enumerate and no exhaustiveness to check.

---

## 2. Part 1 — Create a non-trivial flow: `support-triage`

A real flow: trigger → classify → multi-way route by type/mood → draft (3 variants, one with a KB tool) → human approval → send email + log to CRM. Agents + deterministic routing + tools + human-in-loop + branching + durable I/O.

**Beat 1 — Intent → a plan (not YAML).**
```
$ swarm create "triage inbound support email: classify it, route by type and mood,
   draft a reply, get a human to approve, then send and log to the CRM"

Proposed flow: support-triage   (entity: ticket)
  ⚡ trigger   email.received       ← gmail@2.1
  🧠 classify  triage-classifier    emits {category, sentiment, urgency}
  ⑃ branch    route on classification
  🧠 draft     billing | technical | general
  🙋 approve   human approval        ← approval@1.3
  ⚙ send       send_email            ← gmail@2.1  (activity, idempotent_write)
  ⚙ log        crm.create_note       ← hubspot@1.0 (activity, idempotent_write)
Packs: gmail@2.1, approval@1.3, hubspot@1.0, support-ontology@1.1
Approve this shape?  [y / edit / explain]
```
*The review artifact is the shape + packs, not a config file. The agent got 80% in one line; you refine the 20% that's yours.*

**Beat 2 — The classifier: define the typed signals (the hinge).**
```
$ swarm agent edit triage-classifier
> Signals it emits (typed — the router dispatches on these):
    category   enum: billing | technical | complaint | other
    sentiment  enum: angry | neutral | happy
    urgency    enum: high | low
> verify ✓   (output is a declared event: ticket.classified)
```
*The agent does the judgment and emits typed fields. The router never does semantics — it dispatches on these.*

**Beat 3 — The router: a decision table from the typed signals (the dumb router).**
```
$ swarm route ticket.classified
> Route on which signal(s)?   sentiment, category
> These are enums — map each case to a branch:
    sentiment == angry    → escalate-to-human   (skip auto-draft)
    category  == complaint → escalate-to-human
    category  == billing   → billing-draft
    category  == technical → technical-draft
    else                   → general-draft
> Generating switch… verify ✓   (exhaustive: every path covered)
```
*You picked typed fields; the CLI enumerated the cases; you mapped each to a pin. No CEL. `verify` proved exhaustiveness — the thing n8n can't safely offer.*

**Beat 4 — Draft agents + tools (capability-declared).**
```
$ swarm agent edit technical-draft
> Tools?  [search Colony]  kb.search  ✓
> Emits:  ticket.draft_ready {draft_text}
> verify ✓
```
*`tools:` is the allowlist — the sandbox injects only `kb.search`; the agent cannot touch anything else, and it shows in the audit surface.*

**Beat 5 — Human gate + actions (durable activities, not inline calls).**
```
$ swarm gate approve --on ticket.draft_ready --pack approval
> On approve → send_email (activity), then crm.create_note (activity)
> On reject  → loop back to draft with the human's note
> send_email idempotency key?   ticket.id      (a retry never double-sends)
> verify ✓
```
*Sending email is external I/O → a durable activity with an idempotency key. Crash after "approve", before "send" → replay resumes without double-sending.*

**Beat 6 — The trust review: the capability surface (not the YAML).**
```
$ swarm describe support-triage
support-triage (entity: ticket)
  CAN:  read inbound email · classify (LLM, no I/O) · draft replies (technical may search KB) ·
        send email ONLY after approval (idempotent) · write a CRM note (idempotent)
  CANNOT: send without approval · call undeclared tools · touch anything outside gmail/kb/hubspot
  NEEDS SECRETS: gmail_oauth, hubspot_api_key, anthropic_api_key
  REPLAYABLE: yes   DURABLE: yes   EFFECTS VISIBLE IN CONTRACT: yes
```
*You approve **what it can do**, provably, before it runs. The generated `support-triage/` is real YAML you can open/diff — you just never had to write it.*

**Beat 7 — Run, watch, replay.**
```
$ swarm secrets set gmail_oauth --oauth      # inline OAuth, not "go edit a file"
$ swarm serve --dev
$ swarm trace <run>          # every step, tool call, agent decision
$ swarm run replay <run>     # reproduce exactly why it drafted / escalated
```

---

## 3. Part 2 — It becomes composable: the flow gets an interface

Publishing derives the **pins** (#1468) — what makes it a building block:
```
$ swarm publish support-triage --to colony
interface:
  inputs:  [ email.received ]                         (from gmail trigger)
  outputs: [ ticket.resolved@1 {ticket_id, category, resolution, sentiment},
             complaint.escalated@1 {ticket_id, customer_id, reason} ]
  requires: backend[anthropic], tools[gmail, kb, hubspot]
  ontology: support-ontology@1.1
verify ✓ · platform_version ">=1.8" · pushed support-triage@1.0
```
*Outputs are ontology-typed events, so a different author's flow can consume them without knowing anything about triage internals.*

---

## 4. Part 3 — Compose into a larger system: `customer-ops`

**Pull the pieces (one ours, one a stranger's, one a template):**
```
$ swarm new customer-ops
$ swarm add support-triage@1.0
$ swarm add churn-watch@2.3
$ swarm add csat-survey@1.1
```

**Wire by pin compatibility — the tool proposes only compatible connections:**
```
$ swarm wire
Unwired outputs → compatible inputs (matched via ontology):
  support-triage.ticket.resolved     → csat-survey.on_resolved   ✓ exact match
  support-triage.complaint.escalated → churn-watch.signal        ⚠ cross-ontology
      churn-watch expects: risk.signal (retention-ontology)
      found mapper: support→retention@0.4 (complaint.escalated → risk.signal)
      insert it? [y]
Wire all proposed? [y / pick]
> verify ✓
```
*Two connections are exact (shared ontology); one crosses ontologies → the tool finds and inserts a mapper pack and shows the field mapping to confirm. Impedance-matching handled, not a black box.*

**The composed capability surface (whole system, one screen):**
```
$ swarm describe customer-ops
customer-ops  composes: support-triage@1.0, churn-watch@2.3, csat-survey@1.1
  CAN: read/send support email · classify & draft (LLM) · human-approve sends ·
       write CRM · survey resolved tickets · flag at-risk customers on complaints
  NEEDS SECRETS: gmail_oauth, hubspot_api_key, sendgrid_api_key, anthropic_api_key
  DATA FLOW:
    email.received → support-triage → ticket.resolved    → csat-survey → survey.sent
                                    → complaint.escalated → [mapper] → churn-watch → risk.flagged
  REPLAYABLE across the whole system: yes
```
```
$ swarm serve --dev          # runs the whole composed system, durably
```
*Three independently-authored flows — one a stranger's, bridged by a mapper — and you can still see and prove the whole system's effect surface on one screen, and replay the entire composed run.*

---

## 5. Why this is the ideal path (the through-line)

1. **You never wrote YAML — but the YAML exists**, verified and auditable. Ease *and* moat, because the CLI generates the contract instead of hiding it.
2. **Two type-directed operations do the heavy lifting** — wiring (graph search over pins) and routing (decision tables over typed fields). Both reduce to "pick typed options; tool generates + verifies." Neither is possible in an arbitrary-code tool.
3. **The architecture that makes it easy is the same one that makes it safe** — judgment→agents, computation→compute, I/O→durable activities, dispatch→dumb router, connection→typed pins. Each is *both* the auditability mechanism *and* the "specifiable via CLI" mechanism.
4. **The capability surface is the review artifact** at creation *and* composition — the trust moment n8n and LangChain can't offer.
5. **Composition preserves the guarantees** — a system built from three flows (one a stranger's) is still replayable and its full effect surface is one screen. That is the thing that "really can't be achieved with existing technology."

## 6. What it rests on (the roadmap implication)

Everything above is downstream of three investments — over-invest here:
1. **Typed interfaces / pins (#1468)** — makes wiring type-directed and routing enumerable. The single highest-leverage unlock.
2. **switch/lookup/threshold helpers (#1668)** — the declarative forms the CLI generates routing into.
3. **Ontology packs** — shared event vocabularies, so independent authors' flows connect (the interop stdlib).

Plus the prerequisites the wedge needs to be *felt*: **`serve --dev` just works** (the #1578 onboarding thread), **credentials are inline/OAuth** (#1645/secrets), and **the ~20 flagship packs exist in Colony** so the picker/agent has real things to pick.

**Sequencing:** make composition *type-checkable* first (pins + ontology + `verify` catching bad wiring/routing), *then* the assembly agent is a thin, improving layer on top — it just drives the same checkable operations a human would, with `verify` as its net. Build the agent before the type layer is solid and you get a confident-but-wrong composer.

## 7. Honest hard parts / open questions

1. **The assembly agent (NL → correct composition) is genuinely unsolved** — pack selection, mapper insertion, ambiguity resolution. Ship the *guided* scaffold first (deterministic, achievable); treat NL `create` as the improving layer, with `verify` guaranteeing even a wrong guess produces a valid-or-rejected flow, never a broken one.
2. **Mappers are the real gap** — cross-ontology wiring needs them. Escape: a marketplace mapper library for common pairs, or the agent *generates* a mapper (a small pure-compute transform) and `verify`s it + shows the field mapping to confirm.
3. **Semantic ≠ structural** — two events with the same schema can mean different things. Ontologies mitigate; the *capability-surface review* is where a human confirms intent that types can't guarantee. (So the review moment is not optional.)
4. **Logic is not wiring** — auto-wiring does the plumbing; branch *conditions* are the routing/decision-table problem (#1668) or an agent-proposed condition. Keep them separate; both get reviewed.
5. **Ambiguity always needs a chooser** — you can't automate away *choice*, only reduce it from "design the graph" to "pick one of two."
6. **Credential friction can kill the wedge** — if `create` ends with "go set up secrets," it's lost. The create flow must prompt/OAuth inline so it ends *running*.
7. **Pack-library chicken-and-egg** — "triage my email" fails with no gmail pack. Seed Colony's flagship packs before the `create` demo is worth showing.
