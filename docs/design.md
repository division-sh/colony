# Colony — design & review

One place to review the whole idea: what Colony is, how it's structured, the metadata decision, and the platform work it depends on.

---

## 1. Vision

**Colony is a curated, CI-verified library of Swarm flows** — reusable, working starting points to fork, and composable building blocks to wire in. It is the **trusted, first-party precursor to the Swarm flow marketplace**.

It does three jobs at once:
1. **Kills the authoring learning curve** — "fork this working thing" beats "learn the dialect cold."
2. **Is the corpus for the authoring agent** — a flow-authoring agent needs verified examples to pattern-match against.
3. **De-risks the marketplace** — you can't open a marketplace to third parties until you can curate reusable flows internally. Colony is that proof, and it *forces* portability/composition (#1450, #1468) to be real.

**Curated library ≠ marketplace.** Colony is first-party + trusted (bar = *reusable + verified*). The marketplace is third-party + untrusted and additionally needs the sealed boundary, a capability-audit/trust model, sandboxing (why `logic_node`→WASM matters), versioning, and discovery. Build the curated version first; let it tell us what the trust layer needs.

## 2. Two tiers

| Tier | Dir | What | Extra bar |
|---|---|---|---|
| **Templates** | `templates/` | Complete, end-to-end flows — *fork-and-go* | — |
| **Patterns** | `patterns/` | Small, single-concept building blocks — *composable* | declared pins/interface (`schema.yaml`, #1468) |

A template you copy whole; a pattern you wire in — so a pattern's interface *is* its contract.

## 3. Metadata model (the key decision)

**One manifest, derive the rest. No Colony-specific metadata file.**

The initial scaffold invented a `template.yaml` — which duplicated data the flow already declares (name/version → `package.yaml`; interface → `schema.yaml` pins; requires → `tools.yaml`). That's the "restate what's already declared" footgun we fight everywhere else. Dropped.

Instead:
- **`package.yaml` is the manifest.** Colony's catalog is **derived** from it (+ contracts).
- **The only Colony-owned data** is two curation fields — `maturity`, `category` — namespaced under an optional `colony:` block in `package.yaml` that the runtime ignores.
- **Interface/pins** come from `schema.yaml` (#1468) — never restated.
- **`requires`** (backend/tools/credentials) is the one deliberate exception to derive-don't-restate: it's the **import-time audit surface**, so it's declared *explicitly* **and** cross-checked by `verify` against actual usage.

```yaml
# package.yaml
name: twitter-prospecting
version: 1.0.0
author: youmew
description: > …
platform_version: ">=1.6.0"     # anti-rot anchor — must be ENFORCED
flows: [ … ]

keywords: [prospecting, social, scoring]   # (proposed manifest field)
license: MIT                               # (proposed)
repository: https://github.com/division-sh/colony   # (proposed)
requires:                                  # (proposed) explicit + verified
  backend: [anthropic]
  tools: [getxapi]
  credentials: [getxapi_api_key]

colony:                          # Colony curation only (runtime ignores)
  maturity: flagship             # flagship | reference | pattern
  category: lead-gen
```

The line held: **`package.yaml` = what the package says about *itself*** (platform-owned, serves marketplace too); **Colony = the library's *judgment about* it** (`maturity`/`category`).

*Open choice:* `colony:` block inside `package.yaml` (one file, per "just use package.yaml") vs a separate `colony.yaml` (keeps the package pure for forking). Currently: namespaced block.

## 4. Anti-rot (the non-negotiable)

The platform dialect churns; templates rot (twitter-gen already drifted in weeks). So verification is **structural**:
- CI (`.github/workflows/verify.yml`) builds `swarm` from `division-sh/swarm@master` and runs `swarm verify` on **every** entry against current `platform-spec` — on push, PR, **and daily**.
- A platform change that breaks an entry turns CI red → **fix or quarantine**, never weaken the check.
- Each entry pins `platform_version`; enforcing it (platform-side) is the anchor.
- TODO: add the `#1252` scenario runner (`swarm test`) once it lands.

## 5. Repo structure

```
colony/
├── README.md · CONTRIBUTING.md · CODEOWNERS      # what it is + the bar + curation
├── index.yaml                                    # GENERATED catalog (feeds future `swarm add`)
├── scripts/build-index.sh                        # derives index.yaml from each package.yaml
├── .github/workflows/verify.yml                  # anti-rot CI
├── docs/authoring-guide.md · docs/design.md      # how to author + this doc
├── templates/<name>/                             # fork-and-go flows (+ scenarios/)
└── patterns/<name>/                              # composable building blocks (+ scenarios/)
```

## 6. Platform dependency — the manifest issue (DRAFT, to file)

Most of the metadata work is **platform-side, not Colony-specific** — it makes every flow package a proper, versioned, marketplace-ready manifest.

> **[architecture][cli] Enrich `package.yaml` into a package manifest: keywords, license, provenance, enforced platform-version, verified `requires`**
>
> `package.yaml` already carries `name`/`version`/`author`/`description`/`platform_version`/`flows`. To make flow packages proper, marketplace-ready manifests (and to fix template rot), add/enforce:
> - **Enforce `platform_version`** — `verify`/boot must reject when the running spec version doesn't satisfy the range. Today it appears decorative; that's the root cause of dialect-drift rot.
> - **`keywords`** (list) — discovery/search.
> - **`license`** (SPDX) — redistribution, required for a public library.
> - **`repository` / `homepage`** — provenance.
> - **`requires`** (backend/tools/credentials) — an **explicit** capability/dependency block that `verify` **cross-checks against actual usage** (declared-but-unused or used-but-undeclared → error). This is the import-time audit surface and the marketplace trust surface; it should be explicit, not derived-and-hidden.
>
> Non-goals: interface/pins stay in `schema.yaml` (#1468); curation metadata (`maturity`/`category`) stays with the consumer (Colony), not the package.

## 7. Consumers / related work

- **#1468 flow interfaces** — Colony derives the interface from here; do not restate.
- **#1252 scenario runner** — the proof mechanism (`scenarios/` + `swarm test`); CI runs it once it lands.
- **#1450 sealed package boundary** — makes fork/import clean; Colony is its acceptance test.
- **#1447 batch agent** / logic-node work — source of the first `batch-classify` pattern and the rebuilt flagship.

## 8. Sequencing & status

- **Now:** scaffold in place (this repo), empty of entries (no drifted content). Local-only — no GitHub remote yet.
- **Next:** file the manifest issue (§6); as #1447 lands, add `patterns/batch-classify`; rebuild `twitter-prospecting` on #1447 + logic-node and add it as the flagship template (verified against fresh master).
- **Deepen** as #1450/#1468/#1252 stabilize — don't over-invest in entries before the composition primitives are solid, or they'll be rewritten.

## Open questions
1. `colony:` block in `package.yaml` vs separate `colony.yaml` (purity-for-forking vs one-file)?
2. Separate GitHub repo `division-sh/colony` (mirrors npm/registry separation; needs cross-repo CI token) vs a directory in the main repo?
3. Is `platform_version` enforced today, or decorative? (Determines urgency of §6.)
4. `requires`: explicit-and-verified (proposed) vs pure-derived — confirm the audit-surface rationale holds.
