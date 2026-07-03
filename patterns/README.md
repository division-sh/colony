# Patterns

Small, single-concept building blocks you **compose into your own flows**. Each declares its pins (`interface:` in `pattern.yaml`) so it's swappable.

Unlike templates (which you fork whole), a pattern is meant to be wired in as a component — so its interface *is* its contract.

See [../docs/authoring-guide.md](../docs/authoring-guide.md) for the layout and metadata schema, and [../CONTRIBUTING.md](../CONTRIBUTING.md) for the bar.

### Planned

- **`batch-classify`** — first building block: batch-invoke one agent turn over N accumulated entities and scatter typed results back (demonstrates the batch-agent step, #1447).
- **`cost-router`** — deterministic N-way survivor router: cheaply gate which entities pay for expensive downstream turns.
- **`human-approval-gate`** — a mailbox-backed pause for a human decision before proceeding.
