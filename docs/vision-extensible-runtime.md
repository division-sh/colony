# Vision: a maximally extensible agent runtime (microkernel + pack marketplace)

> Status: thinking doc for review. Not committed spec. Captures the extensibility thesis, the pack zoo, and the two locked design directions it rests on (#1704 provider-trigger packs, and WASM/logic-node support from #1460). Colony is the registry for these packs.

---

## 1. Thesis

**Swarm is a microkernel for agent workflows.** A small, trusted **kernel** owns the guarantees — determinism, atomicity, replay, isolation, the trust/crypto primitives, and the sandbox. Everything else is a **marketplace of declarative packs** that plug into well-defined **boundaries** and only ever *compose the kernel's primitives* — they never become, or extend, the kernel.

The differentiator (the moat, §6): because a pack can only compose the deterministic kernel + durable activities + pure compute, **a workflow assembled from N third-party packs is still fully replayable, auditable, and verifiable.** Adding plugins does not erode correctness. That is structurally impossible in an arbitrary-code plugin marketplace (Zapier / n8n / community activities), and it is the reason to build this.

## 2. The core principle

> Packs compose primitives at boundaries. The kernel owns guarantees. If the guarantees *rest on* it, it's kernel — never a pack.

- **Kernel (never pack-extensible):** determinism engine / event loop / routing; the transactional store (it *is* the atomicity/replay guarantee); the sandbox / workspace-execution substrate (pack-added execution = sandbox escape); trust/crypto/signature primitives (pack-added verification = supply-chain attack); novel LLM protocols.
- **Boundaries (pack surface):** ingress (triggers), egress (connectors/tools), compute (pure logic), composition (flows/agents), contracts/interop (types/ontologies), policy, human-in-the-loop.

The test for any proposed extension: *does it compose a primitive (→ pack) or do the guarantees rest on it (→ kernel)?* When a proposed pack can't be expressed as **manifest + optional pure-compute + optional durable-activity + capability declaration**, that's the signal it needs a new **kernel primitive** first — a deliberate platform decision, not a pack.

## 3. The unified pack shape

Every pack, whatever its type, is the same envelope:

```
pack
├── manifest              # declarative: composes closed platform primitives (fail-closed)
├── optional pure compute # WASM: post-trust, deterministic, capability-sandboxed (see §8)
├── optional activities   # durable external I/O (#1666)
└── capability declaration # which primitives/secrets/effects/tools it touches — the audit surface
```

Properties, uniform across all pack types:
- **Fail-closed** at every boundary the primitives don't cover.
- **Capability-audited** — one capability language across pack types; this is what an importer reviews.
- **Publishable as data** — a pack is a directory, verified, versioned. No recompiling the platform.
- **`platform_version`-pinned + anti-rot verified** (#1659/#1660) — every pack type inherits Colony's guarantees.
- **Determinism-preserving** — packs run through the same event-loop/journal, so composition stays replayable.

**Colony is the registry for *all* pack types**, not just flows. Triggers, connectors, compute modules, agents, ontologies — all the same pack shape, one registry, one trust/verification/versioning model.

## 4. The pack zoo

Fit: ✓ clean · ◐ partial (needs a primitive) · 🚫 kernel (not a pack).

**Ingress — how the world starts a flow**
- ✓ **Provider-trigger** (§7) — webhook admission. Composes HMAC/timestamp/JSONPath/dedupe. (Stripe/GitHub/Slack/Twilio/Shopify.)
- ✓ **Schedule/cron** — emit on a schedule. Composes the timer primitive.
- ✓ **Poller** — poll a source, emit per new item, dedupe. Composes activity + timer + dedupe. (RSS/IMAP/API.)
- ✓ **Inbound-email** — parse inbound-parse webhook → event.
- ◐ **Queue/stream** — Kafka/SQS/PubSub. Needs a stream-consumer kernel primitive.

**Egress — how a flow affects the world**
- ✓ **API connector** — declarative tool set for a service's API. Composes HTTP tool + auth + retry + response-map + effect-class (#1666). Symmetric twin of triggers; a real "provider pack" is trigger **+** connector.
- ✓ **Notification/channel** — Slack/email/SMS/push.
- ✓ **Data-sink** — Sheets/Airtable/S3/DB. Effect-class `idempotent_write`.
- ✓ **MCP-server** — bundle+register an MCP server as tools.
- ◐ **Telemetry-exporter** — traces/events → Datadog/OTLP. Needs a subscribe-and-export primitive.

**Compute — pure, deterministic, no external effects**
- ✓ **Compute-module** (§8, #1460 logic_node) — WASM: scoring, validation, normalization, parsing, rule-classification.
- ✓ **Predicate/guard** (#1464) — reusable named CEL predicates.
- ✓ **Transform/mapping** — reusable field normalization (#1683 generalized).
- ✓ **Parser** — CSV / a document type / a log format → structured.

**Composition — assembled behavior**
- ✓ **Flow** (Colony) — complete workflow (template) or composable sub-flow (pattern).
- ✓ **Agent/role** — a reusable, tuned agent (prompt + tools + config). *High value, near-zero new mechanism — agents are already declarative.*
- ✓ **Prompt-library** — reusable prompt templates/fragments.

**Contracts & interop — the stdlib**
- ✓ **Type/schema** — shared types ("Money", "Person").
- ✓ **Entity/state-machine** — a reusable lifecycle ("Order" FSM).
- ✓ **Event-ontology** — a canonical event vocabulary for a domain. **The most strategically important type — see §9.**

**Policy / governance**
- ✓ **Policy** — reusable budget/rate/escalation/retention rules.
- ◐ **Guardrail/compliance** — PII redaction, content filters (some declarative, some compute).

**Human boundary**
- ✓ **Approval/mailbox** — reusable human-decision surfaces.

**Backends (partial)**
- ◐ **LLM-backend** — only the OpenAI-*compatible* slice (base URL + model map). Novel protocols = platform work.

## 5. Near-term vs later
Already building the first four: **provider-trigger (#1704), connector/tool (http + #1666), compute (#1460), flow (Colony).** Highest value-per-effort next tier: **agent/role packs** (huge value, near-zero mechanism) and **event-ontology packs** (the interop stdlib, §9). Everything else is deliberately later, one boundary at a time.

## 6. Why this is a moat (the not-Zapier answer)
Every plugin marketplace has the same fatal property: plugins are arbitrary code, so the *composed* workflow is a black box — not replayable, not verifiable, unauditable. Swarm inverts it: packs can only compose the deterministic kernel + durable activities + pure compute, so **a workflow assembled from third-party packs is still provably deterministic, replayable, and auditable.** You can hand someone a workflow built from strangers' packs and *prove what it does and what it can reach*. That property is impossible in an arbitrary-code marketplace, and it is the whole reason this is worth building.

---

## 7. Locked design A — provider-trigger packs (#1704)

From the #1704 refinement: Stripe is not "another hardcoded Go adapter" — it's the first proof of a **provider-trigger manifest engine**. A provider pack has three layers:

```
provider pack
├── trigger manifest      # declarative ingress/security/admission — platform-interpreted, fail-closed
├── optional compute      # pure normalization/classification only (post-admission)
└── setup flows           # durable activities for provider API setup/teardown
```

**Layer 1 — trigger manifest** (the pre-event admission layer; platform-interpreted, fail-closed):

```yaml
provider: stripe
trigger:
  provider: stripe
  secret: webhook_signing.stripe
  signature:
    type: hmac_sha256
    header: Stripe-Signature
    timestamp_param: t
    signature_param: v1
    signed_payload: "{timestamp}.{raw_body}"
    tolerance: 5m
  delivery_id: { json_path: $.id }
  event_type:  { json_path: $.type }
  event_name:  inbound.stripe.{event_type}
  ack: { mode: durable_before_dispatch }
```

Runtime owns the generic primitives: raw-body capture, HMAC verification, timestamp tolerance/replay rejection, constant-time compare, JSONPath extraction, delivery-id/event-type validation, dedupe marker persistence, durable-ack policy, redaction. The pack declares *how* Stripe wires them. **It runs no code before admission.**

**Layer 2 — optional pure compute** (post-admission normalization only; `compute_module.wasm`, pure, cannot decide trust). Often unnecessary if signature + JSONPath + event templating suffice.

**Layer 3 — setup/teardown via durable activities** (creating/deleting a webhook endpoint is external I/O → activities, not admission code). Split from the first slice.

**Governing invariant (the load-bearing rule):** the manifest *composes* a **closed, platform-owned set** of verification primitives; it can **never introduce a verification algorithm via code.** A provider whose scheme doesn't fit the primitives is **fail-closed unsupported** (needs a deliberate new platform primitive) — *not* an escape hatch. Provider code deciding trust = supply-chain attack.

**Review feedback to fold in (de-risking the generality claim):**
1. Design the primitive vocabulary against the real **provider zoo**, not Stripe alone — the axes vary a lot (header format; signed-payload construction: body vs body+ts vs Twilio's URL+sorted-params; hash SHA256/SHA1/RSA; hex vs base64; timestamp location; challenge flows). **Prove generality by re-expressing the two already-shipped adapters (GitHub, Slack) as manifests and retiring their Go.** If it can only do Stripe, it's premature and you ship a split model (2 Go + 1 manifest = worst of both).
2. Make the trust-primitive set explicitly **closed and platform-owned** (above).
3. Be honest this is a **manifest-engine slice**, not "add Stripe." If the manifest can't cleanly re-express GitHub+Slack in one PR, split: ship Stripe as a Go adapter + the clean package boundary now, do the engine as a deliberate next slice from three real providers.
4. **Dynamic event names** (`inbound.stripe.{event_type}`, ~200 Stripe types) need a subscription story — lean toward publishing `inbound.stripe` with `event_type` in payload and letting a deterministic node route, rather than 200 event names.
5. Provider packs share the **flow/Colony packaging + capability model** — the manifest's `secret:` + declared primitives *are* its capability surface.

---

## 8. Locked design B — WASM / logic-node support (#1460)

**Two primitives, declarative-first:** `tool_call` (declarative "call a tool, map I/O", the trivial-caller majority) + `logic_node` (real computation, the minority). Keep CEL simple; graduate to `logic_node` rather than growing a middle DSL.

**The `logic_node` authoring shape** — contract declares the *effect surface*; a referenced module implements the body; types are generated from the contract:

```yaml
# nodes.yaml
- id: template-validator
  execution_type: logic_node
  entry: logic/template_validator.ts       # compiles to WASM
  subscribes: [repo_scaffold.template_requested]
  tools: [read_flow_data]                    # the capability allowlist
  emits: [repo_scaffold.commit_requested, repo_scaffold.template_rejected]
```

```ts
// logic/template_validator.ts — types generated by `swarm gen` from the contract
import { handler, TemplateRequested } from "./gen/repo-scaffold";
export default handler(async (event: TemplateRequested, ctx) => {
  const key = `${event.payload.component_type}/${event.payload.artifact_type}`;
  const template = TEMPLATES[key];
  if (!template || event.payload.template_id !== ctx.entity.expected_template_id)
    return ctx.emit.templateRejected({ reason: `unknown template tuple: ${key}` });
  const files = await Promise.all(
    Object.values(template).map(path => ctx.tools.readFlowData({ path })));
  return ctx.emit.commitRequested({
    request_id: ctx.requestId(),             // derived from source_event_id — replay-stable
    files: files.map(f => ({ path: f.path, content: f.content })),
  });
});
```

**Why this factoring survives our constraints (verifiable / deterministic / replayable / safe-to-import):**
- **Capability surface is contract-declared** (`subscribes`/`tools`/`emits`), so `verify` works *without reading the code*, and the sandbox injects only declared tools into `ctx` — the body physically cannot call an undeclared tool.
- **Generated types** make the code conform to the contract at compile time (undeclared emits / malformed payloads fail before boot).
- **Determinism by provision, not prohibition** — `ctx.requestId()` / `ctx.now()` / `ctx.random()` replace the nondeterministic globals, which the sandbox strips. Replay works because the platform owns every nondeterminism source.
- **Colocated + mirrors `agents.yaml`** — module lives in the package (portable), slots into the existing execution-type model.

**The substrate is WASM (for untrusted/importable code).** Reasoning we locked:
- Inline-code + an AST analyzer, and "sandboxed Python," both *converge on WASM* — you need a runtime sandbox regardless (halting/resources are undecidable), so make the sandbox the boundary and keep code free inside. "Sandboxed Python done safely" is just Python-on-WASM.
- Prefer **soundness by construction** (declared capabilities + generated types + provided determinism) over **soundness by analysis** (a heroic analyzer over an adversarial general language — the vm2 graveyard).

**Envelope decisions to pin (around the authoring shape):**
1. **Substrate = WASM** for untrusted/importable; a locked isolate only for first-party-non-importable.
2. **Finish the determinism surface** — strip `Date.now`/`Math.random`/`crypto.randomUUID`; provide `ctx.now/random/uuid/requestId`. Async contract: tool results cached per `(event_id, call)`; `Promise.all` order-preserved OK; `Promise.race`/timers/wall-clock banned.
3. **Runtime resource bounds** (the one thing no analyzer gives you): fuel/timeout/memory → exceed → `handler_error` → fail-closed, never hang.
4. **Packaging** — ship source (`.ts`, auditable) + compiled `.wasm` (capability-verified), bound by a content hash.
5. **Execution trace** — the `logic_node` equivalent of `conversation.get_turn`, so moving logic out of agents doesn't cost debuggability.
6. **Entity-write relationship** — emit-only, or writes *declared* in the contract; the single-writer invariant survives.

**Keep it the escape hatch above declarative.** `tool_call` (no build, no code) is the default for the trivial-caller majority; `logic_node` is for genuine logic. The code tier must be *low-friction* (fast `gen` + compile, good local dev) or the boundary rots (people cram logic into CEL or trivial agents).

**Relationship to activities (#1666):** `logic_node` is Layer-2 pure compute (post-trust, deterministic, no I/O). External I/O is *always* a durable `activity`. So: declarative `activity` for tool calls, `logic_node` for computation, and both compose the kernel — which is what keeps a pack-assembled workflow replayable.

---

## 9. The interop stdlib — event-ontology packs
The zoo is a pile of islands until packs from *different authors* can compose. That requires shared **event vocabularies**: a Stripe-trigger pack emitting `payment.captured` is only useful if a fulfillment-flow pack (different author) consumes that same event. Event-ontology packs are the interop layer (npm's `@types`) — treat them as first-class, not an afterthought. They're what turn "a marketplace of packs" into "an ecosystem that compounds."

## 10. Open questions to think through
1. Is the microkernel/boundary line drawn correctly? (Are triggers/connectors/compute/flows/agents/ontologies the right boundary set, and are store/sandbox/trust/engine the right kernel set?)
2. Does the **one unified pack envelope** (manifest + compute + activities + capability decl) actually hold across all types, or do some (ontology, policy, agent) want a different shape?
3. Capability-audit UX at scale: as pack types multiply, does the importer's audit stay tractable with one capability language?
4. Sequencing: prove the pattern on providers (#1704), then connectors (symmetric), then agents/ontologies? Or unify the pack model *first* and let types fall out?
5. The failure mode to watch: **half-declarative extension points** (manifest covers 80%, code bolted on for the rest) — where's the discipline that keeps every pack composing-only, fail-closed on the rest?
6. Does Colony's current design (flow-only) generalize cleanly to a **multi-type pack registry**, or does that need its own rethink?
