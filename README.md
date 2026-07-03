# Colony

A **curated, CI-verified library of Swarm flows** — reusable, working starting points you can fork, and composable building blocks you can wire into your own flows.

Colony is the trusted, first-party precursor to the Swarm flow marketplace. Everything here is hand-vetted, passes `swarm verify` against current `platform-spec`, and is proven by a scenario test. If it's in Colony, it works.

## Two tiers

| Tier | Directory | What it is | Bar |
|---|---|---|---|
| **Templates** | `templates/` | Complete, end-to-end flows — *fork-and-go* starting points | verified + scenario-tested + documented |
| **Patterns** | `patterns/` | Small, single-concept building blocks — *composable* | verified + scenario-tested + documented + **declared interface** |

A **template** is something you copy as a starting point (e.g. a full lead-gen pipeline). A **pattern** is something you compose into your own flow (e.g. a batch-classify step, a cost-router, a human-approval gate) — so patterns declare their pins/interface for swappability.

## Using an entry

Today: browse the directory, read the entry's `README.md`, copy the package, and wire it in. (A future `swarm add <name>` will pull directly from the generated `index.yaml` catalog.)

## Contributing

Every entry must clear the curation bar — see [CONTRIBUTING.md](./CONTRIBUTING.md) and [docs/authoring-guide.md](./docs/authoring-guide.md). The bar is enforced in CI: `swarm verify` (and, once available, the scenario runner) runs against **every** entry on every change and on a schedule, so a platform dialect change that breaks an entry fails CI. That's the difference between a library and a graveyard of broken examples.

## Status

Bootstrapping. First planned entries:

- **`templates/twitter-prospecting`** — flagship: X/Twitter account-prospecting pipeline (discovery → dedup → enrich → classify → bucket). Added once rebuilt on the batch-agent (#1447) and logic-node work.
- **`patterns/batch-classify`** — first building block, demonstrating the batch-agent step (#1447).
- **`patterns/cost-router`** — the deterministic N-way survivor router.
