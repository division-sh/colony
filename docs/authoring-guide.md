# Authoring guide

How to write a Colony entry that clears the bar. See [CONTRIBUTING.md](../CONTRIBUTING.md) for the acceptance criteria.

## Principle: one manifest, derive the rest

An entry has **no Colony-specific metadata file**. The flow's own `package.yaml` is the manifest; Colony's catalog is *derived* from it (plus the contracts). The only Colony-owned data is two curation fields, namespaced under an optional `colony:` block in `package.yaml` that the runtime ignores.

We do **not** restate anything the contracts already declare: the interface/pins live in `schema.yaml` (see #1468), and required tools/credentials are declared once (see `requires` below and #-manifest issue).

## Entry layout

```
templates/<name>/            (or patterns/<name>/)
├── package.yaml             # the manifest (below)
├── schema.yaml, nodes.yaml, events.yaml, entities.yaml, agents.yaml, tools.yaml, …
├── flows/                   # child flows, if any
├── README.md                # what it does, I/O, requirements, how to run/compose
└── scenarios/               # deterministic scenario tests that prove behavior
    └── happy_path.yaml
```

## `package.yaml`

The manifest is the flow's own package file. Fields marked **(today)** exist now; **(proposed)** are tracked in the platform manifest issue (see [design.md](./design.md)).

```yaml
name: twitter-prospecting            # (today)
version: 1.0.0                       # (today) semver of the package
author: youmew                       # (today)
description: >                       # (today) what it does
  X/Twitter account-prospecting pipeline…
platform_version: ">=1.6.0"          # (today, must be ENFORCED) platform-spec compat — the anti-rot anchor
flows:                               # (today)
  - { id: discovery, flow: discovery, mode: static }
  - { id: account,   flow: account,   mode: static }

# ── manifest additions (proposed; platform-owned, serve marketplace too) ──
keywords: [prospecting, social, scoring]     # (proposed) discovery/search
license: MIT                                 # (proposed) redistribution
repository: https://github.com/division-sh/colony   # (proposed) provenance
requires:                                    # (proposed) EXPLICIT + verified against actual usage
  backend: [anthropic]                       #   the import-time audit surface — declared, not just derived
  tools: [getxapi]
  credentials: [getxapi_api_key]

# ── Colony curation only (runtime ignores this block) ──
colony:
  maturity: flagship                 # flagship | reference | pattern
  category: lead-gen                 # catalog shelf
```

`interface` / pins are **not** here — they come from `schema.yaml` (#1468). `requires` is the one deliberate exception to "derive, don't restate": it's the capability/dependency **audit surface**, so it's declared explicitly *and* cross-checked by `verify` against actual usage (declared-but-unused or used-but-undeclared → error).

## Principles

- **Push logic into deterministic nodes; keep the LLM at the boundary.** Routing, gating, scoring math, and bucketing belong in `system_node` handlers (CEL) or (when available) `tool_call` / `logic_node`; agents are for irreducible judgment only.
- **Prove it deterministically.** Scenario tests inject canned agent outputs and assert on the resulting events/state — no live LLM spend. An entry without a scenario isn't proven.
- **Pin the platform version.** Set `platform_version` to what you verified against; CI holds you to it.
