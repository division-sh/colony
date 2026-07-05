# Vision: a capability-bounded microkernel for agent workflows

> Status: thinking doc for review. Not committed spec. Captures the extensibility thesis, the pack zoo, and the locked design directions it rests on. Colony is the registry (not the authority) for these packs.
>
> **v2** — reworked after review: §8 replaced (no more `logic_node`-with-tool-access; pure compute + durable activities + declarative selection), moat wording sharpened, two-layer pack envelope, capability-diff UX, registry-is-not-authority, ontology governance, the new-kernel-primitive process, and staged sequencing.

---

## 1. Thesis

**Swarm is a microkernel for agent workflows.** A small, trusted **kernel** owns the guarantees — determinism, atomicity, replay, isolation, the trust/crypto primitives, scheduling, and the sandbox. Everything else is a **capability-bounded marketplace of declarative packs** that plug into well-defined **boundaries** and only ever *compose declared kernel primitives* — they never become, or extend, the kernel.

The one-line moat:

> **Third-party extension does not erode the runtime's guarantees.**

Everyone has packs, plugins, templates, nodes, actions — rectangles with marketing names. The differentiator is that in Swarm, adding third-party extensions preserves replay, auditability, typed contracts, and static effect visibility. That is the architectural moat, and it only holds if the discipline below holds.

## 2. The core principle (and the moat, stated carefully)

> **If guarantees rest on it, it is kernel. If it composes primitives, it can be a pack.**

A pack must not redefine signature verification, storage/replay semantics, scheduling, entity isolation, or sandbox rules. If it can, it is a kernel extension pretending to be a plugin.

And the discipline that keeps the marketplace from becoming "download random code and pray":

> When a proposed pack cannot be expressed as **manifest + optional pure compute + optional durable activity + capability declaration**, that signals the need for a new **kernel primitive** — a deliberate platform decision, not "just let the pack run code here."

**The moat, worded to survive attack** (not "structurally impossible for competitors" — that invites edge-case hunting):

> In arbitrary-code plugin marketplaces, preserving replayability and static effect visibility across composition is not the default property. Swarm's bet is that packs remain statically auditable because **all effects pass through platform-owned primitives** and **every pack exposes a declared capability surface** — structurally enforced, verified by the runtime, not by trust.

## 3. The kernel-vs-pack test (four checks)

For any proposed extension:

1. **Composition vs guarantee** — does it compose a primitive (pack) or do the guarantees rest on it (kernel)?
2. **Verify-without-executing** — can `verify` reject a bad pack *without running its code*? If no → it's kernel or unsupported.
3. **One-screen capability delta** — can an importer understand what it can do (and the delta on upgrade) on one screen? If no → the capability language is too weak or the pack type too broad.
4. **Replay reproduce-or-gate** — can replay reproduce, or intentionally gate, *every* external effect? If no → the effect model is incomplete. **This is the master test — it *is* the moat, restated as an admission criterion.**

## 4. The unified pack envelope — two layers

A prompt library and a webhook verifier are not the same animal; one speaks to customers, the other accepts hostile internet traffic. So: **one universal envelope + a type-specific body.**

**Universal envelope (every pack):**
```yaml
pack:
  id:
  version:
  platform_version:      # enforced (#1659) — anti-rot anchor
  type:                  # trigger | connector | compute | flow | agent | ontology | policy | …
  manifest_hash:
  provenance:
  capabilities:          # the audit surface (§6)
  dependencies:
  exports:
  tests:
```

**Type-specific body (examples):**
```yaml
trigger:    { admission: { signature, delivery_id, event_type } }
connector:  { tools: [{ id, endpoint, auth, effect_class }] }
compute:    { modules: [{ path, interface, resources }] }
ontology:   { events: [{ name, schema, compatibility }] }
agent:      { roles: [{ prompt, model_alias, allowed_tools }] }
```

One registry, one trust envelope — without pretending all pack types carry the same lifecycle, risk, or review burden.

## 5. The pack zoo

Fit: ✓ clean · ◐ partial (needs a primitive) · 🚫 kernel.

**Ingress:** ✓ provider-trigger (§7) · ✓ schedule/cron (timer) · ✓ poller (activity+timer+dedupe) · ✓ inbound-email · ◐ queue/stream (needs a consumer primitive).
**Egress:** ✓ API connector (http+auth+retry+effect-class) · ✓ notification/channel · ✓ data-sink · ✓ MCP-server · ◐ telemetry-exporter (**governance boundary — §9**).
**Compute (pure, no I/O):** ✓ compute-module (§8) · ✓ predicate/guard (CEL) · ✓ transform/mapping · ✓ parser.
**Composition:** ✓ flow (Colony) · ✓ agent/role (high value, low mechanism) · ✓ prompt-library.
**Contracts & interop:** ✓ type/schema · ✓ entity/state-machine · ✓ **event-ontology (the stdlib — §10)**.
**Policy/governance:** ✓ policy · ◐ guardrail/compliance (**enforcement is kernel-owned — §9**).
**Human:** ✓ approval/mailbox.
**Backends:** ◐ LLM-backend (**only the OpenAI-compatible slice; novel protocols = platform work — §9**).

🚫 **Kernel (not packs):** determinism engine / event loop / routing; transactional store; sandbox/workspace substrate; trust/crypto/signature primitives; scheduling.

**Near-term four already building:** provider-trigger (#1704), connector/tool (http + #1666), compute (#1666/#1669), flow (Colony). Best next value-per-effort: **agent/role packs** and **event-ontology packs**.

## 6. Capability audit as a first-class design object

"Capability-audited" must mean more than "a YAML file exists." The importer sees a plain-language summary on install:

```text
Import: stripe-payments@1.4.2
This pack CAN:
- Receive HTTPS webhooks at /inbound/stripe; verify with webhook_signing.stripe
- Persist Stripe event-id dedupe markers
- Emit: payment.captured@1, payment.failed@1, subscription.cancelled@1
- Call Stripe API: GET /v1/events/{id} (read_only), POST /v1/webhook_endpoints (idempotent_write)
- Read policy: stripe.webhook_tolerance ; Requires: stripe_api_key, webhook_signing.stripe
This pack CANNOT:
- Write entity state directly · emit undeclared events · call undeclared tools · run code before admission
```

And a **capability diff on upgrade**, requiring re-approval:
```text
stripe-payments 1.4.2 → 1.5.0
+ NEW: POST /v1/refunds (idempotent_write)   → importer approval required
```

## 7. Locked design A — provider-trigger packs (#1704)

Stripe is the first proof of a **provider-trigger manifest engine**, not another hardcoded Go adapter. Three layers: **trigger manifest** (platform-interpreted admission, fail-closed), **optional pure compute** (post-admission normalization only), **setup activities** (durable I/O for endpoint create/teardown).

The trigger manifest composes kernel-owned primitives (raw-body capture, HMAC, timestamp/replay rejection, constant-time compare, JSONPath extraction, dedupe, durable-ack). **No provider code runs before admission; provider code never decides trust.**

**Governing invariant:** the manifest *composes a closed, platform-owned* verification set; a scheme it can't express is **fail-closed unsupported** (→ a new kernel primitive via §12), never an escape hatch.

Robustness details to include:
- **Secret rotation:** `secret: { key: webhook_signing.stripe, rotation: { allow_previous_for: 24h } }`.
- **Challenge/handshake:** `challenge: { type: echo_param, param: challenge, response: "{challenge}" }` — platform-owned, no pre-admission code.
- **Mandatory generality acceptance:** the manifest engine is **not accepted** until it expresses **Stripe, GitHub, Slack, and one URL-canonicalization provider (Twilio)**. If it can't, ship the provider as a Go adapter and don't pretend the engine exists.
- **No dynamic event explosion:** publish one `inbound.stripe` with `provider_event_type`/`provider_delivery_id` in payload; let a deterministic `switch` node route. A 200-event catalog is a phone book, not a contract.

## 8. Locked design B — pure compute + durable activities (supersedes the old `logic_node`)

**The old model (removed):** a `logic_node` execution type with direct `ctx.tools` access and direct `ctx.emit`. That makes code a mini-workflow-engine — the exact "plugin runs effects" pattern the thesis forbids, and it fails Test 3 (replay can't gate effects it can't see). Gone.

**The end-state split** (from #1663/#1666/#1668):
- **`compute` handler step** — pure WASM. Declared input/output, `store_as`, `capabilities: []` (none — it's pure), resource bounds. **No tools, no emits, no writes.** Removing the code body still leaves the contract's states/emits/writes enumerable.
- **`activity` handler step (#1666)** — the *only* path for external I/O. Tool + input + typed success/failure events + effect class + idempotency. Durable, journaled, replay-aware.
- **Declarative selection (`rules`/`switch`, #1668)** — consumes compute/activity results to choose branches and construct emits. Contract-visible.
- **Orchestration functions (#1671) — future, not yet designed.** The eventual *source-syntax* layer for durable-execution chains over declared activities. Referenced here as the intended direction; **not a locked design** — I'm marking it speculative rather than repeat the §8 mistake on a different rung.

Reworked scaffold example — WASM computes, activity does I/O, handlers emit, contract stays enumerable:

```yaml
# nodes.yaml
scaffold-orchestrator:
  event_handlers:
    repo_scaffold.template_requested:
      compute:                                   # pure WASM — no tools, no emits
        module: ./logic/template_resolver.wasm
        export: resolve
        interface: empire:repo-scaffold/template-resolver@1
        input:
          component_type: { from: payload.component_type }
          artifact_type:  { from: payload.artifact_type }
          template_id:    { from: payload.template_id }
        store_as: computed.template_resolution
        capabilities: []
        resources: { fuel: 250000, memory_mb: 16, output_kb: 64 }
      rules:
        - id: invalid_template
          condition: computed.template_resolution.valid == false
          emit: { event: repo_scaffold.template_rejected, fields: { reason: computed.template_resolution.reason } }
        - id: load_template
          condition: computed.template_resolution.valid == true
          activity:                              # durable I/O — the only way code touches the world
            id: load-template
            tool: read_flow_data
            input: { path: { from: computed.template_resolution.path } }
            success_event: repo_scaffold.template_loaded
            failure_event: repo_scaffold.template_load_failed
            idempotency_key: { from: event.id }

    repo_scaffold.template_loaded:               # a later handler consumes the activity result
      emit: { event: repo_scaffold.commit_requested, fields: { request_id: event.id, files: payload.files } }
```

Boundaries are now clean: WASM resolves a path; the activity reads flow data; handlers emit typed events; the contract shows every effect; no code calls tools or owns workflow authority.

**WASM substrate reasoning (unchanged, still right):** inline-code+AST-analyzer and "sandboxed Python" both converge on WASM (you need a runtime sandbox regardless; halting/resources are undecidable). Prefer **soundness by construction** (declared capabilities + generated types + platform-provided determinism) over **soundness by analysis**. Determinism by provision: the platform supplies `ctx.now/random/uuid`/derived ids and strips ambient nondeterminism; resource bounds (fuel/timeout/memory) → fail-closed, never hang; ship source + compiled `.wasm` bound by hash; execution trace for debuggability.

## 9. Higher-risk categories need extra rules

- **Telemetry exporter = governance boundary, not a normal connector.** Traces can leak payloads, entity state, tool results, PII, credentials. So it declares an *observation capability* with field allow/redact:
  ```yaml
  capabilities: { observe: { events: [payment.*], fields: { allow: [event.type, payload.amount], redact: [payload.customer_email, payload.card_last4] } } }
  ```
- **Guardrail/compliance:** a pack may declare *policies and pure classifiers*, but **enforcement points are kernel-owned.** If a pack's own trust decision gates an event, a guarantee rests on it → it's kernel-adjacent, not a pack.
- **LLM backend:** OpenAI-*compatible* only (base URL + model map). Backends affect determinism, cost accounting, tool-call semantics, retry, and safety boundaries — not casual extension territory.

## 10. Event-ontology packs — the interop stdlib (with governance)

A marketplace compounds only if independently authored packs agree on shared event contracts; otherwise every pack emits a private dialect and the ecosystem is Babel with semver. Ontology packs are the stdlib — but they need governance:
- **Versioning + compatibility:** add-optional = compatible; add-required / rename / narrow-enum = breaking; widen-enum = maybe; meaning-change-without-schema-change = forbidden (and unpoliceable — humans).
- **Mapping packs:** keep raw provider events separate from canonical business events — `stripe trigger (inbound.stripe) → stripe-commerce mapper → commerce.payment.captured → fulfillment flow`.
- **Namespace ownership — open governance question, not a spec field.** Lean: **federated reverse-DNS namespaces** (`com.stripe.payment.*`) **plus a small curated canonical set** Colony blesses (npm-scopes + an stdlib). Don't over-specify policy now; flag it early so two "standard" payment ontologies don't fork the ecosystem.

## 11. Colony is a registry, not a trust oracle

> Registry trust helps humans decide what to import. Runtime verification decides what is allowed to execute.

- **Colony distributes:** discovery, version metadata, signed artifacts, provenance, compatibility/anti-rot test results, deprecation notices, reputation signals.
- **The runtime independently verifies (every import):** manifest schema, `platform_version`, hashes, signatures, declared capabilities, dependency resolution, resource limits, module validity, effect classes, credential bindings.

A registry listing is **never** sufficient authority to execute.

## 12. When a pack can't express something: the primitive process

The answer is never "just add code here." It is:

```
unsupported → RFC → primitive vocabulary → conformance tests → platform implementation
```

A new kernel primitive is a deliberate, tested, platform-owned addition — reviewed as security-critical. That's what keeps "capability-bounded" true.

## 13. Sequencing — earn the abstraction, don't build it upfront
Do **not** build the unified pack registry first (grand abstraction, no scar tissue). Instead:
1. **Provider trigger + connector pair** (one Stripe pack: trigger manifest + connector tools + setup activities + ontology mapping + capability summary). Acceptance: re-express ≥2 existing adapters as manifests (§7).
2. **Durable activity / effect-class foundation** (#1666) — must land before the connector marketplace gets serious.
3. **Compute pack** — pure WASM only; no tools, emits, or writes.
4. **Flow pack imports** — a flow importing trigger + connector + compute + ontology; prove the composed workflow stays verifiable.
5. **Agent/role packs** — high value; needs prompt provenance, model compat, tool restrictions, eval fixtures.
6. **Ontology governance** — design early, make first-class after one or two real provider/flow examples show the shape.

## 14. Open questions
1. Is the kernel/boundary line drawn correctly?
2. Does the two-layer envelope hold across *all* types, or do some want more divergence?
3. Capability-audit UX at scale — does the one-screen delta stay tractable as pack types multiply?
4. Namespace governance model (federated vs curated vs both)?
5. The failure mode to police forever: **half-declarative extension points** (manifest covers 80%, code bolted on for the rest). §12 is the discipline; is it enough?
6. Does Colony's flow-only design generalize cleanly to a multi-type pack registry?
