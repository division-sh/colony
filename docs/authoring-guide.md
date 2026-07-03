# Authoring guide

How to write a Colony entry that clears the bar. See [CONTRIBUTING.md](../CONTRIBUTING.md) for the acceptance criteria.

## Entry layout

```
templates/<name>/            (or patterns/<name>/)
├── package.yaml             # the flow package root
├── schema.yaml, nodes.yaml, events.yaml, entities.yaml, agents.yaml, tools.yaml, …
├── flows/                   # child flows, if any
├── template.yaml            # metadata (schema below) — pattern.yaml for patterns
├── README.md                # what it does, I/O, requirements, how to run/compose
└── scenarios/               # deterministic scenario tests that prove behavior
    └── happy_path.yaml
```

## `template.yaml` schema

```yaml
name: twitter-prospecting            # unique within Colony
version: 0.1.0                       # semver of THIS entry
summary: One-line description of what it does
category: lead-gen                   # for catalog grouping
tags: [prospecting, social, scoring]
maturity: flagship                   # flagship | reference | pattern

platform_spec: ">=X.Y"               # platform-spec version this entry targets (the anti-rot anchor)

requires:                            # what a consumer must provide
  backend: [anthropic]               # llm backend(s) it assumes
  tools: [getxapi]                   # external tools/integrations
  credentials: [getxapi_api_key]     # credential keys it reads

interface:                           # its pins — REQUIRED for patterns, recommended for templates
  inputs:  [scan.requested]
  outputs: [account.bucketed]

proven_by: scenarios/happy_path.yaml # the scenario test CI runs
```

`pattern.yaml` is identical but `maturity: pattern` and `interface` is mandatory (patterns are composed, so their pins are the contract).

## Principles

- **Push logic into deterministic nodes; keep the LLM at the boundary.** Routing, gating, scoring math, and bucketing belong in `system_node` handlers (CEL) or (when available) `tool_call` / `logic_node`; agents are for irreducible judgment only.
- **Prove it deterministically.** Scenario tests inject canned agent outputs and assert on the resulting events/state — no live LLM spend. An entry without a scenario isn't proven.
- **Declare, don't assume.** Backends, tools, and credentials a consumer must supply go in `requires:`. Pins go in `interface:`. If a consumer can't tell what to wire in from the metadata + README, it isn't ready.
- **Pin the platform version.** Set `platform_spec` to what you verified against; CI holds you to it.
