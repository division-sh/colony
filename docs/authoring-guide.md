# Authoring guide

How to write a Colony entry that clears the bar. See [CONTRIBUTING.md](../CONTRIBUTING.md) for the acceptance criteria.

## Principle: one manifest, derive the rest

An entry has **no Colony-specific metadata**. The flow's own `package.yaml` is the manifest — everything an entry declares is a *fact the package states about itself*. Colony's catalog (`index.yaml`) is **generated** from it, adding only **verification status** (which a package can't self-assert — it's computed at verify time).

We do **not** restate anything the contracts already declare: the interface/pins live in `schema.yaml` (see #1468), and the import contract lives in `requires` (already enforced at boot). Which tools/backends a flow reaches is **derived** from `tools.yaml`/agents and surfaced in the catalog — not re-authored.

## Entry layout

```
templates/<name>/            (or patterns/<name>/)
├── package.yaml             # the manifest (below)
├── schema.yaml, nodes.yaml, events.yaml, entities.yaml, agents.yaml, tools.yaml, …
├── flows/                   # child flows, if any
├── README.md                # what it does, I/O, requirements, how to run/compose
└── scenarios/               # smoke scenarios that prove the example still works
    └── happy_path.yaml
```

`templates/` vs `patterns/` (complete flow vs composable building block) is carried by **directory**, not a field.

## `package.yaml`

Fields marked **(today)** exist now; **(proposed)** are tracked in the platform manifest issues — and must be *strictly modeled* by the loader, since unknown top-level fields are silently ignored today (a marketplace footgun).

```yaml
name: twitter-prospecting            # (today)
version: 1.0.0                       # (today) semver of the package
author: youmew                       # (today)
description: > …                     # (today) what it does
platform_version: ">=1.6.0"          # (today, must be ENFORCED) platform-spec compat — the anti-rot anchor
flows: [ … ]                         # (today)

keywords: [prospecting, social, scoring]   # (proposed) discovery/search
license: MIT                               # (proposed) redistribution
repository: https://github.com/division-sh/colony   # (proposed) provenance
category: lead-gen                         # (proposed) catalog shelf (descriptive self-fact)
```

- `interface`/pins → `schema.yaml` (#1468), never here.
- `requires` (import contract: inputs/outputs/policy/credentials the parent binds) → already exists and is enforced; declare it there, don't invent a parallel shape.
- Capability surface (tools/backends reached) → **derived** and surfaced in `index.yaml`, not authored.

## Principles

- **Push logic into deterministic nodes; keep the LLM at the boundary.** Routing, gating, scoring math, and bucketing belong in `system_node` handlers (CEL) or (when available) `tool_call` / `logic_node`; agents are for irreducible judgment only.
- **Prove it deterministically.** Scenario tests inject canned agent outputs and assert on the resulting events/state — no live LLM spend. These are *smoke tests for the example*, not the platform's semantic conformance (that stays in `swarm`).
- **Pin the platform version.** Set `platform_version` to what you verified against; CI holds you to it.
